## Motivation

Users want to programatically trigger pipelines.

The default in Pachyderm is that pipelines eagerly process data that's
committed to their inputs. This should remain the default but it breaks down in
some use cases. For those use cases we have the pattern of deferred processing,
this allows you to fully control when pipelines run by moving branches. This is
also something that should remain an option, but it also breaks down in some
use case. The motivation for triggers is to add something between these two
options that doesn't require a separate process (or a human) to move the branch
heads but also doesn't move them blindly on every commit.

## Requirements

For the purposes of this rfc we'll limit the trigger conditions to the
following metadata aspects:

- cron: trigger a pipeline if a cron spec is satisfied and new data has been committed.
- size: trigger a pipeline if an input has grown by a given amount.
- commits: trigger a pipeline if input has received a given number of commits.

The only trigger that might have made this list but didn't is number of files.
This wouldn't be terribly hard to add, the only downside is that it requires
walking the files which would make it less performant than the other triggers.

It is not possible for a trigger to set more than one of these aspects.

It should be possible for a pipeline that takes multiple inputs to define
triggers on each of these inputs and those triggers should happen atomically.
This is important so you don't have two inputs trigger at the same time and
create 2 jobs, one of which will process both new inputs, the other will only
process one.

## Proposal

Users will mostly interact with triggers through pps, while most of the
implementation will be in pfs.

### PPS


Add a trigger field to pfs inputs called "trigger" with the following spec:

```proto
message TriggerCond {
  string branch = 1;
  // Triggers if the cron spec has been satisfied since the last trigger and
  // there's been a new commit.
  string cron_spec = 2;
  // Triggers if there's been `size` new data added since the last trigger.
  string size = 3;
  // Triggers if there's been `commits` new commits added since the last trigger.
  int64 commits = 4;
}
```

For example if you wanted to run the edge detection pipeline only when there
have been 10 megabytes of new data it would look like this.

```json
{
  "pipeline": {
    "name": "edges"
  },
  "description": "A pipeline that performs image edge detection by using the OpenCV library.",
  "input": {
    "pfs": {
      "glob": "/*",
      "repo": "images",
      "trigger" {
        "size": "10M"
      }
    }
  },
  "transform": {
    "cmd": [ "python3", "/edges.py" ],
    "image": "pachyderm/opencv"
  }
}
```

### PFS

Triggers are a new primitive which we'll add to pfs. Similar to provenance,
triggers are a feature which is mostly implemented and reflected in the pfs api
but that users will mostly interact with through the pps api. Triggers link
branches to other branches, and should be though of as saying that one branch
should be updated to point to the head of another branch when certain
conditions are met. We refer to this as the first branch being "triggered by"
the second branch. The API for this looks like so:

```proto
// TriggerCond defines the conditions under which a head is moved, and to which
// branch it is moved.
message TriggerCond {
  string branch = 1;
  // Triggers if the cron spec has been satisfied since the last trigger and
  // there's been a new commit.
  string cron_spec = 2;
  // Triggers if there's been `size` new data added since the last trigger.
  string size = 3;
  // Triggers if there's been `commits` new commits added since the last trigger.
  int64 commits = 4;
}

// TriggerPair is a helper struct for TriggerInfo.
message TriggerPair {
  Branch branch = 1;
  TriggerCond cond = 2;
}

// TriggerInfo defines a trigger and is the structure stored in etcd to persist
// it. It may operate on any number of branches by having multiple specs, it
// cannot have multiple specs for the same branch though. Having multiple specs
// within the same trigger is different from having those specs spread across
// multiple triggers because the specs trigger atomically.
message TriggerInfo {
    Trigger trigger = 1;
    repeated TriggerPair spec = 2;
}
```

One thing to notice, a trigger is defined by a `TriggerPair` and it does not
allow you to have a branch trigger another branch on a different repo. Why we
need this constraint should be obvious but the fact that it's enforced by the
type system rather than validation logic is an intentional and good feature.

