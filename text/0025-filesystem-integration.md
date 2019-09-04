- Feature Name: Filesystem Integration
- Start Date: 2019-07-02
- RFC PR: [#45](https://github.com/qri-io/rfcs/pull/45)
- Issue:

# Summary
[summary]: #summary

This feature adds a "live link" between a qri dataset in the user's IPFS repo and a directory on the filesystem. It adds new qri commands for initializing this link, and changes the behavior of some commands to use the files in the working directory.

# Motivation
[motivation]: #motivation

The motivation for this feature is to provide a more tangible connection between the qri versioning/publishing ecosystem and the file-based data that data practitioners are used to working with. Since all of the user-modifiable information in a qri dataset can map to files, we should just make those files exist on the filesystem.  Likewise, we should watch for changes to those files, and ensure that they are all in a valid state before allowing the user to commit changes to a qri dataset.

*This will significantly lower the barrier to entry for new users to qri, especially GUI users, as they can quickly pull down published datasets and begin working with them without having to understand how IPFS/QRI stores and maintains the data.*  

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The intent is to make a qri dataset more tangible (and thus easier to get started with) by adding a common interface (the user's filesystem).  

When a new user wants the files in a remote git repo, they use `git clone {repoUrl}` and a new directory is created on their filesystem containing those files.  They can use _whatever tooling they want_ to make changes to the files, and commit them locally when they reach a critical point.

We want the qri dataset user to have a similar experience: `qri add /b5/world_bank_population/` should create a new directory called `world_bank_population` on the filesystem containing the data file and associated metadata. To change dataset, the user uses _whatever tooling they want_ to modify the files, then commits the changes.

## One-to-One

This is a 1-1 relationship, a dataset can only link to one directory on the filesystem.

## Files

The following files abstract the qri dataset model, providing well-named files for the most recognizable parts of the dataset.

### `meta.json`

The dataset's `meta` component.

### `schema.json`

The dataset's `structure.schema` component

### `body.[csv|json|xlsx]`

The data file!

### `dataset.json`

Any additional uncontrolled `dataset` fields.  This file will probably not be used by novice users.  Users who are more familiar with the dataset model can use this file to add advanced configuration.

### `.qri-ref`

An invisible file used to specify the link between the current directory and the Qri repo.

## Commands

- `qri init` - interactive prompt that sets up a qri dataset with linked files in the current directory. (User should have already created a directory to hold the dataset, `cd` into it, then run `qri init`)

This should have an `non-interactive` flag that only needs a dataset name, which will link an empty dataset working directory to a name reference in the qri repo. (reserving the name for a future commit)

### Prompts:
  -  **Dataset Name** - (autopopulate with the name of the current directory) generates `meta.json` in the working directory with the supplied name as `title` and boilerplate `description`, `keywords`, etc

  -  **File Type** - csv/json? Based on the user's selection, add dummy `body.csv|json` and `schema.json` to the working directory.

- `qri clone` - hybrid command that adds an existing dataset to the user's local Qri repo _AND_ creates a linked working directory on the filesystem.  

- `qri status` - provides a report of the current state of the files in the working directory versus the last commit to the dataset.  Validates changes and gives a green light if everything is ready to be committed.

  - `.qri-ref`
    - If this file doesn't exist, notify the user that they are not in a qri dataset directory
    - If it does, provide info about the dataset and version this directory is linked to

  - `body.csv|json`
    - No changes/XX changes
    - valid schema/XX schema violations
    - error if not parsable CSV/JSON etc

  - `meta.json`
    - No changes/XX changes
    - warning if missing suggested minimum metadata fields
    - error if invalid

  - `schema.json`
    - No changes/XX changes
    - error if invalid JSON schema

  - `dataset.json`
    - No changes/XX changes
    - error if invalid


- `qri log` - show the log of commits for the current dataset

- `qri checkout {hash}` - change the files in the working directory to reflect a previous version of the qri dataset

- `qri commit|save` - commits changes on the current dataset, ensures that the working directory reflects the version of the dataset after the commit (e.g. if there was no `schema.json` because the user just initialized an empty working directory)

Notes:
- `qri add` - May cause confusion because it does not work the same way as `git add` as qri has no staging area.  For now it should remain unchanged, `qri add` will add the dataset to the user's local IPFS repo but will not create a linked directory on the filesystem.

- `qri use` can still work, but if the user is currently in a linked qri dataset directory, all commands should assume the user is working with the linked dataset

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When a filesystem folder is linked to a qri dataset, we create a file `.qri-ref` in that directory. This file contains a full path to the qri dataset (peername, dataset name, profile id, network, version hash). Every command that normally allows for `use` refs, will check for the presence of this file, and if found, will use that dataset reference contained within to pass along to the command's arguments.

Running `diff` would compare the working directory against the head version of the dataset, using the existing diff lib functionality.

Running `status` would compare the same, but only as a binary yes/no "are they the same?" comparison, to display if the working directory is dirty or clean.

Running `save` would save using the dataset.yaml and body.csv in the working directory as file parameters.

# Drawbacks
[drawbacks]: #drawbacks

- More overhead around the core qri service
- Permissions/Cross-platform filesystem issues?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is preferable because it makes Qri more relatable to data consumers who are used to working with data files.  It also emulates the established and popular workflow of git.

If we don't do this, new qri users have the additional burden of manually exporting qri components to files, and must use more advanced commands/tooling/scripting to commit changes to a dataset. _This proposal opens up the universe of qri-compatible tools to anything that can save json and csv files to the filesystem_

This feature is also critical to the desired GUI for Qri, which would make the Qri frontend behave more like Github Desktop.  The Qri frontend will clone, validate commits, show diffs, allow for pushing and publishing, etc.  Some basic editing will be possible via the frontend, but most users will move to another environment to actually make changes.

# Prior art
[prior-art]: #prior-art

Git is the inspiration for this model, specifically the idea that cloning a repository results in new files on the local machine that can be modified and subsequently committed as a new version.

# Unresolved questions
[unresolved-questions]: #unresolved-questions
