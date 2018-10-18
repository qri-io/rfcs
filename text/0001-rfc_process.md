- Feature Name: rfc_process
- Start Date: 2018-08-13
- RFC PR: #1
- Issue: N/A

# Summary
[summary]: #summary

The "RFC" (request for comments) process is intended to provide a consistent
and controlled path for charting the roadmap of qri. We've seen a number of 
projects in the distributed space suffer from under-considered design choices 
and unclear roadmapping, We're hoping strong adherence to a lightweight RFC 
process can help mitigate these problems. Anyone should be able get a sense of 
_where qri is going_ by reading through the accepted proposals.

# Motivation
[motivation]: #motivation

I openly adknowledge this may seem premature for such an early-stage project.
I'm intending to put this RFC place in process now to develop a design-driven
culture so that others have a clear path to contribute to the future of the 
project, and replace myself as the arbiter of "what qri is" with a codified 
principle of [rough consensus and working code](https://www.ietf.org/about/participate/tao/).

One of the main concerns I have is ambiguity exists around what is _expected 
behaviour_ when interacting with qri. To me, this is a _major_ problem. 
Not knowing precisely what working-as-expected looks like cuts down on what core 
team members can confidently reason about, which means the core can't
confidently tell _others_ how qri should work, which means the whole project is
both ambiguous and without a clear path for resolving that ambiguity.

To me this ambiguity is a sign that we (the core team) have haven't taken enough 
time to reach design-level consensus about how qri _should_ behave, and why I 
think it's not too early to start this process.

There are a few key motivations for implementing this now: 
  - develop a culture of rough consensus
  - inject more design-time thinking into the core team's process
  - maintain an accurate, reliable roadmap
  - create an on-ramp for others to contribute to the design of qri

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

For the core team adopting this RFC process will feel like three things at once:
  - a source of truth for _how qri should behave_
  - a new channel to directly impact the design of qri
  - lots of additional work to get new stuff into qri

The first batch of RFCs should ideally come from me (@b5), outlining in detail
how the various aspects of qri _should_ work (which as we know, is different 
from how things do/not work). The process of debating these initial RFCs should 
serve as a chance to work together to establish language, clarity & detail 
around expected behaviours. Accepting these initial RFCs provides a new,
consensus-driven foundation for the project that should give core team members
a confident, consistent source of truth to work from.
By requiring sign-off from the core team, this should be a chance for the
core team to achieve consensus on how things should work.

Upon accepting this proposal, we'll also move all of the issues filed in 
[qri-io/notes](https://github.com/qri-io/notes) & depricate the repo. If we
accept this RFC process, we should commit to it fully. Notes should now
start as issues on this repo with the intention that they could develop into 
formal RFCs.

At the same time, this should also be a chance for others to start documenting
ideas we've been kicking around for enhancing qri. Things we've discussed in the
past like skylark modules, simulation testing, new file format support, 
etc. should be collected as issues on this repo so we can start working them
into RFCs.

Finally this process should not get in the way. If done properly, day-to-day 
development should  _accelerate_ once we've accepted enough RFCs to get 
proposals ahead of current development.  
It should be easier to implement a feature with confidence because implementing 
new things should just be coding up an already-approved design document.
I want to save RFCs for design-level features, not day-to-day fixes or 
implementation details. It should still be perfectly acceptable to make changes 
on interfaces between packages, move packages around, and riff on ideas outside 
of the formal RFC process. RFCs should be saved for decisions that affect how we 
expect the user-facing edges of qri to behave.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

_Most of this is copied from the Rust RFC process, except I've added a section
intended to keep us moving quickly: if a core team member creates an RFC, and
two other core team members approve it, it's automatically merged. We should
remove this provision once qri reaches a 1.0 release._

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
    days. This way all stakeholders have a chance to lodge any final objections 
    before a decision is reached.
  - In most cases, the FCP period is quiet, and the RFC is either merged or
    closed. However, sometimes substantial new arguments or ideas are raised,
    the FCP is cancelled, and the RFC goes back into development mode.

### The RFC life-cycle
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

### Reviewing RFCs
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


### Implementing an RFC
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


# Drawbacks
[drawbacks]: #drawbacks

This is going to slow us down and consume precious time.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

On a philosophical level, I'm tired of shitty software that doesn't work. I want
to raise our standards, aspiring toward principles of 
["Zero Defects Programming"](https://en.wikipedia.org/wiki/Zero_Defects):

> Instill in workers the will to prevent problems during design and manufacture
rather than go back and fix them later

The only way to prevent problems during the design phase is to **require a 
design phase for things that matter**. In the context of Open Source Software, 
the RFC process is the best I've seen for having a strong design that yields 
software others can depend on.

# Prior art
[prior-art]: #prior-art

- [Rust RFC Process](https://github.com/rust-lang/rfcs)
- [IETF Standards Process](https://www.ietf.org/standards/process/)
- [Python Enhancement Proposals](https://www.python.org/dev/peps/)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What structures need to be in place to align RFCs with the mission of the
project?
- Do we need to somehow distill approved RFCs into a single roadmap document?
