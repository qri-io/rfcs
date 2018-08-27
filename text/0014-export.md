- Feature Name: Export Dataset
- Start Date: 2018-08-27
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Peers need to be able to get q

# Motivation
[motivation]: #motivation 

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->
In order for Qri to be useful to a large swath of folks, we need a well 
defined way to get datasets out of Qri. Our two methods for doing so are the
`/export` api endpoint and the `qri export` command. Both methods should have
the same options, level of control, and outputs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

<!-- Explain the proposal as if it was already included in the language and you were teaching it to a Qri _developer_. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Qri developer should *think* about the feature, and how it should impact the way they use Qri. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to a Qri developer vs a Qri _user_.

For implementation-oriented RFCs (e.g. for Qri codebase internals), this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->
Export should allow users to export one, some, or all of:
  - header (meta, structure, commit)
  - body, or a section of it
  - transform
  - viz
  - (rendered version of dataset in html?)

Export should allow users to export the dataset header as:
  - json
  - yaml

Export should allow users to export the dataset body as:
  - xlrs
  - csv
  - json
  - cbor
  - yaml

Export should allow users to export the transform as:
  - the format it was imported as

Export should allow users to export the viz as
  - html

(Export should allow users to export the render as:
  - html)

Export should allow users to export one, some, or all of the dataset as a zip

Export should allow users to specify an export path. It should default to:
  - `qri export`: `/working_directory/dataset_name`
  - `qri export` a peer's dataset: `/working_directory/peername/dataset_name`
  - `/export`: `/Downloads/dataset_name`

Export should export blank/templated versions of the different dataset
sections:
  - `qri export --blank header`
  - `qri export --blank transform`
  - `qri export --blank viz`









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
