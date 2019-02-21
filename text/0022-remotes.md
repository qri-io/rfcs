- Feature Name: Remotes
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Remotes act as a way for any user of Qri to setup their own server that keeps datasets alive, providing availability and ownership over data within a set of nodes that they control.

# Motivation
[motivation]: #motivation

Multiple users have requested a way to keep datasets alive and available within their own network. Currently, we have a public Registry that serves a similar purpose, but it acts too centralized and is also not designed to be duplicated and deployed by existing users. Although IPFS may keep data blocks alive due to its distributed nature, there's no guarantee to keep data around forever unless it is pinned, and the pinning node remains online. Remotes solve this problem by giving control to users, letting them own their data. Having it work as a do-it-yourself service goes a long way torwards turning Qri into a descentralized (as well as distributed) service.

Relatedly, the concept of the Registry, as it is now, should be deprecated and eventually removed. What is acting as the Registry now should be moved over to a normal Qri backend node that is acting as a remote.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Some definitions used in this document:

* Registry - the Qri-run service for pinning, searching, etc
* Remote - the future functionality included in the main Qri binary
* Client - a Qri command-line program that get commands from stdin
* Backend - a Qri program that is not the Electron frontend

Currently, the Registry exists to solve multiple problems centered around a distributed system's lack of centralization:

* User identity and registration
* Search
* Availability

The Registry acts in a "semi-centralized" way: it provides these types of features, but is not strictly required to use Qri by itself. However, there is currently no way for an average user to run something like the Registry on their own.

Currently functionality that exists only in the Registry now includes:

* `POST /dataset` to upload a dataset's head
* `/search` to search over known datasets (online or not)
* `/datasets` to list all known datasets (online or not)
* `/profile` user information
* `/reputation` user trustability
* `/pins` pinning a dataset
* `/dsync` data block uploading

Long-term we want to obliviate the need for these to be in a separate executable.

The plan of action is as follows:

* Add a flag to Qri's backend to run it as a remote. This will work similarily to the current "read-only" flag, disabling most write functionality.
* Add APIs to the backend that search a similar purpose to what the Registry is doing now: `POST /dataset` in particular.
* Get enough APIs in the backend such that it can be run as a remote which provides availability to users that want it.
* Add functionality to Qri's command-line to publish to this remote instead of, or in addition to the public Registry.
* Over time, move the rest of the functionality from the Registry into the backend remote.
* Replace the live Registry with a standard remote.
* Delete old Registry code.
* New semi-centralized features get added to the Remote feature set.

Some specific notes as to how this is presented. First, the concept of the Registry is a good way to explain Qri to users: it exists as a way to keep data online even if your own node is not. This shouldn't change as we move to Remotes; instead of deleting the _concept_ of a Registry, we're generalizing it by introduced a Remote as something you can run yourself if you're an advanced user with a special use-case. Secondly, since a Remote is considered an expert-level concept, it's okay for its UX to be slightly more advanced than it may be with other features. The easy thing to use should remain to be the Qri run public Registry. This implies, for example, that setting up a client to use a remote should be more like `git config set remote <url>` instead of `git remote add <name> <url>`, with `remote` as a top-level command.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Posting a dataset to a Remote involves expanding our current protocol:

* Client sends Dataset Manifest and Head to the Remote, signed with their cryptographic key
* The Remote has some ruleset about what it accepts, for example, no more than 10Mb in a body
* Remote replies with what it wants from the user
* DAGSync occurs (or not, if the dataset body is rejected)
* Dataset is pinned
* Remote tells the client about the result

In the future, advanced users may want to configure a Remote with full granularity how an acceptance ruleset works. Perhaps they could even use `starlark` as a way to decide this.

# Drawbacks
[drawbacks]: #drawbacks

Centralization is bad, which is why we need to be careful to make Remotes remain an optional feature that users are not forced into.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The current version of the Registry as a special citizen is not working for multiple reasons. It must be coalesed into the main codebase.

# Prior art
[prior-art]: #prior-art

Github plays a similar role to git. However, it is already possible for a user to run their own git server, albiet without the nice features like Issues and Pull Requests.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Access controls




