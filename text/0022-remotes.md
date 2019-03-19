- Feature Name: Remotes
- Start Date: 02-21-2019
- RFC PR: [#36](https://github.com/qri-io/rfcs/pull/36)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Remotes act as a way for any user of Qri to setup their own server that keeps datasets alive, providing availability and ownership over data within a set of nodes that they control.

# Motivation
[motivation]: #motivation

Multiple users have requested a way to keep datasets alive and available within their own network. Currently, we have a public Registry that serves a similar purpose, but it acts too centralized and is also not designed to be duplicated and deployed by existing users. Although IPFS may keep data blocks alive due to its distributed nature, there's no guarantee to keep data around forever unless it is pinned, and the pinning node remains online. Remotes solve this problem by giving control to users, letting them own their data. Having it work as a do-it-yourself service goes a long way torwards helping Qri fulfill its goal of putting data everywhere.

Relatedly, the current implementation of the Registry should be reworked so that it isn't duplicating work done inside of the normal Qri backend. Rather, it should simply be a variation of a Remote. By avoiding code duplicated across code bases, we will make maintenance easier and have a better story to explain how Qri works.

The eventual goal is to allow advanced users to run their own Remotes, both to keep data alive and provide certain federated services, while the Registry exists as a Qri run service and is the default location for publishing data and centralization.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Some definitions used in this document:

* Registry - the Qri-run service for pinning, searching, etc
* Remote - the future functionality included in the main Qri binary
* Backend - the main Qri command-line program
* Client - a Qri command-line program that lets users run their own command, as opposed to running as a server that receives remote commands

Currently, the Registry exists to solve multiple problems that are the result of Qri being primarily a distributed system:

* User identity and registration
* Search
* Data availability

The Registry acts to solve these problems, the first two of which require moving away from a purely distributed network model, and the third of which is an easy thing to add once a centralized component exists. While the Registry isn't strictly required to use Qri, as Qri can be run entirely in p2p mode, the tradeoff would be losing these types of features.

Our immediate benefit of creating Remotes is that we can unlink these features, and let users run their own services to provide better availability for their datasets. In addition, while a Remote can't provide global search or user identity, it can support a federated model, or a limited version within a specific network.

Once Remotes exist, and contain a sufficient amount of capability, we hope to reposition the Registry as just a special type of Remote that is run by Qri, and is used as the default for users, in addition to its necessary role of handling global user identity and search.

## Proposed workflow

Assuming a new Qri remote is up and running, users who wish to push to a remote would have to do three things:

1. Find the URL of a running remote, (let's pretend a remote is running at `https://example.com`)
2. Run `qri config add remotes.eg https://example.com`. This would create a new remote named `eg`
3. Publish a dataset using `qri publish --remote eg`

Running the default `qri publish` would still push to the default registry. Users should be able to publish to multiple remotes with many `remote` flags, or by separating remote names with commas.

Users can pick whatever name they would like for remotes. Remotes will initially need to be set up with DNS records we'll describe in detail later. Setting up a remote will be an advanced thing for a while.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Currently functionality that exists only in the Registry now includes:

* `POST /dataset` to upload a dataset's head
* `/search` to search over known datasets (online or not)
* `/datasets` to list all known datasets (online or not)
* `/profile` user information
* `/reputation` user trustability
* `/pins` pinning a dataset
* `/dsync` data block uploading

Long-term we want to obviate the need for these to be in a separate executable.

The plan of action is as follows:

* Add a flag to Qri's backend to run it as a remote. This will work similarly to the current "read-only" flag, disabling most write functionality.
* Add APIs to the backend that serve a similar purpose to what the Registry is doing now: `POST /dataset` in particular.
* Get enough APIs in the backend such that it can be run as a remote which provides availability to users that want it.
* Add functionality to Qri's command-line to publish to this remote instead of, or in addition to the public Registry.
* Over time, move the rest of the functionality from the Registry into the backend remote.
* Replace the live Registry with a standard remote.
* Delete old Registry code.
* New semi-centralized features get added to the Remote feature set.

Some specific notes as to how this is presented. First, the concept of the Registry is a good way to explain Qri to users: it exists as a way to keep data online even if your own node is not. This shouldn't change as we move to Remotes; instead of deleting the _concept_ of a Registry, we're generalizing it by introduced a Remote as something you can run yourself if you're an advanced user with a special use-case. Secondly, since a Remote is considered an expert-level concept, it's okay for its UX to be slightly more advanced than it may be with other features. The easy thing to use should remain to be the Qri run public Registry. This implies, for example, that setting up a client to use a remote should be more like `git config set remote <url>` instead of `git remote add <name> <url>`, with `remote` as a top-level command.

## Posting protocol

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

Adding remotes to the mix adds an additional layer of complexity. Specifically, the "is my dataset available" question is now a little harder to answer consistently, given that it'll depend on the updtime of the remote. Complexity aside, this problem is the same as it is now, but this heightens the need for a clearer story around dataset availability measurement.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The current version of the Registry as a special citizen is not working for multiple reasons. It must be coalesed into the main codebase. Without having Remotes, users must rely upon the single qri-run Registry or their own nodes to keep their data alive, which does not fit many use cases.

# Prior art
[prior-art]: #prior-art

Github plays a similar role to git. However, it is already possible for a user to run their own git server, albeit without the nice features like Issues and Pull Requests.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Access controls, federation


