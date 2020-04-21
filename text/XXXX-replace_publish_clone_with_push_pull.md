- Feature Name: Dataset Push & Pull
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Replace `publish/add` with `push/pull`, which default to pushing all data, pulling only the latest version.

# Motivation
[motivation]: #motivation

This RFC addresses two problems with the way we move versions around in qri:
1. datasets published to the registry are creating weird states where users can't get versions
2. the terms we use for moving versions around are confusing.

The first problem comes down to improper defaults. Publish currently sends _only the latest version_ of a dataset. If a user creates five versions, then runs publish, only the fifth version is pushed, leaving any party intersted in earlier versions with only the author as a viable source of versioned data. If the author isn't online and dialable, those prior versions are unavailable to the network. On the upside, our work on logbook and log syncronization means that at least the _history_ of the dataset is visible.

Publishing should default to pushing _as much data as the remote is willing to accept_. Publishers sending data to a willing host should be trying to keep the network as robust as possible by making lots of copies.

_Consumers_ of data on the other hand want just enough data to make use of the dataset, so the present default of only grabbing the latest version works _as a start_. Consumers are having problems satisfying deeper use cases that require prior versions because we don't present an easy API for common steps beyond grabbing the latest version of a dataset. It's currently too hard to:
* fetch all versions of a dataset
* fetch a specific historical version of a dataset
* re-fetch once a dataset update has been published

The second problem comes from poor metaphore choices. Moving dataset histories around with "publish" and "add" doesn't map properly to the terms used. "publish" is putting versions of a dataset on a remote peer, and it's really only a "publish" if that peer elects to broadcast the existince of that dataset to other peers, and misses the main thing publish does: place a dataset version on a remote.

Making things worse, we're still stuck in a part-way transition to renaming "add" to "clone". What's worse, our concept of "clone" doesn't presently match what happens within git: `git-clone` both downloads _and checks out the repo into a target directory_. To trans

We should remove `add` and `publish`, and replace them with two new commands:
* push: upload datasets to qri peers
* pull: download datasets from the qri network

Push & pull improve on `add/publish` because they're clear metaphors based on physical verbs. push & pull both describe _who_ is initiating the action (you) and the _directionality_ of the action:

* push: me - data -> you
* pull: me <- data - you

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
  qri push [REMOTE] DATASET [DATASET...] [flags]

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

Because pull grabs the latest version by default, it's common to 

If qri connect is running

Usage:
  qri pull [REMOTE] DATASET [DATASET...] [flags]

Examples:
  # download a dataset log and latest version
  $ qri pull b5/world_bank_population

  # download a dataset log and all versions
  $ qri pull b5/world_bank_population --all

  # pull a specific version from a remote by hash
  $ qri pull ramfox b5/world_bank_population@/ipfs/QmFoo...

Flags:
  -l, --logs              download only dataset history
  -r, --revisions         fetch a set number of versions from the latest commit
  -a, --all               fetch all versions, overrides revisions
  -k, --keep-files        when in a working directory, don't update files
```

`revisions` and `all` flags mimic the way `qri remove` works.


### checkout can pull

We already have a checkout command, this RFC proposes that checkout will do the default of `qri pull` if a user tries to checkout a dataset they don't have.

```
$ qri checkout --help
Create a linked directory and write dataset files to that directory. 

If the dataset you are checking out isn't in the qri database, the latest 
version of the dataset will be pulled before checking out.

Usage:
  qri checkout DATASET [flags]

Examples:
  # Place a copy of me/annual_pop in the ./annual_pop directory:
  $ qri checkout me/annual_pop

Flags:
  -h, --help      help for checkout
      --offline   don't fetch datasets from the network

Global Flags:
      --ipfs-path string   override IPFS path location (default "/Users/b5/.ipfs")
      --log-all            log all activity
      --no-color           disable colorized output
      --no-prompt          disable all interactive prompts
      --repo string        provide a path to load qri from (default "/Users/b5/.qri")
