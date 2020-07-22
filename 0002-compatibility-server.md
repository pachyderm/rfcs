Start Date: 2020-07-21

# Compatibility server

## Summary

Create a "compatibility server", which stores a mapping of pachyderm core
versions to the versions of associated products (dash and IDE.)

## Motivation

We currently store a mapping of pachyderm core version to dash version(s) in
`etc/compatibility` of the pachyderm repo. When a user deploys dash, `pachctl`
makes an HTTP call to the appropriate file on github's static content server
to get the compatible version(s) of dash. e.g. to get the dash version for
1.11.0, we make a call to:

    https://raw.githubusercontent.com/pachyderm/pachyderm/1.11.x/etc/compatibility/1.11.0


This works, but is slightly brittle: we'll need maintain this directory in its
current path and structure for the foreseeable future.

A similar compatibility mapping is needed for Pachyderm's IDE. Rather than
replicate this setup, it may make sense to create a "compatibility server"
that handles IDE version mapping, as well as the mapping for future dash
versions so that we can eventually remove this directory.

## Detailed design

### Server

For the foreseeable future:

1) Add a subdomain `compatibility.pachyderm.com` that points to an object
store bucket.
2) The bucket contains a `dash` directory with the current contents of
`etc/compatibility`.
3) The bucket contains a separate `ide` directory which contains a similar
mapping for pachyderm core -> IDE versions.

When a user runs `pachctl deploy dash` or `pachctl deploy ide`, we'd find
the appropriate file in `https://compatibility.pachyderm.com/dash` and
`https://compatibility.pachyderm.com/ide` respectively, rather than via
`raw.githubusercontent.com` as per usual.

This design should be more easily extensible along two fronts:

1) Should our compatibility needs get more complex, we have the flexibility of
swapping out statically hosted content with a custom server.
2) As the need for future compatibility matrices must be maintained, we won't
pollute the core repo with more directories that cannot be moved or changed.

### Releasing

Currently, we determine compatible versions of dash at release time: a script
is called that stores the result in `etc/compatibility/<pach core version>`.
That same process would be maintained, but instead of storing it in the
aforementioned path, all releases subsequent to this RFC's implementation
would instead upload that same file to the compatibility bucket. Likewise
with the IDE.

## How We Teach This

This would be an internal implementation detail, so wouldn't need to be taught
to users.

## Drawbacks

This adds a bit of infrastructural complexity, because we'd need to maintain
the subdomain, object store bucket, and scripts to push compatibility files
to the object store on deployment.

Additionally, we'd need devs who can release to have write access to the
appropriate bucket. So long as IAM permissions are conservative, I don't
foresee this being a problem.