They also support the standard Create, Inspect, List and Delete methods.

```proto
message CreateTriggerRequest {
  repeated TriggerPair spec = 1;
}

message InspectTriggerRequest {
  Trigger trigger = 1;
}

message ListTriggerRequest {
  Repo repo = 1;
  Branch branch = 2;
}

message DeleteTriggerRequest {
  Trigger trigger = 1;
}
```


## Implementation

In order to implement this we'll need to store a link from branches to the
branches they trigger. I don't see a need to store the reverse link, but we may
want to just in case that changes. This will mean expanding `BranchInfo` like so:

```proto
message BranchInfo {
  Branch branch = 4;
  Commit head = 2;
  repeated Branch provenance = 3;
  repeated Branch subvenance = 5;
  repeated Branch direct_provenance = 6;
  repeated TriggerCond triggers = 7;

  // Deprecated field left for backward compatibility.
  string name = 1;
}
```

### Non-cron triggers

Cron is a bit more complicated implementation wise than the other ones so we'll
discuss that separately. For all of the other ones this should be implemented
in `propagateCommits` and should just require adding some code when a new
branch head is created that iterates through the branches that branch can
trigger and checks if any of the conditions have been met. Note that the
comparision for the condition is made between the heads of the two branches.
I.e. if branch `A` triggers on branch `B` when there has been a GB of new data
added that means that when the head of `A` changes we compare it to the head of
`B` and if it has a GB of new data we update the head of `B` to be the head of
`A`.

### Cron triggers
Everything above also applies to cron triggers, the condition for a cron
trigger is that if `A` cron-triggers `B` then `B`'s head will be updated to
`A`s if there's a time that matches the cron spec between the timestamp for the
current head of `B` and the present time and the head of `B` isn't already the
same as the head of `A` (in which case triggering would be a no-op). This
description is a little tough to parse but I think in practice the semantics
will be intuitive and users won't have to think of it the way I just described
it. This condition can be satisfied in two ways:
- by a new commit being added to `A`, which is why the above logic still
  applies to cron triggers.
- by enough time passing that the existing head of `B` is out of date and we
  want to trigger again. Which is the part the above logic doesn't satisfy.

For the second, time based version of the trigger, we need to setup a pfs
master process that's responsible for keeping track of time and triggering when
the cron window elapses. Normally this would be a pretty big thing to add for
part of a smallish feature like this, but we're already adding / have added
a pfs master for new storage stuff, so this will be just a small addition to
that.

## Branch Naming
Astute readers may have noticed that the `TriggerCond` message has a field to
specify the name of the branch, but it was left blank in the Edges example.
This is intended because it doesn't make sense for users to have to come up
with branch names when there's likely nothing they'll want to do with those
names. We should make the names up for them, and ideally they should be
something readable such as `<pipeline-name>-<input-name>-trigger` or something
like that. In implementing this make sure to think about ways in which you can
get name collisions. User's accessing the api directly through pfs will have to
come up with their own names.

## Pachctl
Most of the important changes are in the pipelines spec so those won't require
pachctl changes, but we will have to add trigger specific commands. The only
really interesting one is `create trigger` `inspect` `delete` and `list` should
all be fairly obvious except maybe figuring out which fields to display in
`list`, but there aren't many fields so the answer is probably just all of
them. For `create trigger` I'm imagining something like this:

```
# staging triggers master every 1G of data
pachctl create trigger repo@staging repo@master --size 1G
# staging triggers master every day
pachctl create trigger repo@staging repo@master --cron "@daily"
```

This doesn't give users a way to create a trigger with multiple conditions,
I think trying to squeeze that into the cli might be more trouble than it's
worth. Although if the implementor wishes to try they're welcome to. It should
be a very rare use case so I think it may make sense to provide a `-f` flag
where one can specify the trigger as json similar to a pipeline spec.
