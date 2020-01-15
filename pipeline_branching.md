# Pipeline Branching

## Goal

Allow users to have multiple, named versions of a pipeline (branches)
running concurrently.

## Use Case:

The primary use case for this is to give users a good way to have `dev`,
`staging` and `prod` versions of pipelines. There are a ton of use cases for
this feature, but so far I haven't encountered one that presented challenges
that the `dev`, `staging` `prod` use case doesn't, so for this document we'll
just stick to that for inspiration.

## PPS API changes:
The pipeline message will be extended to include a `Branch` field, which
is a string:

```proto
message Pipeline {
  string name = 1;
  string branch = 2;
}
```

In pachctl this is implemented with the familiar syntax of `pipeline@branch`.

An empty `branch` field is equivalent of a value of `"master"`.

Different branches of the same pipeline are effectively different pipelines. That means:
- They output to different branches, with names that match their branch names.
- You can delete, update, pause, etc. one without pausing the other.
- They can take different inputs, have different resource requests, etc.
- They run in different pods, and their success / failure is independent.

There are a few ways in which they're not quite totally distinct:
- They consolidate in list pipeline, this one's mostly cosmetic, I'm
imagining something that looks like this:
```
NAME     BRANCHES     INPUT    CREATED    STATE / LAST JOB  DESCRIPTION
pipeline dev, staging input:/* 6 days ago running / success
```
- They share datums for deduplication, if you want a dev version to
reprocess, you need to create it with `--reprocess`.
- They can't take each other as input. Implementation wise there's no
particular reason for this, but it seems very weird and I can't think of any
reason you'd want to do this that there isn't a better way of accomplishing.

### A few examples of what syntax would look like for this:
Creating a dev branch:

```
extract pipeline pipeline >pipeline.json
# modify pipeline.json, setting Pipeline.Branch to "dev"
create pipeline -f pipeline.json
```

List jobs from dev branch:

```
list job -p pipeline@dev
```

Get files from dev branch (all other file commands work similarly):

```
get file pipeline@dev:/file
```

Promote dev branch to master

```
# modify pipeline.json, setting Pipeline.Branch to ""
create pipeline -f pipeline.json
```

## Downstream Pipelines:
The above changes are fairly straightforward, both in terms of how we
would implement them and in terms of explaining them to users. Downstream
pipelines is where it starts to get complicated. Downstream pipelines are
important to consider because when you're iterating on a pipeline, you
care that the results of your new version can be consumed by the rest of
the DAG downstream. For example, suppose I have the pipelines: A, B and C
with a DAG like this:

```
A -> B -> C
```

Initially, each pipeline has only a `master` version, then we create
`A@dev` this should create branches of `B` and `C` also called, `dev`
which process `A@dev` and `B@dev` respectively. So now the picture looks
like this:

```
A -> B -> C
A@dev -> B@dev -> C@dev
```

B@dev and C@dev are identical to B and C, except that they process the
`dev` branch of their inputs, rather than `master`.

Adding a new pipeline works similarly, if someone comes and creates `D`
then we will automatically create `D@dev` which is identical to `D`,
except that it takes `C@dev` instead of `C` as input. The picture would
now look like this:

```
A -> B -> C -> D
A@dev -> B@dev -> C@dev -> D@dev
```

This means that there is a special relationship between the `master`
branch of a pipeline and its other branches, we formalize this
relationship as a parent-child relationship.

## Child Branches
Children is a new relationship that we add to branches, the proto changes
like this:

```
message BranchInfo {
  Branch branch = 4;
  Commit head = 2;
  repeated Branch provenance = 3;
  repeated Branch subvenance = 5;
  repeated Branch direct_provenance = 6;
  repeated Branch children = 7;
  Branch parent = 8;

  // Deprecated field left for backward compatibility.
  string name = 1;
}
```

`CreateBranch` request will also need a `Parent` field added to it so this
value can be set.

Child branches maintain the following invariant:
a parent of b -> (a' subvenance of a -> a' parent of b' and b' subvenance of b)

In english: if a is the parent of b, and a has a subvenant branch called
a', then a' is the parent of a branch called b'.

Note that these semantics are implemented entirely in PFS, and leveraged
by PPS to implement the semantics described above, similar to the rest of
our provenance semantics. This means that you can leverage these semantics
in PFS to have, for example a dev version of the data. That gets run
through the dev versions of your pipelines.

## Open Questions:
How this interacts with deferred downstream processing is an interesting
question. Because it breaks the chain of branch provenance the development
branch won't automatically propagate all the way through the DAG, you'll
have to manually hook it up at the repo where you're deferring. I think
this is probably ok, because the whole point of deferred downstream
processing is that you manually trigger pipelines, rather then it
happening automatically, so it makes sense that you would have to do that.
