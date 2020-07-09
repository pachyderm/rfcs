# Goal

Spouts are a heavily used feature but they have stability issues. These
stability issues have proven hard to fix and the biggest reason for this is
that named pipes have a lot of weird (and arguably downright buggy) behaviors,
especially when they're opened and closed multiple times. This is a
proposal for an implementation that doesn't use named pipes with the goal of
making it more reliable.

# History

The challenge with implementing this feature has always been that we need to
give users a way to output a filesystem, but we can't use the same technique we
use for user code because we don't want the process to have to exit. And if the
process doesn't exit then we have no way of knowing when it's safe to read a
commit from the filesystem. This was what led us to the tar format, which is
designed for representing a filesystem as a stream of bytes, and the named pipe
interface which seemed like the obvious way for the user code to push it to the
worker. There are a number of other filesystem tricks we considered, things
like ioctl and unmounting volumes. There may well be some good ideas there, but
for this proposal we're going to focus on doing something much simpler since
the root issue comes from filesystem tricks.

# Proposal

The proposal here is to reuse the existing `put file` api this would mean you
could write your spout in Python or Go using our officially supported drivers,
or using `pachctl`. This is kind of a non-feature since this is already the
normal way that people ingress files into Pachyderm, and you could achieve
something similar by deploying your own service in k8s that talks to the
Pachyderm API. However, there are some important ergonomic differences:

- Deploying your own service is a non-negligible amount of work. Especially if
you're familiar with Pachyderm pipelines but aren't familiar with directly
using the Kubernetes API. Saving users this work has always been the most
important aspect of spouts.
- It allows us to track provenance between the spout and the spec commit which
contains the code running the spout.
- It might allow us to simplify the API a bit, for example it might not be
necessary to specify the repo you want to put the files in since it's
determined by the name of the spout. I'm not totally sure if this is worth
doing or if it makes it more confusing.

Here's an example of what this would look like in practice:

```json
{
  "pipeline": {
    "name": "spout"
  },
  "spout": {},
  "transform": {
    "cmd": [ "bash" ],
    "stdin" [
        "# download files to /tmp/files",
        "pachctl put file -r -f /tmp/files",
        "rm -rf /tmp/files"
    ]
  }
}
```

Notice that in this example `put file` doesn't specify a repo or a branch, it
just specifies where to read the files from. We can intuit the repo and branch
from the name and output branch (in this case defaulting to master) of the
spout. I'm not 100% sure it's necessary for us to intuit this and save the user
writing out the repo and branch, we're not really saving them that much typing,
and it would require modifying pachctl. However there are a few things we
should definitely do here:

- If auth is enabled we should inject credentials into the spout, probably they
should match the owner of the spout. But maybe a robot user is preferable.
- We shouldn't allow the spout to write to other repos, that could get weird.
- Whatever we do should also work with the Golang and Python APIs in much the
same way, I think that will mostly happen automatically since pachctl and the
various clients mostly read config from the same places. It's very important
that users not have to muck about with config when deploying a spout.
