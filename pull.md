# Pull

## Goal

Give users a way to pull resources from one cluster into another. Pulling
should take advantage of preexisting state in puller so that data doesn't need
to be transferred twice. Pulling should be robust in the face of network issues
sustained network failures can prevent a pull from succeeding, but network
hiccups shouldn't. Pulling will often require transferring very large amounts
of data, so a network hiccup shouldn't be able to break the entire process, it
should be smart about retrying.

## Implementation

The command we'd like to support for this would be `pachctl pull` and the
semantics look something like this:

``` sh
# pull everything from <remote>
$ pachctl pull <remote>
# pull <pipeline> from <remote>
$ pachctl pull <remote> -p <pipeline>
# pull <repo> from <remote>
$ pachctl pull <remote> -r <repo>
```

By default, pulling a resource pulls resources it depends on. Dependency is
defined as follows:

- Pipelines depend on their output repos
- Repos depend on their commits and branches
- Branches depend on the repos that contain them, and on all commits that are reachable from the branch (ie the results of `list commit repo@branch`)
- Commits depend on the repos that contain them, and on their parents and provenance commits.

Note that some resources can't be pulled individually, such as files you need
to pull in the full commit that contains the files.

The API for this should look something like this:


```proto
message Resource {
  pfs.Repo repo = 1;
  pfs.Branch branch = 2;
  pfs.Commit commit = 3;
  pfs.Object object = 4;
  pfs.Tag tag = 5;
  pfs.Block block = 6;
  pps.Pipeline pipeline = 7;
  pps.Job job = 8;
}

message PullRequest {
  // Where you want to pull from, this is set by the client when initiating a
  // pull by sending a request to the puller. The puller then strips out this
  // information and sends the request to the remote who responds to the
  // request with real resources.
  string remote = 1;
  // Resources that you want to pull from the server. Empty means pull all
  // resources.
  repeated Resource resources = 2;
  // Base is resources that you already have and thus shouldn't be included in
  // the response from the server. Passing a commit will filter that commit and
  // all of its children.
  repeated Resource base = 3;
}

message PullResponse {
  // Ops can be applied to a cluster to create the requested resources.
  repeated Op ops = 1;
}

service API {
  rpc Pull(PullRequest) returns (stream PullResponse) {}
}

```

## Remotes

The semantics of `pachctl pull` allow you to specify a remote, which is an
address to pull the resources from. However, I think it makes sense to have hub
built in as the default remote, similar to what Docker does with Dockerhub.
This would mean that you can do:

```sh
pachctl pull -p <public-pipeline>
```

And that will pull in a publicly available pipeline from a central Pachyderm
server that we run in hub.

## Auth

To perform a pull the puller must have authorization to read the resources from
the remote and authorization to write to the puller. This gets a little tricky
because right now auth is confined to a single cluster and users will have
different accounts for different clusters. I'm not quite sure what the right
architecture for this is but it should be taken into account in designing auth
2.0. In the existing architecture this would probably be a robot user that the
puller uses to auth to the remote.
