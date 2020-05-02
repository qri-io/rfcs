- Feature Name: access_command
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Decouple dataset visibility from data pushing, moving `publish` & `--unpublish` into a new subcommand: `access`. Position `access` for future commands.

# Motivation
[motivation]: #motivation

In the current version of qri (v0.9.8) "listing" a dataset so other peers can see it is tangled up with moving a dataset around. Both actions are tightly coupled in the `publish` command, and can be untangled with flags like `--no-registry`, which prevents publish from pushing data, leaving only the listing behaviour.

Publish is the only way to send datasets somewhere else, and the way publish is set up gives us very little room to grow. `publish` works well as a one-liner for "put this public data on qri.cloud", but falls short for other uses on the near-term roadmap. Some examples:

* send a dataset to a trusted peer without listing it
* send encrypted data to the registry
* ask a peer to _stop_ listing a dataset sent to them for others to see

All are examples of _unlisted collaboration_, where _sending_ and _listing_ data are independant actions. Forcing these use cases through the `publish` command yields strange results (like having to "publish" encrypted data).

To accommodate more use cases, we should flip the mental model, prioritizing thinking about _where data is being sent_. `publish` should join a family commands for managing _access control_. Sending data to a public destination should imply public listing. In other words: publishing should be an automatic part of pushing, instead of pushing being an automatic part of publishing.

The actual process of moving data around is the subject of another RFC on pushing and pulling. In this RFC we tackle how to manage publication status of a dataset.

This RFC proposes four concrete changes:
* create a new `access` subcommand to view & edit access control for users & datasets
* move `publish` and `unpublish` into access, changing both commands to edit _only_ `list` visibility (not push data)
* add an `info` command under access to show access info for a dataset
* pushing to the registry automatically runs "publish". dropping from the registry automatically unpublishes

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


### The Access Subcommand:
Here's the proposed access subcommand `help`:

```
$ qri access --help
Access controls visibility and permissions for datasets.

Usage:
  qri access [command]

Available Commands:
  info           show access control details for a dataset
  publish        list dataset for others to see and pull
  unpublish      unlist dataset, removing it from public visibility
```

Publish used to have a note talking about how it pushes data. This RFC assumes that work is now handled by `qri push`, so now publish only controls `list` visibility:

```
$ qri access publish --help
Publish makes your dataset publically visible to others. While online, peers 
that connect to you can only see datasets and versions that you've published. 
Publishing a dataset always makes all previous history entries available, and 
any updates to a published dataset will be immediately visible. Published 
datasets will show up in feeds.

New datasets are *not* published by default, remaining unlisted until published.

Pushing to the registry requires a dataset be either encrypted or published.
By default pushing a dataset to the registry also publishes.

Usage:
  qri access publish [DATASET] [DATASET...] [flags]

Examples:
  # Publish a dataset so others can see it with "qri list":
  $ qri access publish me/dataset

  # Publish a few datasets:
  $ qri access publish me/dataset me/other_dataset
```

Unbpublish used to be a flag. In this RFC it's bumped up to a subcommand of it's own, to match other reciprocal commands like `push/pull`. 

```
$ qri access unpublish --help
Unpublish reverses the publish process, removing a dataset from public view. 
Unpublishing a dataset will try to "clean up", asking remotes this dataset has
been published to

Unpublished datasets cannot be pushed to the registry, but can be pushed to
peers that accept unpublished datasets from you.

Usage:
  qri access unpublish [DATASET] [DATASET...] [flags]

Examples:
  # unpublish a dataset, removing
  $ qri access unpublish me/dataset

  # Publish a few datasets:
  $ qri access unpublish me/dataset me/other_dataset
 
Flags:
  -r, --remove     issue "remove all" commands to any published remotes
```


Publish & Unpublish on their own may not seem like they warrent being moved into a subcommand. To understand the justification for the `access`, this is how `access` _might_ look one day, after we've landed encryption and role-based access control (RBAC) for datasets:

```
$ qri access --help
Access controls visibility and permissions for datasets.

Usage:
  qri access [command]

Available Commands:
  info           show access control details for a dataset
  publish        list dataset for others to see and pull
  unpublish      unlist dataset, removing it from public visibility
  encrypt        restrict access to a dataset with a secret key
  decrypt        remove access restrictions from a dataset
  allow          give permissions to a peer for an encrypted dataset
  deny           remove peer access permissions
```

### `qri access info`
```
$ qri access info --help
Info describes permissions for a dataset.

Usage
  qri access info [DATASET] [DATASET...]
```

An example output for a published dataset:
```
$ qri access info me/dataset
can-edit:  true
published: true

You can edit this dataset.

This dataset is "published". Other users may see it when you're online, and if
pushed to the registry it will show up in feeds for other users to browse and
pull.
```

An example for a dataset you _don't_ own:
```
$ qri access info ca-state-parks/park-features
can-edit:  false
published: true

You cannot edit this dataset, attempting to save to it's history will create a 
fork.

This dataset is "published". Other users may see it when you're online, and if
pushed to the registry it will show up in feeds for other users to browse and
pull.
```

`$ qri access info me/dataset` for now will only but it will be _very_ helpful for viewing dataset permsissions that control both who can access a dataset and what they can do with it.

### Publish on registry push
Pushing unencrypted datasets† to the registry _requires_ a dataset be published. To keep the old behaviour of a one-liner publish, pushing to the registry will automatically run publish on the user's behalf. The push command will present this as user feedback:

```
$ qri push me/some_dataset
setting dataset access to "published"... done.
pushing 28 versions of chriswhong/some_dataset...
X of XX blocks pushed for version 12 August 2019 (Qmss4h552h3j2h33)...
```

† _encrypted datasets don't exist yet._


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### The `Published` field
For some time we've modelled `published` as a boolean field outside the dataset itself, stored as a logbook operation. We've struggled with this field not capturing the problem of _where_ a dataset's been published. Re-conceiving of `published` as a "permission bit" clarifies the intent of this field. Published now answers "can the existence of this dataset be broadcasted" for the _entire history_ of a dataset. 

### Corner cases
A few extra bullet points to consider while we're implementing this:

* qri list `--published` should work as it does today, showing only local published datasets
* We should create a new set of `lib.AccessMethods`, talking to a number of subsystems for access control management.
* many `publish` tests should move over to the new `push` command

# Drawbacks
[drawbacks]: #drawbacks

Access control is really irritating.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### `listed` and `unlisted`
Instead of the publish/unpublish terminology, we could use a more direct reference to list visibility. We can't use `list` directly because `qri access list` should be the command for listing all access-controlled datasets (if such a thing turns out to be necessary). An alternative would be to use a different conjucation or tense for list like "listed" & "unlisted". using a past tense is the best I could come up with, and in my opinion `$ qri access listed` isn't very memorable, and feels like your asking about things that have been listed in the past?

The other thing to keep in mind: unlisted datasets shouldn't show up in feeds. The registry requires "published: true", but feeds built on the peer-2-peer network won't. Peers building feeds should only use datasets that are published. This clearly aligns

# Prior art
[prior-art]: #prior-art

* Google Docs Sharing settings
* Unlisted Vimeo Videos
* RBAC - Role Based Access Control

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### An "Access" Data Structure
This RFC describes `published` as a _permissions flag_. At some point we should be looking to build a data structure that describes permissions. `Published` might be a field on that data structure. The challenge here: permissions will apply to, what, users? groups? Needs more thought.