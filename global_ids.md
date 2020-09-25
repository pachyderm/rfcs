# Global Identifiers

## Goal

Simplify Pachyderm's data model, both from a user and implementation
perspective by having global identifiers for jobs and commits, rather than
local identifiers that are linked together.

## Use Case:

Give users intuitive answers to the question how do I get from `X` to `Y`,
     where `X` and `Y` can be commits or jobs at any point in the DAG.

## Proposal

When you commit data to Pachyderm your new commit has an ID associated with it.
This ID is the closest thing to a global identifier for what users think of as
the DAG / job associated with that data. Using this identifier you can do a
bunch of things:

- check the progress of the data through the DAG with `inspect commit`
- wait for downstream commits to finish with `flush commit`
- wait for downstream jobs to finish with `flush job`
- cancel the job with `delete commit`

I propose that we formalize this as a global ID, and rather than having
downstream jobs and commits with different IDs that can be linked to this ID
with commands such as the ones above, we simply have them all share an ID.
Globalizing IDs has some subtle but very powerful effects, both on how people
use the system and how we implement it.

## Usage

At a high level, the major benefit of global IDs is that it completely removes
any questions about how you get from one point in the DAG to another point,
Because all the related steps in the DAG have the same ID, you can get the ID
from a commit or job at the top, middle, or bottom of the DAG it's all the same
ID, and you can then use the ID to request whatever you want. For example:


You can get the job associated with an ID by doing:
```
inspect job <id>
```
note that because all jobs in a DAG now have the same ID, this returns
information about the entire DAG, not just a single step as it currently does.
If you want just one step you can do:
```
inspect job <pipeline>@<id>
```
to get information about just the step in this job for `<pipeline>`. This is
nice because it doesn't require you to find any extra information, you have an
ID, if you want to see what happened in the job associated with a specific
pipeline, all you need is the human readable pipeline name, which you already
have, no multiple hops.

This will also greatly declutter `list job` since it will now show all of the
steps on a single line, rather than one per step. Similar to above, you can
still do `list job -p` to filter to a single pipeline.

Traversing Provenance and Subvenance now becomes trivial. If you have an ID of
a commit and you want to see its provenance or subvenance in a different repo,
all you need to do is swap out the repo name. `input@<id>` is the provenance of
`output@<id>` and vice versa.

Deletion of commits and jobs also becomes a little cleaner. When you delete
a commit it deletes the commits subvenance, that's nothing new, it already
works that way, but now all those commits have the same ID, so it's a lot
clearer what gets deleted by the operation. We may want to change the syntax of
delete commit to be `delete commit <id>` rather than `delete commit repo@<id>`
since it makes it even more clear what will be deleted, and removed the ability to delete from the middle of a DAG, which is an error anyways.

Logs becomes a little more ergonomic in this setting, since you can now do
`logs -j <id>` to get logs for the entire DAG in one command. This should make
it a bit quicker to diagnose what went wrong. You can still use `logs -j <id>
-p <pipeline>` to get logs from a specific pipelines step of a job.

## Implementation

The implementation of this is actually pretty pleasant, and should have
a number of implementation side benefits in addition to the user benefits
above. Although it does introduce a few complications as well, I think it
should overall be a simplification.

The first thing to notice is that this removes a lot of our need to store
provenance and subvenance. Since those commits will all have the same ID we
don't need to store that ID, just a record of which repos have the commit.
I think it'll probably make the most sense to consolidate commits into single
documents in etcd. This has a number of benefits:

- it should greatly simplify the code that keeps track of provenance and make
  inconsistent states a lot harder to get into because consistency is defined
  within a single document, rather than a chain of them.

- It greatly decreases our etcd usage. In the current system a commit to a DAG
  with 10 pipelines creates 11 commits in total. These are all committed
  transactionally, and when deletions happen, deleted transactionally. Which
  can get us over the op limit etcd pretty quickly. In this new system creating
  and deleting a commit could only require a single op. It'll likely require
  a bit more to adjust branches and commit parentage. But it'll be well
  bounded, and not proportional to the size of the DAG.

