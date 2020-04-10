# Background
This feature allows Pachyderm jobs to expose an S3 interface through which a
job's specific input and output data may be read and written, respectively.
From the user's perspective, this is an alternative to interacting with the
`/pfs/(in|out)` directory in the context of a job. This S3 interface allows
workers outside of Pachyderm user containers--Kubeflow workers, Spark workers,
Presto workers, etc--to interact with Pachyderm in the context of a job and,
because of the constraints imposed by the job-specific S3 interface, read and
write data freely without violating our data lineage invariants.

The S3 interface exposed in the context of a job ("job-scoped S3 gateway")
exposes job inputs, but not other commits in the input branch branch or any
other Pachyderm commit that isn't a job input, unlike our current S3 gateway,
which exposes all Pachyderm commits. Likewise, the job-scoped S3 gateway only
allows callers to write to the job's output commit, unlike our current S3
gateway, which allows callers to write to any open commit or create new
commits.

# Interface

The pipeline spec gets two new fields:

1. `PFSInput` gets a new field called `S3`. Setting `S3: true` exposes commits
from this input through the job-scoped S3 gateway as a read-only bucket
`s3://input-name`, where `input-name` is taken from the PFSInput's `name` field
(or `repo` if name is unset). PFS inputs with `S3: true` are **S3 inputs**

2. `CreatePipelineRequest` gets a new field called `S3Out`. Setting `S3Out:
true` exposes the job's output commit through the job-scoped S3 gateway as a
writable bucket called `s3://out`. Pipelines with `S3Out: true` are **S3Out
pipelines** and jobs in these pipelines are **S3Out jobs**.

**Note**: Bucket names in a job-scoped S3 gateway are parallel to paths in
`/pfs`: `/pfs/foo` is exposed through the bucket `s3://foo`.

User code communicate with the job-scoped S3 gateway via a job-specific
kubernetes service, called the **S3 job kubernetes service**. User code
shouldn't make any assumptions about this service's domain name, but in the
first version of this feature, it'll be `http://s3-<job id>`. This URL is
passed to user code via the `S3_ENDPOINT` environment variable.

# Implementation notes:
- Pipelines that either have `S3Out: true` or an S3 input are **S3 pipelines**.
  - Jobs in these pipelines are **S3 Jobs**.
- Pachyderm only creates a job-scoped S3 gateway and an S3 job kubernetes service for S3 jobs.
- Pachyderm only sets the `S3_ENDPOINT` environment variable for user code running in S3 jobs.

### Code architecture
- Most of this feature is implemented in two places:
  1. a re-architecting of the S3 gateway (`src/server/pfs/s3`)
  2. the implementation of `src/server/pps/server/s3g_sidecar.go`.
		- This file defines a new long-lived struct, `sidecarS3G`, (initialized in
      `pps.NewSidecarAPIServer()`) that is responsible for creating and deleting this
      feature's job-scoped resources: the job-scoped s3 gateway and s3 job kubernetes
      service
- Because the S3 job kubernetes service should only be created once, but the
  job-scoped s3 gateway should run in each sidecar container (requests may be
  sent to any sidecar), sidecarS3G establishes two watches: one that creates the
  former and runs in every sidecarS3G and one that creates the latter and only
  runs in an elected master.

### The Job-Scoped S3 Gateway
- The job-scoped S3 gateway is a running instance of a modified version of our
  existing S3 gateway.
- The modifications to our S3 gateway include two new features, both of which
  are initialization options: bucket aliases and bucket permissions
	- Bucket aliases identify a caller-specified bucket name with a particular
    commit. For example, `s3://foo` could be an alias for the commit
    `foo@<commit-id>`. Reads from or writes to `s3://foo` are applied to that
    commit.
  - Bucket permissions allow buckets to be designated read-only or write-only.
- When a new job is created, `s3g_sidecar.go` creates a new instance of S3 gateway where,
	- for each of the job's s3 inputs, the bucket `s3://<input.name>` is aliased
    to `input.repo@input.commit` and made read-only.
  - If the job is an S3Out job, then the bucket `s3://out` is aliased to the
    job's output commit and made write-only.
  - Any commits not given aliases won't be exposed in this mode

