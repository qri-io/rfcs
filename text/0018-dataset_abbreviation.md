- Feature Name: dataset abbreviation
- Start Date: 2018-10-25
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Define the different names, limitations, and use cases for subsets of dataset information

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->
Datasets can theoretically be any size, because of this

By defining these formally, there's a better chance we'll know when to use which type of abbreviation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

| name      | description | example use cases | byte size |
| --------- | ----------- | ----------------- | --------- |
| hash      | the content-addressed dataset identifier | | |
| reference | a dataset name | | |
| preview   | a fixed size description of a dataset | | |
| head      | all dataset content except the body | | |
| partial   | a subsection of a dataset, including rows from a body, or a subsection of the head | | |
| document  | the full dataset document | | | |
| history   | the full dataset document and all previous verions of a dataset | | | |


<!-- Explain the proposal as if it was already included in the language and you were teaching it to a Qri _developer_. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Qri developer should *think* about the feature, and how it should impact the way they use Qri. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to a Qri developer vs a Qri _user_.

For implementation-oriented RFCs (e.g. for Qri codebase internals), this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other places and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other projects, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that Qri sometimes intentionally diverges from other projects. -->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
