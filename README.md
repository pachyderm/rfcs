# Pachyderm RFCs

tl;dr

- High-level feature ideas get discussed as issues on this repo
- Specific feature proposals get written up as RFCs (see process below)
- RFCs land before their feature
- Keep issues on pachyderm/pachyderm for things that are implementation-ready
 (bugs, changes for agreed-upon features)

More detailed problems this process tries to solve are here:
https://github.com/pachyderm/rfcs/issues/2

---

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put
through a bit of a design process and produce a consensus among the Pachyderm
core developers.

The "RFC" (request for comments) process is intended to provide a
consistent, repeatable path for new features to enter Pachyderm.

[Active RFC List](https://github.com/pachyderm/rfcs/pulls)

## When you need to follow this process

You need to follow this process if you intend to make "substantial"
changes to Pachyderm, or its documentation, or any other
[official projects of the Pachyderm organization](https://https://github.com/pachyderm).
What constitutes a "substantial" change is evolving,
but may include the following:

   - A new feature that creates new API surface area.
   - The removal of features that already shipped as part of an official release.
   - The introduction of new idiomatic usage or conventions, even if they
     do not include code changes to Pachyderm itself.

Some changes do not require an RFC:

   - Rephrasing, reorganizing or refactoring
   - Addition or removal of warnings
   - Additions that strictly improve objective, numerical quality
criteria (speedup, scalibility, etc.)
   - Additions only likely to be _noticed by_ other implementors-of-Pachyderm,
invisible to users-of-Pachyderm.

If you submit a pull request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.

## Gathering feedback before submitting

It's often helpful to get feedback on your concept before diving into the
level of API design detail required for an RFC. **You may open an
issue on this repo to start a high-level discussion**, with the goal of
eventually formulating an RFC pull request with the specific implementation
 design. We also highly recommend sharing drafts of RFCs in `#rfcs` on 
the [Pachyderm Slack](http://slack.pachyderm.io) for early feedback.

## The process

In short, to get a major feature added to Pachyderm, one must first get the
RFC merged into the RFC repo as a markdown file. At that point the RFC
is 'active' and may be implemented with the goal of eventual inclusion
into Pachyderm.

* Fork the RFC repo http://github.com/pachyderm/rfcs
* Copy the appropriate template. For most RFCs, this is `0000-template.md`, 
for deprecation RFCs it is `0000-deprecation-template.md`.
Copy the template file to `text/0000-my-feature.md`, where
'my-feature' is descriptive. Don't assign an RFC number yet.
* Fill in the RFC. Put care into the details: **RFCs that do not
present convincing motivation, demonstrate understanding of the
impact of the design, or are disingenuous about the drawbacks or
alternatives tend to be poorly-received**.
* Submit a pull request. As a pull request the RFC will receive design
feedback from the larger community, and the author should be prepared
to revise it in response.
* Find a champion on the Pachyderm core team. The champion is responsible for 
shepherding the RFC through the RFC process.
* Update the pull request to add the number of the PR to the filename and 
add a link to the PR in the header of the RFC.
* Build consensus and integrate feedback. RFCs that have broad support
are much more likely to make progress than those that don't receive any
comments.
* Eventually, the core team will decide whether the RFC is a candidate
for inclusion in Pachyderm.
* RFCs that are candidates for inclusion in Pachyderm will enter a "final comment
period" lasting 7 days. The beginning of this period will be signaled with a
comment and tag on the RFC's pull request. When this happens, the champion will
post a message about the RFC into Pachyderm's slack channel to attract the
community's attention.
* An RFC can be modified based upon feedback from the core team and community.
Significant modifications may trigger a new final comment period.
* An RFC may be rejected by the core team after public discussion has settled
and comments have been made summarizing the rationale for rejection. The RFC 
will enter a "final comment period to close" lasting 7 days. At the end of the 
"FCP to close" period, the PR will be closed.
* An RFC may also be closed by the core team if it is superseded by a merged
RFC. In this case, a link to the new RFC should be added in a comment.
* An RFC author may withdraw their own RFC by closing it themselves.
* An RFC may be accepted at the close of its final comment period. A core team
member will merge the RFC's associated pull request, at which point the RFC will
become 'active'.

 
### Finding a champion

The RFC Process requires finding a champion from the relevant core teams. The 
champion is responsible for representing the RFC in team meetings, and for 
shepherding its progress. [Read more about the Champion's job](#champion-responsibilities)
 
- Slack
The `rfc` channel on the [Pachyderm Slack](https://slack.pachyderm.io) is 
reserved for the discussion of RFCs.
We highly recommend circulating early drafts of your RFC in this channel to both 
receive early feedback and to find a champion.  

- Request on an issue in the RFC repo or on the RFC
We monitor the RFC repository. We will circulate requests for champions but highly 
recommend discussing the RFC in Slack.  

## The RFC life-cycle

Once an RFC becomes active, the relevant teams will plan the feature and create 
issues in the relevant repositories.
Becoming 'active' is not a rubber stamp, and in particular still does not mean 
the feature will ultimately be merged; it does mean that the core team has agreed 
to it in principle and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor whether anybody is currently working on it.

Modifications to active RFC's can be done in followup PR's. We strive
to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged RFC to actually reflect what the end result will be at
the time of the next major release; therefore we try to keep each RFC
document somewhat in sync with the feature as planned,
tracking such changes via followup pull requests to the document.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active'
RFC, but cannot determine if someone else is already working on it,
feel free to ask (e.g. by leaving a comment on the associated issue).

## For Core Team Members

### Reviewing RFCs

The core team is responsible for reviewing open RFCs. Please ensure that
RFCs address any consequences, changes, or work required in your area
of responsibility.

As it is with the wider community, the RFC process is the time for 
teams and team members to push back on, encourage, refine, or otherwise comment 
on proposals.

### Referencing RFCs

- When mentioning RFCs that have been merged, link to the merged version, 
not to the pull-request.

### Champion Responsibilities

* achieving consensus from the core team to move the RFC through the stages of 
the RFC process.
* ensuring the RFC follows the RFC process.
* shepherding the planning and implementation of the RFC. Before the RFC is 
accepted, the champion may remove themselves. The champion may find a replacement 
champion at any time.

### Helpful checklists for Champions

#### Becoming champion of an RFC
- [ ] Assign the RFC to yourself

#### Moving to FCP to Merge
- [ ] Achieve consensus to move to "FCP to Merge" from relevant core team members
- [ ] Comment in the RFC to address any outstanding issues and to proclaim the 
start of the FCP period
- [ ] Post in the `#users` channel in Slack about the FCP 
- [ ] Ensure the RFC has had the filename and header updated with the PR number 

#### Move to FCP to Close
- [ ] Achieve consensus to move to "FCP to Close" from relevant core team members
- [ ] Comment in the RFC to explain the decision

#### Closing an RFC
- [ ] Comment about the end of the FCP period with no new info
- [ ] Close the PR

#### Merging an RFC
- [ ] Achieve consensus to merge from relevant core team members
- [ ] Ensure the RFC has had the filename and header updated with the PR number 
- [ ] Create a tracking project for the RFC implementation on https://github.com/pachyderm/pachyderm
- [ ] Update the RFC header with a link to the tracking project
- [ ] Merge
- [ ] Update the RFC PR with a link to the merged RFC (The `Rendered` links often
go stale when the branch or fork is deleted)
- [ ] Ensure relevant teams plan out what is necessary to implement
- [ ] Put relevant TODO items on the tracking project as notes. As these start being planned/worked on, convert to issues

**Pachyderm's RFC process owes its inspiration to the RFC processes of
[Rust](https://github.com/rust-lang/rfcs) and
[Ember.js](https://github.com/emberjs/rfcs/)**

