- Feature Name: Filesystem Integration
- Start Date: 2019-07-02
- RFC PR:
- Issue:

# Summary
[summary]: #summary

Adds filesystem integration to qri datasets including a current working directory context for some qri commands.  The goal is to provide an experience that is more analogous to the git workflow, where working changes to files are sensed prior to committing a new version.

# Motivation
[motivation]: #motivation

The motivation for this feature is to provide a more tangible connection between the qri versioning/publishing ecosystem and the file-based data that data practitioners are used to working with. Since all of the information in a qri dataset maps to files, we should just make those files exist automatically when a qri dataset is "added" to the user's local machine.  Likewise, we should watch for changes to those files, and ensure that they are all in a valid state before allowing the user to commit changes to a qri dataset.

*This will significantly lower the barrier to entry for new users to qri, especially GUI users, as they can quickly pull down published datasets and begin working with them without having to understand how IPFS/QRI stores and maintains the data.*  

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The intent is to create a workflow more analogous with the git workflow.  When a new user wants the files in a remote git repo, they use `git clone {repoUrl}` and a new directory is created on their filesystem containing those files.  They can use _whatever tooling they want_ to make changes to the files, and commit them locally when they reach a critical point.

We want the qri dataset user to have a similar experience: `qri add /b5/world_bank_population/` should create a new directory called `world_bank_population` on the filesystem, containing the following files:

- `body.csv`
- `meta.json`
- `structure.json`
- `transform.star`

To change the dataset, the user uses _whatever tooling they want_ to modify the files, then commits the changes.

## Commands

Under this proposal, `qri use` is no longer relevant, as qri commands should be run from the qri dataset's directory.

- `qri add` - creates a copy of the dataset's files in the present working directory
- `qri status` - provides a report of the current state of the files in the current directory versus the last commit to the dataset
- `qri save` - commits changes on the current dataset, and should throw errors if the changes to be committed are not valid

## Strict Filenames

This feature allows for qri datasets to live anywhere on the user's filesystem, and be mixed-in with other files and folders.  The initial implementation should require the strict filenames and extensions listed above (with the exception that `body` can have extension csv, json, xlsx, etc)

This means `qri status` and `qri save` should ignore any non-qri files in the dataset directory.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

- Something about the invisible `.qri-ref` file that links the current directory to the central qri store

# Drawbacks
[drawbacks]: #drawbacks

- More overhead around the core qri service
- Permissions/Cross-platform filesystem issues?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is preferable because it makes Qri more relatable to data consumers who are used to working with data files.  It also emulates the established and popular workflow of git.

If we don't do this, qri users have the additional burden of manually exporting qri components to files, and must use more advanced commands/tooling/scripting to commit changes to a dataset. _This proposal opens up the universe of qri-compatible tools to anything that can save json and csv files to the filesystem_

This feature is also critical to the desired GUI for Qri, which would make the Qri frontend behave more like Github Desktop.  The Qri frontend will clone, validate commits, show diffs, allow for pushing and publishing, etc.  Some basic editing will be possible via the frontend, but most users will move to another environment to actually make changes.

# Prior art
[prior-art]: #prior-art

Git is the inspiration for this model, specifically the idea that cloning a repository results in new files on the local machine that can be modified and subsequently committed as a new version.

# Unresolved questions
[unresolved-questions]: #unresolved-questions