We can do a similar thing with jobs and move them into a single document for
the whole DAG, and unlike with commits we kind of have to because jobs aren't
namespaced the way that commits are. We also want to do this because it has
similar benefits of decreasing the total number of documents we store in etcd.
It should also simplify code a little, although the code for manipulating jobs
is a lot simpler, so less to be gained there.

## Complications
While I think this overall simplifies our implementation it does introduce
a few complications.

### Stat Commits
Stat commits are a little complicated because they occur in the same repo as
the commits they're provenant on, which would cause a name collision. I think
this is fairly easily patch by either namespacing commits by repo and branch,
or by moving them into a separate repo.

### Multiple inputs
Pipelines with multiple inputs have create an interesting complication.
Consider a DAG in which repos `A` and `B` are inputs to a pipeline `C`. When
you commit to repo `A` you get a downstream commit in `C` with the same ID,
however `B` doesn't have a copy of that commit now. So we're back to a world of
needing to link commits with different IDs together to get reasonable
provenance and subvenance. The solution is simply to have committing to `A`
also commit to `B`, because they share subvenance. This might seem like it
makes a lot of unnecessary commits, and in some sense it does, but in terms of
what's stored in etcd it doesn't, since it just adds a few extra bytes to the
commit document we're making anyway, which is less than the subvenance update
we had to do in the old system. It does add an extra commit to the history of
`B`, but I think that's overall a good thing, since it means you can use this
ID to see which data in `B` was used to create the downstream commits. It's
helpful to think of the new commit on `B` as not really a commit, as much as an
alias for whatever's currently the head commit of master. That's basically how
it will be implemented.

### Branch Movements

Note: this section is mostly superseded by the section after it, but is left
here for posterity.

Moving branches around is the other case where this can start to break down.
For the most common use case for branch movements, deferred processing, this
should work fine, because generally the commit you're moving to hasn't been
processed yet, and thus doesn't have any downstream commits of the same name,
so moving the branch creates those commits.

Things get a little more complicated if you move the branch back to an older
commit that's already been processed because we already have downstream commits
with that ID. That use case is already a little bit weird in Pachyderm as it
exists today. If the commit was successfully processed before then moving the
head to it will create a new commit, in which all of the datums will be skipped
and thus will have the same output as the previous commit, which is to say it's
not really useful for anything. If the commit failed then you can force the
pipeline to reprocess it, which is actually useful, although generally not
necessary because generally those datums will have been processed in later
commits. Sometimes it's nice to see the results merged for a specific commit
though.

I think a good solution to this in the new system would be as follows: the
branch to a commit that's already been processed moves the subvenant branches
to the commits with the same ID. This is mostly equivalent to what happens now
in that you get the same result for the heads of the downstream branches. It's
different in that it doesn't create a new commit, and doesn't redo the merge,
both of which are not really benefits. If you would like to reprocess you can
pass a `--overwrite` flag to `create branch` which overwrites the contents of
the downstream commit with a reprocessed version.

This case is the part of this proposal I'm least sure about, so I think we
should leave this a little open once we actually dive into the code to see what
makes sense. I think overall though this particular piece isn't that important
of a use case, so we should be able to find something good enough here.

### Generalized Solution

This supersedes most of what's in the Branch Movements section above, should be
easier to reason about and apply generally to whatever crazy things people do
without us having to think about each specific case one by one.

Every Pachyderm operation occurs within a transaction which has an ID
associated with it. Whenever a transaction runs it moves around the heads of
branches, and creates downstream jobs which process those heads. For each
transaction we simply use the transaction ID as the ID for all new commits
created, the head of all branches are also set to this commit ID, which is then
aliased to the right commit for the branch if a new one wasn't created, and
finally create a job for this transaction, also using the ID.
