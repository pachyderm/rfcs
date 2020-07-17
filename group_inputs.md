# Group Inputs

## Goal

Pachyderm is often used to store timeseries data. One of the most common things
people want to do with timeseries data is sort it into groups based on
timestamp (also called time windowing) we don't currently have a great way to
do this other than a shuffle pipeline or committing the data presplit into
groups. Both of which can be cumbersome. They also don't give you a great way
to select only the most recent groups.

## Proposal

To support this workload I propose adding a new type of input called a Group
Input. Group inputs are similar to join inputs in that they use glob capture
to map inputs to keys that are then used to create the datums. The difference
is that in a join things with the same key get crossed together, whereas in a
group input things with the same key all get grouped together into one big
datum. This also means that group inputs are useful when you only have one
input while join inputs aren't.

## Example

Suppose an input repo in which the file names are timestamps such as
"/2020-07-04T00:00:28Z" you could group these with an input like so:

```json
{
  "pipeline": {
    "name": "cron"
  },
  "transform": {
    "cmd": [
      "sh"
    ],
    "stdin": [
      "# process a day's worth of data"
    ]
  },
  "input": {
    "group": [{
      "pfs": {
        "repo": "input-repo",
        "glob": "/(*)-(*)-(*)T*",
        "group_on": "$1$2$3"
      }
    }],
  }
}
```

This would group your data into days, it does this because part of the glob
we're capturing is the days section of the timestamp. If the glob was replaced
with `"/(*)-(*)-*T*"` and the `group_on` with "$1$2" it would be months instead.

# Input slicing

This technically could have been a seperate rfc, but the motivation makes more
sense if you already group inputs and in the original proposal it was more
specific to group inputs so it's included here.

## Goal

Users want a way to limit the datums presented to their pipelines. Building on
the example above, suppose that the user wants to group by day, but only
process that last 3 days. I.e. when a new pipelines is created it will only
have to churn through 3 days of data, not however many have accumulated. And
as new data is added, data that's more than 3 days old will be dropped.

## Proposal

The support this workload I propose adding a new field the the Input protobuf
to slice the results. This is similar to slices in Python or Go. The slice
would look like this:

```proto
message Slice {
  int64 lower = 1;
  int64 upper = 2;
}
```

and have the same semantics as those in python. The only difference would be
the meaning of 0 for upper. Proto doesn't give us a good way to distinguish an
explicit 0 from an unset value, and we want an unset upper bound to mean
unbounded (like it does in Python and Go) rather than a bound of 0. An explicit
upper bound of 0 is unlikely to be useful because it'll give you 0 total
datums, but there's also other ways to do that if users want. Similar to python
(but not go) -1 numbers can be used as slice indices to count from the end
rather than the beginning.

## Example

Suppose you had an input, group or otherwise, that produces 10 datums, numbered
0-9. Slice would have the following semantics.

| Lower | Upper | Result |
| ----- | ----- | ------ |
| 0 | 0 | [0,1,2,3,4,5,6,7,8,9] |
| 1 | 1 | [] |
| 0 | 1 | [0] |
| 0 | 5 | [0, 1, 2, 3, 4] |
| 0 | -1 | [0, 1, 2, 3, 4, 5, 6, 7, 8] |
| -3 | 0 | [7, 8, 9] |

The last one is example from the Goal section above, it gives you whatever that
last 3 datums are and tosses the other ones.