### The S3 Job Kubernetes Service
- S3 job kubernetes services are headless (they set `ClusterIP: None`, and the
  service's domain name resolves to the IPs of the worker containers). This is
  necessary due to internal details of kubernetes routing:
  - Cluster-internal IPs (service IPs in particular) are routed by a kubernetes
    daemon called kubeproxy.
  - As far as I can tell (based on some experimentation) kubeproxy daemons
    can't recieve new routes. Therefore if a pod is created first and a service
    is created second, the pod's kubeproxy won't know how to route messages
    sent the service's IP.
  - Because pipeline workers are scoped to a pipeline version rather than a job, the order in which the relevant resources are created is:

      1. pipeline
      2. workers
      3. output commit
      4. k8s service routing to job-scoped s3 gateway
      5. job
 
    If the s3 job kubernetes service were a ClusterIP service, then when workers tried to send requests to that service, they'd fail to connect.
- Currently, the worker APIServer confirms that the S3 job kubernetes service is available immediately before running user code. If it's not available, then the APIServer fails the job.

### Datums in the context of S3 Jobs
##### S3Out
- S3Out jobs are unusual, and get special handling in the worker. They're much more like input commits than output commits.
- Every datum is reprocessed by every S3Out job
  - This is necessary because when workers write output data to the job-scoped
    s3 gateway, pachyderm has no way to tie that write to a datum (Pachyderm
    has no idea where that write came from or what that worker is doing right
    now, particularly in the case of external workers, like with Kubeflow).
  - Therefore Pachyderm has no way to skip datums.
- S3Out workers write to the output commit using the input commit codepath:
  writes to `s3://out` become `PutFile` requests that create PutFileRecords in
  etcd. S3Out jobs use the input commit's version of `FinishCommit`.
- The only difference between S3Out jobs and input commits is that S3Out jobs
  don't merge the PutFileRecords with the output commit's parent.
  - Input commit semantics are analogous to Git: you check out a commit, make
    changes, and then commit your changes
  - In the case of S3Out jobs, every output commit should contain the product
    of processing the job's datums. The output commit's parent has the result
    of processing the job's parent's datums, which is irrelevant
  - To enforce these semantics, the worker master calls `DeleteFile(/)` on
    S3Out jobs' output commit before creating the EtcdJobInfo

##### S3 Inputs
- non-S3Out jobs that have S3 inputs are more similar to non-S3 jobs, but they
  do get extra validation:
  - S3 Inputs are exposed by the job-scoped S3 gateway from the start of a job
    to the end. This means that
    - Every worker has access to the S3 input
    - Every worker has access to the whole commit
  - To reflect this reality in jobs' data lineage, we impose two constraints on
    S3 inputs:
    - They can only be a top-level input or included in `cross` inputs (i.e.
      `union` and `join` inputs can't contain s3 inputs)
    - Their glob pattern must be `/`
- These constraints impose the expected semantics on S3 inputs: if any path in
  an S3 input changes, every datum must be reprocessed. If any path in a non-s3
  input changes, and that non-s3 input is siblings with an s3 input in a
  `cross`, then only the changed datums are processed

### Combining S3 Jobs and Parallel Jobs
- The main question raised by parallel jobs is: how should the job-scoped S3
  gateway be hosted and accessed by parallel running jobs?
- We've discussed two possible solutions:

  1. Serve each job-scoped S3 gateway on its own port
    - This solution raises the question of how workers will know the port to which they should connect.
    - In the course of researching this, I learned that kubernetes services can
      be created with no selector and that specific endpoints can be added to
      the service directly.
    - This suggests a solution where, when a new job is started, the sidecar
      hosts a new job-scoped S3 gateway on a random, unused port and then adds
      its own IP and that port as an endpoint of the S3 job kubernetes service.
      - Requests sent to the S3 job kubernetes service arrive at the correct
        job-scoped S3 gateway automatically.
    - However, I think that service endpoints are only a feature of ClusterIP
      services, and that because we're required to use headless services, this
      solution may not work.
      - Workers will resolve the S3 job kubernetes service's hostname to a
        worker IP, and they'll get no information about relevant ports

  2. The sidecars can host a single "super-S3-gateway" that virtually routes
     requests to the right job-scoped S3 gateway instance.
    - All S3 job kubernetes services have the same endpoint (the pod IPs), and
      the super-S3-gateway handles S3 requests from all jobs' workers
    - The super-S3-gateway contains, internally, a job-scoped S3 gateway
      instance for each job.
    - The super-S3-gateway inspects the destination of each request (i.e. the
      domain name to which it was sent, even though all s3 job kubernetes
      services' domain names resolve to the same place), and, based on the
      destination's domain name, the super-S3-gateway passes the S3 gateway
      request object to the right job-scoped S3 gateway instance.