```

### No more clone
We already have "clone" language in qri cloud, and the original plan has been to rename `add` to `clone`. In discussion we've come to realize The problem is add isn't the same thing as clone when compared to git. Git needs `clone` because git does not have a built-in naming system. Clone copies a git repo _from a place_, usually a URL. 

Because qri can resolve dataset names without additional information from the user we can add `pull` directly to `checkout`, and drop the term `clone` from our lexicon. We can use this saved term later for something like "following" a dataset. See the unsreolved questions section for more info.

### Automatic pulling & Networking UX
Giving `checkout` the power to pull `pull` when data isn't local is the first concrete example of _automatic pulling_. This is a shift in our intended user experince. We've wrestled with weather qri should default to making network requests for users. `qri --help` has a set of "Network commands", implying the other commands in qri _don't_ use the network. This breaks that assumption.

Automatic pulling requires us to make a decision on this. In this regard, I think qri should work like a package manager, automatically making network requests when the user seeks datasets they don't have. `npm install some_package` downloads from the web. Both npm and qri have name resolution systems that make this process possible. From a UX perspective npm lets me _ignore_ the network and focus on writing software.

We've struggled with the word "network" because unlike npm, qri also has a peer-2-peer system.

I want to live in a world where I can punch a command into `qri sql` that references a dataset & don't have. Qri should figure out what I'm talking about, go find that dataset, pull the latest version, and run the query. There are questions about how this can and should work that require an RFC like this to get to.

### Remotes default to a `none` retention strategy
When pushing a dataset, the receiver is in charge of what they will accept. A remote can reject a push request for any reason. One of the primary reasons for rejecting a push is do to size constraints. Qri has a configuration value `remote.acceptSizeMax` that currently polices the maximium size of a dataset version. Any dataset version that exeeds this value is rejected by default.

This RFC proposes changing the `remote.acceptSizeMax` value in configuration to apply to the on-disk size of _the entire dataset history_. As an example: if `acceptSizeMax` is `300Mb` that quota can be filled with one version that is `300Mb` in size, three totally unique `100Mb` versions, or any other combination where the deduplicated size of all versions is below the maximum threshold. 

We say totally unique versions because qri de-duplicates versioned data. A block of dataset data may be used in any number of versions, and should only count towards the size threshold once.

this RFC introduces the term _retention strategy_ to describe the approach a peer takes to 

If the user requests to push N versions for Y data size, the typical response is to accept all pushed verions, so long as the total disk space consumed is less than `remote.acceptSizeMax`. When a user tries to push a number of versions above or say you are Z over the bar
In which case you can reduce the number of versions until bellow the threshold
Don’t think cloud should be aware of any of your versions until you ask to push them

### Exceeding max storage on a remote errors by default


The way we actually solve that situation is the subject of another RFC on storage. However we do it, a few characteristics will be true

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

* history syncing 
* dsync needs support for a manifest with multiple roots, multiple pins
* receivers may not honor all block push or pin requests
* pushing with `--latest` makes push behave like pull
* pulling with `--all` makes pull behave like default push
* All Peers are Potential Remotes
* IPFS is the natural fallback network

```
TODO (b5) - finish reference-level explanation
```

### Increasing d web reliance
We want to be defaulting to using the decentralized network more. Don't know how we're going to do this exactly, but cleaning up the way we move datasets around should put more hashes on more peers, which is a good thing for network health.

# Drawbacks
[drawbacks]: #drawbacks

### Push & Pull don't work like git
This is a problem, but I think the deviation is justified because we want different things from dataset version control than code version control. Datasets can get big. Versions of datasets can get bigger.

We need to make it _very_ clear in documentation that pulling a dataset doesn't get you all versions by default.

### Default of no retention strategy can leave users stranded

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Use a default version retention strategy a rolling window from HEAD
One alternative we 

### Make pull fetch all versions when the total size is below a certain threshold
This would add convenice when the storage cost is trivial. This can definitely be added later

# Prior art
[prior-art]: #prior-art

* git
* docker
* LOCKSS
* npm / yarn

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Push Permissions
Currently we don't have an easy API for "I'll accept datasets from this person", which will be a prerequisite for any of this working. There's nuance here, and it's closely related to access control. I'll file a separate RFC for this, but the short version is we should use an simple configuration-based allow-list as a start, then layer on something more robust as part of a broader access control project.

### Retention Strategies
This RFC only defines a default behaviour for `remote.acceptSizeMax` is exceeded. There are other behaviours

strategies for dealing with 
* none: error when quota is filled
* archival: retain from initial commit forward, error if chain of versions is broken
* rolling-window: keep the latest, drop older versions

In discussions we've come to realize that retention strategies are a major concern for the health of the qri network as a whole. For example: if all peers default to rolling-window strategy, we end up with a network of actors that "forget" early versions when the max size is exceeded. This destroys the auditability of a dataset, because no one retains the history.

A _retention_ strategy is also a _replication_ strategy. Put another way: which versions peers choose to keep determines the versions availabile to the network.

### Following
Retention Strategy + Automatic Pulling = dataset following.

I think the thing that should replace `clone` long-term is the idea of "following" a dataset, which will check for updates to a dataset when a user goes online, and automatically update dataset versions according to the retention strategy. Use a `rolling-window` retention strategy means always having the most up-to-date data without having to do anything. 

Following a dataset feels natural in an era of social media. A sizable network of peers following datasets they are interested in would mean that 

This implies version retention strategies can also be applied to the "pull" side of a dataset, and may affect the way we define retention strategies in configuration.

### Manual Storage Manipulation
Before we can get to retention strategies at all, we need methods for manually evaluating and pruning storage locally, and making requests to remotes for storage manipulation.