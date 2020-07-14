# Group Inputs

## Goal

Pachyderm is often used to store timeseries data. One of the most common things
people want to do with timeseries data is sort it into groups based on
timestamp (also called time windowing) we don't currently have a great way to
do this other than a shuffle pipeline or committing the data presplit into
groups. Both of which can be cumbersome. They also don't give you a great way
to select only the most recent groups.

## Proposal

To support this workload I propose adding a new type of input called a Bucket
Input. Bucket inputs are similar to join inputs in that they use glob capture
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
    "group": {
        "inputs": [
            {
                "pfs": {
                  "repo": "input-repo",
                  "glob": "/*-*-(*)T*",
                  "group_on": "$1"
                }
            }
        ],
        "count": 10,
    }
  }
}
```

This would group your data into days, it does this because part of the glob
we're capturing is the days section of the timestamp. If the glob was replaced
with `"/*-(*)-*T*"` it would be months instead. Also because we've passed `10`
as the count for the group input we will only process the 10 most recent days.
