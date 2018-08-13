# Qri RFCs

[Qri RFCs]: #qri-rfcs

Many changes can be implemented and reviewed via the normal GitHub pull request 
workflow. Some changes though are "substantial", and we ask that these be put 
through a bit of a design process and produce a consensus among the qri 
community and core team.

The "RFC" (request for comments) process is intended to provide a consistent
and controlled path for charting the roadmap of qri. We've seen a number of 
projects in the distributed space suffer from under-considered design choices 
and unclear roadmapping, We're hoping strong adherence to a lightweight RFC 
process can help mitigate these problems. You should be able get a sense of 
where qri is going by reading through the accepted proposals.

We openly adknowledge this may seem premature for such an early-stage project.
We're intending to put this RFC place in process now to develop a design-driven
culture that others have a clear path to contribute to the future of 
the project.

This process is _deeply_ inspired by the [rust language RFC process](https://github.com/rust-lang/rfcs),
which builds on the [Python Enhancement Proposals process](https://www.python.org/dev/peps/),
a big thank-you to these projects for leading the way.


## Table of Contents
[Table of Contents]: #table-of-contents

  - [Opening](#qri-rfcs)
  - [Table of Contents]
  - [When you need to follow this process]
  - [Before creating an RFC]
  - [What the process is]
  - [The RFC life-cycle]
  - [Reviewing RFCs]
  - [Implementing an RFC]
  - [RFC Postponement]
  - [Help this is all too informal!]
  - [License]


## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

Most qri repositories follow [angular commit conventions](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#type)
which designates 8 _types_ of change:

  - **feat:** A new feature
  - **fix:** A bug fix
  - **docs:** Documentation only changes
  - **style:** Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
  - **refactor:** A code change that neither fixes a bug nor adds a feature
  - **perf:** A code change that improves performance
  - **test:** Adding missing or correcting existing tests
  - **chore:** Changes to the build process or auxiliary tools and libraries such as documentation generation

We only need RFCs for `feat` and breaking `refator` changes to any of our novel
projects, which are the [qri](https://github.com/qri-io/qri) and [dataset](https://github.com/qri-io/dataset)
repositories. `feat` and `refactor` changes to the [frontend](https://github.com/qri-io/frontend)
will also require heavy review, but don't require an RFC.

Most of the RFC's you'll see created will be coming from the core team, but 
that doesn't mean you can't chime in. Constructive one-off comments
are more than welcome!


## Before creating an RFC
[Before creating an RFC]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the RFC can
make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is
generally a good idea to pursue feedback from other project developers
beforehand, to ascertain that the RFC may be desirable; having a consistent
impact on the project requires concerted effort toward consensus-building.

The best place to start is by filing an issue on this repository, the core team
monitors this repo, and will provide feedback on your issue, including weather
it would make for a good RFC.


## What the process is
[What the process is]: #what-the-process-is

In short, to get a major feature added to Qri, one must first get the RFC
merged into the RFC repository as a markdown file. At that point the RFC is
"active" and may be implemented with the goal of eventual inclusion into Qri.

  - Fork the RFC repo [RFC repository]
  - Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is
    descriptive. don't assign an RFC number yet).
  - Fill in the RFC. Put care into the details: RFCs that do not present
    convincing motivation, demonstrate understanding of the impact of the
    design, or are disingenuous about the drawbacks or alternatives tend to be
    poorly-received.
  - Submit a pull request. As a pull request the RFC will receive design
    feedback from the larger community, and the author should be prepared to
    revise it in response.
  - Each pull request will be reviewed by the core team and assigned to a team
    member, who will take responsibility for guiding the RFC
  - Build consensus and integrate feedback. RFCs that have broad support are
    much more likely to make progress than those that don't receive any
    comments. Feel free to reach out to the RFC assignee in particular to get
    help identifying stakeholders and obstacles.
  - The core team will discuss the RFC pull request, as much as possible in the
    comment thread of the pull request itself. Offline discussion will be
    summarized on the pull request comment thread.
  - RFCs rarely go through this process unchanged, especially as alternatives
    and drawbacks are shown. You can make edits, big and small, to the RFC to
    clarify or change the design, but make changes as new commits to the pull
    request, and leave a comment on the pull request explaining your changes.
    Specifically, do not squash or rebase commits after they are visible on the
    pull request.
  - If the proposal is submitted by a core team member, it can be merged
    by at least 2 other core team members approving the RFC, otherwise a member 
    of the core team will propose a "motion for final comment period" (FCP), 
    along with a *disposition* for the RFC (merge, close, or postpone).
    - This step is taken when enough of the tradeoffs have been discussed that
    the core team is in a position to make a decision. That does not require
    consensus amongst all participants in the RFC thread (which is usually
    impossible). However, the argument supporting the disposition on the RFC
    needs to have already been clearly articulated, and there should not be a
    strong consensus *against* that position outside of the core team. Team
    members use their best judgment in taking this step, and the FCP itself
    ensures there is ample time and notification for stakeholders to push back
    if it is made prematurely.
    - For RFCs with lengthy discussion, the motion to FCP is usually preceded by
      a *summary comment* trying to lay out the current state of the discussion
      and major tradeoffs/points of disagreement.
    - Before actually entering FCP, *all* members of the core team must sign off;
    this is often the point at which many core team members first review the RFC
    in full depth.
  - The FCP lasts ten calendar days, so that it is open for at least 5 business
    days. It is also advertised widely,
    e.g. in [This Week in Qri](https://this-week-in-rust.org/). This way all
    stakeholders have a chance to lodge any final objections before a decision
    is reached.
  - In most cases, the FCP period is quiet, and the RFC is either merged or
    closed. However, sometimes substantial new arguments or ideas are raised,
    the FCP is canceled, and the RFC goes back into development mode.

## The RFC life-cycle
[The RFC life-cycle]: #the-rfc-life-cycle

Once an RFC becomes "active" then authors may implement it and submit the
feature as a pull request to the Qri repo. Being "active" is not a rubber
stamp, and in particular still does not mean the feature will ultimately be
merged; it does mean that in principle all the major stakeholders have agreed
to the feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a Qri developer has been assigned the task of
implementing the feature. While it is not *necessary* that the author of the
RFC also write the implementation, it is by far the most effective way to see
an RFC through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We
strive to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect every
merged RFC to actually reflect what the end result will be at the time of the
next major release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new RFCs, with a note added to the original RFC. Exactly what counts
as a "very minor change" is up to the sub-team to decide; check
[Sub-team specific guidelines] for more details.


## Reviewing RFCs
[Reviewing RFCs]: #reviewing-rfcs

While the RFC pull request is up, the core team may schedule meetings with the
author and/or relevant stakeholders to discuss the issues in greater detail,
and in some cases the topic may be discussed at a core team meeting. In either
case a summary from the meeting will be posted back to the RFC pull request.

The core team makes final decisions about RFCs after the benefits and drawbacks
are well understood. These decisions can be made at any time, but the core team
will regularly issue decisions. When a decision is made, the RFC pull request
will either be merged or closed. In either case, if the reasoning is not clear
from the discussion in thread, the sub-team will add a comment describing the
rationale for the decision.


## Implementing an RFC
[Implementing an RFC]: #implementing-an-rfc

Some accepted RFCs represent vital features that need to be implemented right
away. Other accepted RFCs can represent features that can wait until some
arbitrary developer feels like doing the work. Every accepted RFC has an
associated issue tracking its implementation in the Qri repository; thus that
associated issue can be assigned a priority via the triage process that the
team uses for all issues in the Qri repository.

The author of an RFC is not obligated to implement it. Of course, the RFC
author (like any other developer) is welcome to post an implementation for
review after the RFC has been accepted.

If you are interested in working on the implementation for an "active" RFC, but
cannot determine if someone else is already working on it, feel free to ask
(e.g. by leaving a comment on the associated issue).


## RFC Postponement
[RFC Postponement]: #rfc-postponement

Some RFC pull requests are tagged with the "postponed" label when they are
closed (as part of the rejection process). An RFC closed with "postponed" is
marked as such because we want neither to think about evaluating the proposal
nor about implementing the described feature until some time in the future, and
we believe that we can afford to wait until then to do so. Historically,
"postponed" was used to postpone features until after 1.0. Postponed pull
requests may be re-opened when the time is right. We don't have any formal
process for that, you should ask members of the relevant sub-team.

Usually an RFC pull request marked as "postponed" has already passed an
informal first round of evaluation, namely the round of "do we think we would
ever possibly consider making this change, as outlined in the RFC pull request,
or some semi-obvious variation of it." (When the answer to the latter question
is "no", then the appropriate response is to close the RFC, not postpone it.)


### Help this is all too informal!
[Help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the present
circumstances. As usual, we are trying to let the process be driven by
consensus and community norms, not impose more structure than necessary.

[RFC repository]: http://github.com/qri-io/rfcs

## License
[License]: #license

This repository is licensed under the MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

### Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted 
for inclusion in the work by you, as defined in the shall be MIT licensed, 
without any additional terms or conditions.
