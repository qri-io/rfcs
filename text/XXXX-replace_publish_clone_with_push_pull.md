- Feature Name: Dataset Push & Pull
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Replace `publish/add` with `push/pull`, which default to pushing all data, pulling only the latest version. A new `clone` command combines `pull` with `checkout`.

# Motivation
[motivation]: #motivation

This RFC addresses two problems with the way we move versions around in qri:
1. datasets published to the registry are creating weird states where users can't get versions
2. the terms we use for moving versions around are confusing.

The first problem comes down to improper defaults. Publishing should default to pushing _as much data as the remote is willing to accept_. Publishers sending data to a willing host should be trying to keep the network as robust as possible by making lots of copies.

_Consumers_ of data on the other hand want just enough data to make use of the dataset, so the present default of only grabbing the latest version works _as a start_. Consumers are having problems satisfying deeper use cases that require prior versions because we don't present an easy API for common steps beyond grabbing the latest version of a dataset. It's currently too hard to:
* fetch all versions of a dataset
* fetch a specific historical version of a dataset
* re-fetch once a dataset update has been published


The second problem comes from poor metaphorical choices. Moving dataset histories around with "publish" and "add" is causing too much confusion. Making things worse, we're still stuck in a part-way transition to renaming "add" to "clone". What's worse, our concept of "clone" doesn't preently match

We should remove `add` and `publish`, replacing them with three new commands:
* push: upload datasets to qri peers
* pull: download datasets from the qri network
* clone: pull + checkout

I like push & pull because they're clear metaphors based on physical verbs. push & pull both describe who is initiating the action (you) and the directionality of the action. In this world clone becomes a porcelain command that properly matches the way git behaves.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## New Commands
I'm going to focus on writing help text that explains the proposed commands. The help text should describe the commands well enough to understand what this RFC is proposing.


### push: upload datasets to qri peers
```
$ qri push --help
Push sends datasets to a remote qri node. A push updates the dataset log on the
remote and sends as many versions as the remote is willing to accept. A remote
peer determines it's own rules on which versions to accept, but by default
favour the latest versions of a dataset.

If no remote is specified, qri pushes to the registry. But pushes 
can just as easily send to another peer that is configured to accept datasets 
from you. For details on  setting peers for dataset exchange read this doc:
https://qri.io/docs/hosting-datasets

Usage:
  qri push [REMOTE] DATASET [DATASET] [flags]

Aliases:
  publish

Examples:
  # push a dataset to the registry
  $ qri push me/dataset

  # push a specific version of a dataset to the registry:
  $ qri push me/dataset@/ipfs/QmHashOfVersion

  # push only the latest two versions to a peer named xiao
  $ qri push xiao me/dataset --revisions 2

Flags:
  -l, --logs               send only dataset history
  -r, --revisions          send only the lastest commit data
```

### pull: download datasets from the qri network
```
$ qri pull --help
Pull downloads datasets and stores them locally, fetching the dataset log and
dataset version(s). By default pull only fetches the latest version of a dataset

By default pull fetches 

Because pull grabs the latest version by default, it's common to 

If qri connect is running

Usage:
  qri pull [REMOTE] [DATASET] [flags]

Examples:
  # download a dataset log and latest version
  $ qri pull b5/world_bank_population

  # download a dataset log and all versions
  $ qri pull b5/world_bank_population --all

  # pull a dataset from 
  $ qri pull ramfox b5/world_bank_population

Flags:
  -l, --logs              save only dataset history
  -r, --revisions         fetch a set number of versions from the latest commit
  -a, --all               fetch all versions, overrides revisions
  -k, --keep-files        when in a working directory, don't update files

```

`revisions` and `all` flags mimic the way `qri remove` works.

### clone: pull and checkout a dataset

We already have "clone" language in qri cloud, and have intended to rename `add` to `clone` for some time. The problem is add isn't the same thing as clone when compared to git. Removing the idea of publish & unpublish gives us back a little cognitive overhead that we can use for the classic combo of "get the latest version and checkout a directory


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

* history syncing 
* dsync needs support for a manifest with multiple roots, multiple pins
* receivers may not honor all block push or pin requests
* pushing with `--latest` makes push behave like pull
* pulling with `--all` makes pull behave like default push
* All Peers are Potential Remotes
* IPFS is the natural fallback network

# Drawbacks
[drawbacks]: #drawbacks

### Push & Pull don't work like git
This is a problem, but I think the deviation is justified because we want different things from dataset version control than code version control. Datasets can get big. Versions of datasets can get bigger.

We need to make it _very_ clear in documentation that pulling a dataset doesn't get you all versions by default.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Make pull fetch all versions when the total size is below a certain threshold

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

Should pull use the list of remote sources