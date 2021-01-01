# Dynamic Parallelism

## Goal
Right now pipelines can only have a constant parallelism, which means that
either that many workers are up, or 0 workers are up (if the pipeline is in
standby). We'd like to give users a way to have pipelines dynamically scale
themselves based on how much outstanding data they have to process. And give
users the ability to scale pipelines on demand if they want. Up until recently
the main blocker for this has been the architecture of the worker which made it
difficult to know the relevant information, such as datums to process, before
the workers were up. At which point it was too late to decide how many workers
we wanted to create. With the worker refactor landing in 1.11 this is now
doable.

## Proposal

### Parallelism Spec
I propose adding 3 new fields to the parallelism spec:

- maximum:
    a hard cap on the number of workers that will be spun up for a pipeline.
    The default value of 0 should probably mean no maximum, although that's a
    somewhat dangerous value, so we probably want to emit a warning when
    someone sets it to 0 or leaves it unset.

- `datums_per_worker` (int64):
    Setting `datums_per_worker` to `n` means that you'll get one worker per `n`
    datums. So if a user sets datums_per_worker=10 and then commits 50 datums
    they'd get 5 workers. This number should also affect how we chunk up datums
    so we don't wind up with workers with nothing to do.

- `bytes_per_worker` (string):
    Setting `bytes_per_worker` to `n` bytes means that you'll get one worker
    per `n` bytes the pipelines has to process. It's a string so it can support
    SI style quantities, i.e. you can set it to "1G" which will give you a
    worker per gigabyte. Similar to the above this should affect how we chunk
    datums, but you can get weirder situations. In the above example if you had
    a single datum which was "5G" then we might naively spin up 5 workers, but
    that wouldn't be useful since there's only a single datum. We should never
    spin up more workers than we have datums.

Pipelines are only allowed to set one of `datums_per_worker`
`bytes_per_worker`, `constant`, and `coefficient`. Although I recommend we get
rid of `coefficient` since it's not really used and would simplify things a
little bit.

Setting `datums_per_worker` or `bytes_per_worker` implicitly sets `standby` to
true since standby occurs when you have no data to process, and setting either
of those values implies that when there's 0 datums or 0 bytes there should be 0
workers. We may want to consider adding some sort of threshold for how long we
wait to scale workers down.

### Scale pipeline command
Sometimes users realize they want more (or fewer) workers after a job has
started. For that case I propose we add a new endpoint called scale pipeline.
The pachctl syntax would look like this:

```shell
# scale pipeline foo to 10 workers
$ pachctl scale pipeline foo 10
```

this would immediately spin up the new workers without disrupting the existing
ones.

## Implementation

I don't have a full picture of how the implementation will work at the time of
writing this. So this is more of a sketch and some questions that the
implementor should answer before starting.

The first question is where this code should go. I think the answer is probably
in the `startJob` function in
"src/server/worker/pipeline/transform/registry.go". That seems to be the first
place where we compute how many datums we have to process in this job. This
code should be able to talk to kubernetes to scale the pipeline's rc. Right now
the thing responsible for scaling up and down the pipeline rcs is the pps
master. We still need to be able to do scaling there, since pipelines need to
be able to scale down to 0 workers, at which point there's nothing running the
worker code to scale it back up.

The other interesting question is how to scale down the pipeline. Scaling up is
easy, you just tell kubernetes to add more replicas and it'll figure everything
else out. Scaling down is more tricky, you can tell kubernetes to scale down
your rc, but you have no control over which pods it chooses to kill. This
presents a problem if some workers are processing datums while others are idle.
You might wind up killing the workers that are in the middle of processing
datums and losing the progress they've made.

One solution is to just take the hit and sometimes scale down workers that are
running. That can be inefficient but it is simple.

Another solution to this is to only scale down when you don't have an active job
and can scale down to 0. This would probably work pretty well, the downside is
that if you have a workload such that the pipeline is never totally idle you'll
never be able to scale down. That coupled with a big spike in datums at some
point that causes you to scale up can lead to a lot of wasted resources.

There is also a somewhat hacky way to control which pods are killed when you
scale down an rc. It relies on the fact that rcs operate on pod labels, when
you create an RC you're actually telling k8s "I want this many pods which
satisfy this label selector." You can leverage this to selectively kill pods
using the following algorithm:

Suppose you have an rc which is running with the label selector:
`gen-n=<pipeline-name>`.

- Add the pods you want to keep to the next generation by manually applying the
label `gen-n+1=<pipeline-name>`.
- Update the rc to remove the selector `gen-n=<pipeline-name>` and add the
selector `gen-n+1=<pipeline-name>`. Also update to the desired replica count.

Because we manually applied a label which satisfies the rc's selector before
the rc existed those pods will immediately be counted as replicas, which means
they won't be deleted. The pods we didn't apply the label to will be deleted
since they're now orphaned.

I have successfully gotten the above to work so I at least know it's possible.
I haven't experimented with it enough to be able to say if it's a good idea or
not. It definitely looks like it might be overly complicated and have some
unexpected behavior. It also involves making multiple calls to the k8s in a
non-atomic way which might have some downsides. Although afaik applying labels
shouldn't particularly matter if we crash midway through and leave them hanging
around. The bottom line though is that if we want to use this technique we
should do more testing on it to see how k8s behaves.
