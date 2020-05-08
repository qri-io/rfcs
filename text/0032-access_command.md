- Feature Name: access_command
- Start Date: 2020-05-02
- RFC PR: [#54](https://github.com/qri-io/rfcs/pulls/54)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Decouple dataset visibility from data pushing, moving `publish` & `--unpublish` into a new subcommand: `access`. Position `access` for future commands like access control and encryption. Rename `publish/unpublish` to `visible/hidden`.

# Motivation
[motivation]: #motivation

In the current version of qri (v0.9.8) "listing" a dataset so other peers can see it is tangled up with moving a dataset around. Both actions are tightly coupled in the `publish` command, and can be untangled with flags like `--no-registry`, which prevents publish from pushing data, leaving only the listing behaviour.

Publish is the only way to send datasets somewhere else, and the way publish is set up gives us very little room to grow. `publish` works well as a one-liner for "put this public data on qri.cloud", but falls short for other uses on the near-term roadmap. Some examples:

* send a dataset to a trusted peer without listing it
* send encrypted data to the registry
* ask a peer to _stop_ listing a dataset sent to them for others to see

All are examples of _unlisted collaboration_, where _sending_ and _listing_ data are independent actions. Forcing these use cases through the `publish` command yields strange results (like having to "publish" encrypted data).

To accommodate more use cases, we should flip the mental model, prioritizing thinking about _where data is being sent_. `publish` should join a family commands for managing _access control_. Sending data to a public destination should imply public listing. In other words: publishing should be an automatic part of pushing, instead of pushing being an automatic part of publishing.

The actual process of moving data around is the subject of another RFC on pushing and pulling. In this RFC we tackle how to manage publication status of a dataset.

This RFC proposes four concrete changes:
* create a new `access` subcommand to view & edit access control for users & datasets
* move `publish` and `unpublish` into access, changing both commands to edit _only_ `list` visibility (not push data)
* rename `publish/unpublish` to `visibile/hidden`.
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
  hidden         unlist dataset, removing it from public visibility
  visible        list dataset for others to see and pull
```

The old `publish` used to have a note talking about how it pushes data. This RFC assumes that work is now handled by `qri push`. the new `visible` command only controls listing & feed visibility:

```
$ qri access visible --help
Visible makes your dataset publically visible to others. While online, peers 
that connect to you can only see datasets and versions that you've published. 
Making a datset a dataset always makes all previous history entries available, 
and any updates to a visible dataset will be immediately visible. Published 
datasets will show up in feeds.

New datasets are *not* visible by default, and remain hidden until pushed.

Pushing to the registry requires a dataset be either encrypted or published.
By default pushing a dataset to the registry also makes a dataset visible.

Usage:
  qri access visible [DATASET] [DATASET...] [flags]

Examples:
  # make a dataset visible so others can see it with "qri list":
  $ qri access visible me/dataset

  # make a few datasets visible at once:
  $ qri access visible me/dataset me/other_dataset
```

Unbpublish used to be a flag. In this RFC it's bumped up to a subcommand of its own, to match other reciprocal commands like `push/pull`. Here's it's renamed to `hidden`, and again it only deals with weather a dataset will show up in listing & feeds. the `hidden` command does _not_ move data around.

```
$ qri access hidden --help
hidden "unlists" a dataset. A hidden dataset will not show up when browsing 
dataset lists or feeds. Others can still access a hidden dataset if they know
its name.

hidden datasets cannot be pushed to the registry, but can be pushed to peers 
that accept hidden datasets from you.

Usage:
  qri access hidden [DATASET] [DATASET...] [flags]

Examples:
  # unpublish a dataset, removing it from any lists or feeds
  $ qri access unpublish me/dataset

  # Publish a few datasets:
  $ qri access unpublish me/dataset me/other_dataset
 
Flags:
  -r, --remove     issue "remove all" commands to any published remotes
```


`visible` & `hidden` on their own may not seem like they warrant being moved into a subcommand. To understand the justification for the `access`, this is how `access` _might_ look one day, after we've landed encryption and role-based access control (RBAC) for datasets:

```
$ qri access --help
Access controls visibility and permissions for datasets.

Usage:
  qri access [command]

Available Commands:
  info           show access control details for a dataset
  visible        list dataset for others to see and pull
  hidden         unlist dataset, removing it from public visibility
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
visible:   true

You can edit this dataset.

This dataset is visible. Other users may see it when you're online, and if
pushed to the registry it will show up in feeds for other users to browse and
pull.
```

An example for a dataset you _don't_ own:
```
$ qri access info ca-state-parks/park-features
can-edit:  false
visible:   true

You cannot edit this dataset, attempting to save to its history will create a 
fork.

This dataset is visible. Other users may see it when you're online, and if
pushed to the registry it will show up in feeds for other users to browse and
pull.
```

`$ qri access info me/dataset` for now will only but it will be _very_ helpful for viewing dataset permsissions that control both who can access a dataset and what they can do with it.

### Visible on registry push
Pushing unencrypted datasets† to the registry _requires_ a dataset be visible. To keep the old behaviour of a one-liner publish, pushing to the registry will automatically run `$ qri access visible` on the user's behalf. The push command will present this as user feedback:

```
$ qri push me/some_dataset
setting dataset access to visible... done.
pushing 28 versions of chriswhong/some_dataset...
X of XX blocks pushed for version 12 August 2019 (Qmss4h552h3j2h33)...
```

Remotes, on the other hand _can_ accept hidden datasets. Remote configuration will get a `requireVisible` configuration parameter to get the same registry-like behaviour. Pushing hidden datasets is aimed at the collaborator use-case, where a datasets are explicitly pushed to trusted colleagues. when a user is pushed a hidden dataset, it will show up when they run `qri list` locally, but this dataset won't be broadcast back to the network.

† _encrypted datasets don't exist yet._


### Visible means visible _everywhere_
In the past we've had permissions tied to a dataset/remote combination. As an example I could "publish" to a registry and that registry would have it.

This RFC proposes permissions like visibility _only apply to datasets_. If you `push` a visible dataset to remote A, and peer B downloads that dataset, it's also `visible` for peer B.

Later when we add access control, if you say "this dataset can _only_ be accessed by peer B", that will apply to anyone with the dataset.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### The `Visible` field
For some time we've modelled `published` as a boolean field outside the dataset itself, stored as a logbook operation. We've struggled with this field not capturing the problem of _where_ a dataset's been published. Re-conceiving the new `visible` field as a "permission bit" clarifies the intent of this field. Published now answers "can the existence of this dataset be broadcasted" for the _entire history_ of a dataset. 

### Corner cases
A few extra bullet points to consider while we're implementing this:

* qri list `--published` should be renamed to `--visible`, should work as it does today, showing only local, visible datasets
* We should create a new set of `lib.AccessMethods`, talking to a number of subsystems for access control management.
* many `publish` tests should move over to the new `push` command

# Drawbacks
[drawbacks]: #drawbacks

Access control is really irritating.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### `listed` and `unlisted`
Instead of the publish/unpublish terminology, we could use a more direct reference to list visibility. We can't use `list` directly because `qri access list` should be the command for listing all access-controlled datasets (if such a thing turns out to be necessary). An alternative would be to use a different conjucation or tense for list like "listed" & "unlisted". using a past tense is the best I could come up with, and in my opinion `$ qri access listed` isn't very memorable, and feels like your asking about things that have been listed in the past?

The other thing to keep in mind: hidden datasets shouldn't show up in feeds. The registry requires "published: true", but feeds built on the peer-2-peer network won't. Peers building feeds should only use datasets that are published. 

# Prior art
[prior-art]: #prior-art

* Google Docs Sharing settings
* Unlisted Vimeo Videos
* RBAC - Role Based Access Control

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### An "Access" Data Structure
This RFC describes `visible` as a _permissions flag_. At some point we should be looking to build a data structure that describes permissions. `Visible` might be a field on that data structure. The challenge here: permissions will apply to, what, users? groups? Needs more thought.