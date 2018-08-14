- Feature Name: registries
- Start Date: 2018-04-12
- RFC PR: 
- Issue:

# Summary
[summary]: #summary

Registry defines primitives for keeping centralized repositories of qri types 
(peers, datasets, etc). It uses classical client/server patterns, arranging 
types into cannonical stores.

# Motivation
[motivation]: #motivation

It is a long term goal at qri that it be *possible* to fully decentralize all 
aspects of qri this isn't practical short-term, and isn't always a desired 
property.

As an example, associating human-readable usernames with crypto keypairs is an 
order of magnitude easier if you just put the damn thing in a list. So that's 
what this registry does.

Long term, we intended to implement a distributed hash table (DHT) to make it 
possible to operate fully-decentralized, and provide registry support as a 
configurable detail.

This base package provides common primitives that other packages can import to 
work with a registry, and subpackages for turning these primitives into usable 
tools like servers & (eventually) command-line clients

At first glance, this seems to run against the grain of "decentralize or die" 
principles espoused by those of us interested in reducing points of failure in 
a network. Consider this package testiment that nothing is absolute.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

This cuts against the grain of a lot of what we do, and introduces another
concept that needs to be explained and maintained.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could implment regestries as an opt-in aspect of qri at the P2P layer.

# Prior art
[prior-art]: #prior-art

- [Mastadon Federated Server Model]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO