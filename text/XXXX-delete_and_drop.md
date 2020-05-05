- Feature Name: delete and drop
- Start Date: 2020-05-02
- RFC PR: [#53](https://github.com/qri-io/rfcs/pulls/53)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Rename the remove command to `delete`, and clarify it's purpose is for deleting versions from a dataset history. Create a new command `drop` for removing datasets from remotes.

# Motivation
[motivation]: #motivation

The _user experience (UX)_ of removing data from Qri needs some work. We're seeing a number of users get into states they feel they cannot recover from, and rather than use tools like `qri remove`, they're deleting repositories entirely in an effort to start over. 

A better UX for removing data from qri should clearly answer two common "uh oh" scenarios a user is likely to encounter:

1. "I made a mistake while saving"
2. "I want to undo a push"

The first is an example of removing data from history. The second is an example of removing data from the network. In both scenarios the user is a data author, and a good answer to these questions should present well-documented, easy-to-find commands that when executed leave the user feeling that they're in a clean state. 

A well-documented desctrive command should be clearly labeled as the "opposite" of a constructive command and present warnings _before_ execution. Once executed, commands remove data should show as few caveat/errors/warnings as possible, or at least connect caveats to a clear next course of action. A feeling of "clean state" comes when vommands like `qri log`, `qri list` and `qri status` match user expectations after running a destructive command.

To improve our "undo UX", this RFC proposes three changes to qri CLI:
1. rename `remove` to `delete`, clarify its text to talk about history instead of just storage
2. Add an `amend` flag on the `save` command that _replaces_ the HEAD commit
3. Add a new top-level command: `drop`, for dropping datasets you don't own


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Renaming Remove
Here's the current (qri v0.9.8) description of the `remove` command:

```
$ qri remove --help
Remove gets rid of a dataset from your qri node. After running remove, qri will
no longer list your dataset as being available locally. 
[rambling bit about IPFS storage redacted]
```

This text is misleading. It seems to be about storage, when the primary purpose of  `remove` is to edit history. Storage savings are a byproduct of editing one or more commits out of a history. This RFC sets aside the tight coupling between history and storage for another day, and instead focueses on fixing the role of remove by clearifying it's documentation:

```
$ qri delete --help
Delete removes commits from a dataset history. Use delete to "correct the record", 
removing commits in a dataset history. Delete always works from the latest (HEAD) 
commit, working backwards toward the first commit. To delete from a dataset you 
must be able to edit it.

Delete requires a "revisions" flag, specifying

To delete an entire dataset and free up the name again, use `--all` (or a 
number of revisions longer than the history itself). Before doing this be sure
to drop your dataset from other remotes

Usage:
  qri delete [DATASET] [--revisions|--all] [flags]

Examples:
  # delete the latest commit from annual_pop
  $ qri delete me/annual_pop --revisions 1

  # delete the latest two versions from history, keeping the underlying data
  $ qri delete me/annual_pop --revisions 2  --keep-storage

  # destroy a dataset named `annual_pop`
  $ qri delete --all me/annual_pop

Flags:
   --keep-storage     don't remove the deleted versions from storage

```

"Delete" sounds more destructive than "remove", and that's a good thing in this case, becuase delete is editing history. The delete command is also how you _completely remove_ a dataset (with the `--all` flag). The simple change of having a delete command exist should help users find a way to edit history.

### Amend

Running `save` with the `--amend` flag replaces the current HEAD with a new commit. This is pulled directly from git. 

Adding an `--amend` flag has the UX advantage of working language that resembles "undo" directly into the save command, which users will be familiar with before they ever need `delete`. As a git user I (b5) learned to use `--amend` before `rebase`, which I admit wasn't great, but allowed me to stick with git long enough to understand what rebase did.

Users should come to think of `--amend` as a shortcut:

```
# two commands:
$ qri delete me/dataset --revisions 1
$ qri save me/dataset

# same thing, one command:
$ qri save me/dataset --amend 
```

When a user understands these two things to be equal, it'll help solidfy their mental model of versioning with Qri.


### the drop command
Drop removes other people's datasets. By default drop operates on your local node, removing a dataset that you've pulled. Drop can be paired with the `--remote` flag to ask remotes to drop a dataset.

The `drop` flag answers two common scenarios:
* "I was using this dataset and don't need it anymore": `$ qri drop user/dataset`
* "I made a dataset, and want it off qri.cloud": `$ qri drop me/dataset --remote registry`

Drop joins `push` and `pull` as a family of active-verb commands that describe a "me and them" interaction pattern.

```
$ qri drop --help
Drop removes datasets you've pulled. After running drop, a dataset will no
longer show up in your local list, and all stored versions of that dataset will
be deleted.

Use the --remote flag to asks a remote to completely remove a dataset, 
deleting history and stored versions. Drop with the remote flag reverses a push.

Drop only works on datasets you *don't* own. Use the delete command to remove
datasets you've created.

Drop completely removes a dataset, which may not be what you'd like do do. 
If you want to...
* delete a dataset you own:
  $ qri delete me/dataset --all
* remove a version from a remote, for a dataset you own:
  $ qri delete me/dataset --revisions 1
  $ qri push [remote] me/dataset
* drop a specific version from a remote:
  $ qri storage drop me/dataset@[version] --remote [remote]
* unlist a dataset on a remote:
  $ qri access unpublish me/dataset
  $ qri push [remote] me/dataset

Usage:
  qri drop DATASET [DATASET...] [flags]

Examples:
  # Drop a dataset you've pulled
  $ qri drop user/dataset

  # Drop a few datasets at once
  $ qri drop user/dataset other_user/other_dataset other_user/another_dataset

  # Remove a dataset from the registry
  $ qri drop me/annual_pop --remote registry

  # Remove a dataset from a remote you've pushed to named "sally"
  $ qri drop me/annual_pop --remote sally

Flags:
  -h, --help               help for drop
      --remote             send a drop request to a remote
```

`drop` and `delete` have different audiences. `drop` is for data _consumers_, removing datasets they've pulled. `delete` is for data _authors_, correcting data they're pushing. Attempting to drop a dataset you own will error:

```
$ qri drop me/dataset
error: cannot drop a dataset you own

because you are the author of this dataset, dropping it would remove the 
authoritative history of this dataset. Instead of dropping, you can 
delete this dataset:

  $ qri delete --all me/dataset

```

Drop should do the opposite of a `pull` operation, removing from logbook, caches, and stroed versions. This solves a business-logic problem: anything we do "automatically" for a user on pull needs to be undone when the user wants to go the other way. Having an explicit "opposite of pull" command allows us to write the converse business logic to whatever pull does automatically, restoring the user to a clean state without having to resort to plumbing commands.

From the UX side, `drop` is also a chance to point users to other, more nuanced delete commands, and present a _big_ warning before they execute drop. These two together should help move users toward other commands that may match thier intent. Drop intentionally doesn't work for data  _authors_, forcing them to use `delete`, which clarifies they're making edits that will affect _everyone_ that relies on this dataset.

Knowing `drop` exists should make data authors more comfortable with `push`. A safe feeling that a user can `push, drop, push, drop...` should help users feel safe knowing their actions can be undone. We _do_ need to pay careful attention to language here, letting users know that once something's pushed, we can't control who's _already pulled it_.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Use the logbook
Delete actions should examine the history of a logbook to present warnings to a user that help them "clean up" when deleteing stuff. As an example: `$ qri delete --all` on a dataset should look for `push` operations and warn user they should `drop` from a list of remotes.

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
Another RFC currently has `qri push` to the registry automatically adding a "publish" step to the dataset, the drop command as presented here _doesn't_ undo this, leaving p2p visibility in list. Maybe we should present the user with a warning?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### `--delete` flags

Git uses `--delete` as a flag on push to get rid of remote branches:
```
$ git push origin --delete feature/login
```

We do something simliar now with `$ qri publish --unpublish ` now, and it's a viable alternative to having dedicated commands. Looking at the registry, users seem to be finding & using the `--unpublish` flag, which is heartening. My concern is qri wants to deviate from git by giving users fine-grained control over version storage. Layering deletes atop push operations leaves little room for these detailed needs.

### Drop only does one thing
It's a good word. Maybe drop should be doing more?

# Prior art
[prior-art]: #prior-art

Heroku's `rollback` command plays an important UX inspiration for `drop`. Having a top level "go back" command is part of what makes working with heroku feel safe.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Storage management
This RFC puts storage management out of scope.

### logbook garbage collection
Running `qri delete --all` drops a bunch a storage and "caps off" a dataset history with a delete operation, preventing the dataset from showing up in list operations. Delete doesn't remove remove logbook data. We'll have to figure out a way to clean up the logbook itself with a plubming command of some sort.

### undeleting
Speaking of "capped off" logs, it's also possible to "undelete" by re-opening the log, writing a new operation to the closed log. We haven't addressed if this sort of double-undo should be allowed. If so, it shouldn't be a top-level command.