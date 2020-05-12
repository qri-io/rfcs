- Feature Name: expanded remove command
- Start Date: 2020-05-02
- RFC PR: [#53](https://github.com/qri-io/rfcs/pulls/53)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Clarify how `remove` relates to history & storage manipulation, "owned" vs. "cloned" datasets. Expand `remove` to also work on remotes when used with the `--remote` flag. Add an `--amend` flag to `save` for easier single-commit revisions. Introduce _read-only_ and _read-write_ terms to describe Qri's _de facto_ access control model.

# Motivation
[motivation]: #motivation

The _user experience (UX)_ of removing data from Qri needs some work. We're seeing a number of users get into states they feel they cannot recover from, and rather than use tools like `qri remove`, they're deleting repositories entirely in an effort to start over. 

Part of the The `remove` command doesn't presently cover all the use cases we need it to. In v0.9.8, a user `b5` with a dataset `chriswhong/data_portal_snapshot` gets this from the `remove` command:

```
$ qri remove chriswhong/data_portal_snapshot
need --all or --revisions to specify how many revisions to remove
```

Running with the `--all` flag work, but `b5` has no edit access to the dataset, the `--all` flag is misleading & implies to the user they are _able_ to edit history, when they're not.

A better UX for removing data from qri should _clearly_ answer the common "uh oh" scenarios a user is likely to encounter:

1. "I made a mistake while saving"
2. "I want to delete a dataset I've pulled"
3. "I want to undo a push"

Theses are three distinct examples of removing versions, whole datasets, and remote replicas. This RFC proposes expanding the `remove` command to properly cover all of these common scenarios with the following changes:

1. re-work the remove command to deal with all common cases of data removal. Formalize the interactions between these three removal scenarios 
2. introduce _read-only_ and _read-write_ as terms to describe our present dataset acces model.
2. improve help text and error messages, guiding users toward different uses of remove.
3. Add an `amend` flag on the `save` command that _replaces_ the HEAD commit
4. Add a `--remote` flag to `delete` for dropping datasets on remotes

A well-documented desctrive command should be clearly labeled as the "opposite" of a constructive command and present warnings _before_ execution. Once executed, commands remove data should show as few caveat/errors/warnings as possible, or at least connect caveats to a clear next course of action. A feeling of "clean state" comes when vommands like `qri log`, `qri list` and `qri status` match user expectations after running a destructive command.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


### "read-only" and "read-write"

To get through this RFC we need language to distinguish between datasets the current user can and cannot edit, because it has an impact on the default behaviour of removing a dataset. **These terms should describe how qri currently works, not introduce new functionality**

This RFC proposes distinguishing between _read-write_ and _read-only_ datasets like so:

* datasets you create are _read-write_
* all other users datasets are _read-only_
* attempting to save to a read-only dataset creates a fork

This proposes _qri already has an access control model_, we just need to make it explicit. Talking about our present system This gives us room for growth without introducing new terms. In the future what makes a dataset "readable" or "writable" will become more nuanced as we introduce collaboration features.

### Expanding the Remove command

This RFC sets aside the tight coupling between history and storage for another day, and instead focuses on fixing the role of remove by clarifying and expanding it's purpose.

Here's the proposed help text for the new remove command:

```
$ qri remove --help
Remove deletes datasets from qri.

For read-only datasets you've pulled from others, Remove gets rid of a dataset 
from your qri node. After running remove, qri will no longer list your dataset 
as being available locally, and will free up storage space.

For datasets you can edit, remove deletes commits from a dataset history.
Use delete to "correct the record" by erasing commits. Running remove on writable
datasets requires a "revisions" flag, specifying the number of commits to 
delete. Remove always starts from the latest (HEAD) commit, working backwards
toward the first commit.

Remove can also be used to ask remotes to delete datasets with the `--remote`
flag. Passing the remote flag will run the operation as a network request,
reporting the results of attempting to remove on the destination remote.
The remote flag can only be used to completely remove a dataset from a remote.
To edit history on a remote, run delete locally and use `qri push` to send the
updated history to the remote. Any command run with the remote flag has no
effects on local data.

Usage:
  qri remove DATASET [DATASET...] [flags]

Examples:
  # delete a dataset
  $ qri remove user/world_bank_population

  # delete the latest commit from annual_pop
  $ qri remove me/annual_pop --revisions 1

  # delete the latest two versions from history
  $ qri remove me/annual_pop --revisions 2

  # destroy a dataset named `annual_pop`
  $ qri remove --all me/annual_pop

  # ask the registry to delete a dataset
  $ qri remove --remote registry me/annual_pop


Flags:
   --keep-files       don't remove a working directory if one exists
   --force            override checks that prevent data loss
```

### Amend

Running `save` with the `--amend` flag replaces the current HEAD with a new commit. This is pulled directly from git. 

Adding an `--amend` flag has the UX advantage of working language that resembles "undo" directly into the save command, which users will be familiar with before they ever need `delete`. As a git user I (b5) learned to use `--amend` before `rebase`, which I admit wasn't great, but allowed me to stick with git long enough to understand what rebase did.

Users should come to think of `--amend` as a shortcut:

```
# two commands:
$ qri remove me/dataset --revisions 1
$ qri save me/dataset

# same thing, one command:
$ qri save me/dataset --amend 
```

When a user understands these two things to be equal, it'll help solidfy their mental model of versioning with Qri.



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The remove command unpins blocks AND alters the logbook. Removing a read-only dataset drops logbook data entirely. Removing from a writable dataset adds an operation to the dataset log marking the number of versions removed.

All the previously stated local delete behaviors stay the same, with a few exceptions listed below.
But in addition to those behaviors, we have remote behaviors as well:
```
$ qri remove me/dataset --remote registry
```
Unpins all blocks & removes the log on the registry because the registry has a read-only copy. Does nothing locally.

Attempting to drop revisions from a read-only dataset replica is an error:
```
$ qri remove me/dataset --remote registry --revisions 1
ERROR
```

This you can't manipulate the logbook of a dataset on a remote. Instead you would first edit locally & push:
```
$ qri remove me/dataset --revisions 1
$ qri push me/dataset
```

remove works without any flags on read-only datasets:
```
$ qri remove not_me/dataset
```
unpins the blocks and then removes the log of a dataset that is not in your namespace. Because we have to remove the log entirely, we cannot restore a dataset that was removed but not in your namespace. Instead restore by re-pulling.

using revisions with a read-only dataset will error
```
$ qri remove not_me/dataset --revisions 1
ERROR
```
should error: you can't manipulate the logbook of a read-only dataset.

If you alter your logbook locally using qri remove, you can update the remote log by using qri push.


### Use the logbook
Delete actions should examine the history of a logbook to present warnings to a user that help them "clean up" when deleteing stuff. As an example: `$ qri remove --all` on a dataset should look for `push` operations and warn user they should `drop` from a list of remotes.

### rename the `--drop` flag on `qri save`
The current master branch has a `--drop` flag on the save command for "dropping components of a dataset". This RFC repurposes the `--drop` language to be about datasets you don't own, so this flag should be renamed to `--remove`. We shouldn't use `--delete`, because the word delete is now reserved for editing history.

### Corner Cases
There are a few scenarios we should make a point writing tests for:
* Running amend on a dataset with no commits (no history) is an error
* a drop request for a dataset name that doesn't exist locally, but _does_ exist on the remote should not error. This implies remotes should prefer local reference resolution
* Remotes _should_ accept altered InitIDs when an owner Pushes to them. If user creates `b5/population`, pushes, then `delete --all b5/population` locally, then creates a new dataset with the same `b5/population` name, and pushes again, that should work, completely replacing the history

# Drawbacks
[drawbacks]: #drawbacks

### Drop + `--remote` doesn't undo publication status
Another RFC currently has `qri push` to the registry automatically adding a "visible" step to the dataset, the drop command as presented here _doesn't_ undo this, leaving p2p visibility in list. Maybe we should present the user with a warning?

### Situations where a dataset you want to edit can't fit locally
The design of the `remove` command requires edits to logbook happen locally. What happens if you want to work on a dataset that you have write access to, but it won't fit on your local machine? It's possible the user could end up in a situation where they can't manipulate a dataset history because they can't get it onto their machine.

The short term solution will have to be :shrug: get a bigger computer. Long term, there's no reason a user needs to have blocks to run a `remove` operation. Saving a new version is an entirely different story. One that may need to be answered by streaming the previous version instead of retain-and-read.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### `--delete` flags

Git uses `--delete` as a flag on push to get rid of remote branches:
```
$ git push origin --delete feature/login
```

We do something similar now with `$ qri publish --unpublish ` now, and it's a viable alternative to having dedicated commands. Looking at the registry, users seem to be finding & using the `--unpublish` flag, which is heartening. My concern is qri wants to deviate from git by giving users fine-grained control over version storage. Layering deletes atop push operations leaves little room for these detailed needs.


# Prior art
[prior-art]: #prior-art

Heroku's `rollback` command plays an important UX inspiration for `drop`. Having a top level "go back" command is part of what makes working with heroku feel safe.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Storage management
This RFC puts storage management out of scope.


### logbook garbage collection
Running `qri remove --all` drops a bunch a storage and "caps off" a dataset history with a delete operation, preventing the dataset from showing up in list operations. Delete doesn't remove remove logbook data. We'll have to figure out a way to clean up the logbook itself with a plubming command of some sort.

### A Restore Command
It's also possible to "undelete" by re-opening the log, writing a new operation to the closed log. We haven't addressed if this sort of double-undo should be allowed. If so, it shouldn't be a top-level command. 

One possible solution is to repurpose qri's FSI `restore` command to a delete/drop undo command. The previous use of qri restore should be contained in `qri checkout`, if the FSI link already exists. Possible commands:

```
$ qri restore me/dataset
```
Restores a dataset based on the logbook, if those blocks still exist

```
$ qri restore me/dataset@version
```
Restores a dataset version if those blocks still exist.