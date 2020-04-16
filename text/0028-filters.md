- Feature Name: qri_filters
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

qri filters is a query language for shaping datasets. It de

# Motivation
[motivation]: #motivation

Filters shape data in qri datasets. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


[JQ](https://stedolan.github.io/jq/) syntax.

Some examples:
```
```

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

There is no shortage of query langauges 

### SQL

### GraphQL

### jQuery / CSS Query Selectors

### [XPath](https://en.wikipedia.org/wiki/XPath) / [XQuery](https://en.wikipedia.org/wiki/XQuery)

from [wikipedia](https://en.wikipedia.org/wiki/XQuery):
> All XQuery expressions operate on sequences, and evaluate to sequences. Sequences are ordered lists of items. Items can be either nodes, which represent components of XML documents, or atomic values, which are instances of XML Schema base types like xs:integer or xs:string. Sequences can also be empty, or consist of a single item only. No distinction is made between a single item and a singleton sequence

### Pandas

### NumPy

## Why JQ:


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
