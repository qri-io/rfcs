- Feature Name: dataset logsets
- Start Date: 2019-07-01
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

_Note: This RFC one of a "collaboration round 1" suite of proposals:_
* _logsets (this RFC)_
* _encrypted datasets_
* _role-based access control_

_All three proposals build on each other, but serve distinct parts of needs related to collaborating on datasets._

# Summary
[summary]: #summary

refsets syncing of dataset-related operations, setting the stage for a suite of dataset collaboration features

# Motivation
[motivation]: #motivation

For some time we've worked on datasets as a single, versioned thing. It's time to take it up a notch & open up to the possibility of

When compared to git

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Here's a rough model of how dataset dataset version tracking currently works in Qri:

```
  causal time ->
                                      HEAD
                                      │
                                      v
  dataset version:  *<────*<────*<────*
  version number:   1     2     3     4
```

Commits are a chain of immutable versions, with each version pointing to the previous version. `HEAD` is associated with a _dataset name_ in the peer's namespace (for example: b5/world_bank_population), so when users refer to a dataset, they're referring to the HEAD of a version history. It's a simple & powerful system.

There are a number of limitations to this approach:
* Users cannot 

```
                                      HEAD
                                      │
                                      V

  refset:           ┌─────┬─────┬─────┐
                    |     |     |     │
                    |     |     |     │
                    v     v     v     v
  dataset version:  *<────*<────*<────*
  version number:   1     2     3     4
```

A _refset_ is an index of commit history.

Refsets are stored as conflict-free replicated data types

<!-- Explain the proposal as if it was already included in the language and you were teaching it to a Qri _developer_. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Qri developer should *think* about the feature, and how it should impact the way they use Qri. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to a Qri developer vs a Qri _user_.

For implementation-oriented RFCs (e.g. for Qri codebase internals), this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->

```

  Dataset Versions

```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Logfiles must be _concise_

High level:
* a dataset name now point to a logset instead of directly to a dataset head
* each dataset has one logset
* a number of peers can collaborate on the same dataset. peers
* logsets exist in a _peer namespace_, the namespace a logset exists within is the _owner_ of a logset (and the dataset t)
* logsets are formed via an operation-based Observed-Removed CRDT
* logset "entries" are called _operations_ must be size-bounded
* logsets are stored as a flatbuffer, and persisted via IPFS
* logsets _compress_ after a set number of records

CRDT Details:
* logsets use a per-replica vector clock

### Logset schema:

```

```

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

CRDT Research:


# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
