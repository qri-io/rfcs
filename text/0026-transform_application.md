# Summary

Reassessing and understanding how save, dry-run, transforms, and FSI work together.

# Motivation

## Save, Dryrun, and Stdout

Recently, there's been a lot of attention being paid to our save codepath, and some work has begun to dig into it and make it more tractable.

A few days ago, while cleaning up some testing code, I found that when `qri save` writes its message about "dataset saved: [username]/[ds_name]@/ipfs/[full_path]" that message is written to stderr instead of stdout. This seemed to be confusing; we also have purely informational messages like FSI's "for linked dataset..." going to stderr since they're mostly incidental in meaning. The `save` command telling the user what the full reference is seems much more central to its whole purpose.

Looking into why this was happening, I believe the decision was made before FSI existed, and was meant to support transform dry-runs. A command such as this:

```
  qri save --dry-run --file my_transform.star me/my_ds
```

would output the "dataset saved" message to stderr, and the result of the dry-run, which is the dataset's body, to stdout. Then enables piping the body to another program, or saving it to a local file with redirection.

Looking back on this, I don't think this was the best decision. We're affecting the core operation of save and the way it outputs information in order to support a tricky-to-use secondary feature. Furthermore, it's worth asking if this is the correct way to think about developing and introspecting transforms. For the dry-run case, we're confusing `save` which creates commits, with observing and processing the result of a dry-run, which is a read-only operation. The thing we actually care in this case is executing the transform and seeing it's output, not performing a `save`.

At the same time, we have never had a full explanation for how transforms and FSI are supposed to work together, and there's lots of bugs now in how dry-run and transform interact with FSI. We're at a point where we need to re-evaluate how these pieces work together.

Let's go back to the very beginning.

## Manual Save

In the early days, it was decided that the right way to think about a `save` operation on a dataset that already existed was as a "patch application". Let's say I have a dataset named "dustomp/my_ds" that looks like this:

![an example dataset](https://github.com/qri-io/rfcs/blob/rfc-xform/img/dataset_01.png)

Then I run this command to create a second version:

```
  qri save --file new_meta.json me/my_ds
```

What this means is "take my previous version, and *patch it* by updating the meta component with the contents of this file". Everything else in the dataset stays the same. In this way, a manual save operation acts as a state transition between versions by way of patching the previous to arrive at the next.

![diagram of a patch application](https://github.com/qri-io/rfcs/blob/rfc-xform/img/dataset_02_patch_application.png)

## Transforms

When transform scripts were introduced, we needed a similar way to describe the semantics of how they operate. A major question early on was whether a transform living with a dataset should always be reapplied on each save operation, or if it should only be used once, on the commit at which it was added.

We went back and forth on this question, and ultimately arrived at the following functionality: a transform should only be executed on the commit where it gets added, and not every commit in the future, because the resulting dataset may not make sense to use as input to the same transform (think about a transform that adds or removes a column). However, if a user wanted to reapply the same transform, they could opt-in to that functionality by using the `--recall` flag, which would retrieve a recently rerun transform to run again.

This transform changes the shape of the data, so it doesn't make sense to apply it again:
![transform that changes shape](https://github.com/qri-io/rfcs/blob/rfc-xform/img/dataset_03_xform_diff_shape.png)

This transform keeps the same shape of the data; running it a second time makes sense:
![transform that maintains shape](https://github.com/qri-io/rfcs/blob/rfc-xform/img/dataset_04_xform_same_shape.png)

Similar to a manual save, a transform script represents a state transition, with a reproducable method to execute that state transition, and a mechanism called "recall" to clone that state transition when applicable.

A crucial fact about this description is that a manual save and a transform cannot both happen at the same time. They are both state transitions, and since we define a linear history of commits, it's illegal to have two transitions at once. Neither forking nor merging are allowed:

![conflict due to two state transitions](https://github.com/qri-io/rfcs/blob/rfc-xform/img/dataset_05_conflict.png)

What this means in practice is that a save operation cannot have both a transform script and a manual component change, via patch application. These can't safely compose, so we flat out reject them.

```
  qri save --file my_transform.star --file new_meta.json me/my_ds
```

## File System Integration

Introduced next was filesystem integration, which would allow a user to "checkout" a dataset. This creates a "working directory" on their local filesystem which has a link file (.qri-ref) that connects a number of component files (such as meta.json, body.csv) to the dataset that exists within the qri repository. When the user runs `qri save` to create a new version, the files in the working directory are converted into something with the same shape as what's in our repository.

The way this works is that we think of the working directory as being a staging area for a future commit. Since the checked out dataset is not immutable data like that in our repository store, its files are able to be modified. Not until the user issues `qri save` is the working directory turned into a commit; until then it is only a potential future state to transition to:

![checkout to working directory](https://github.com/qri-io/rfcs/blob/rfc-xform/img/dataset_06_working_dir.png)

Furthermore, the type of "patch application" mentioned in the section about "Manual Save" no longer applies. Instead, we need to think about the Working Directory as representing a "full replacement" when a new version is created. So the codepath for `save` knows about both patch applications and full replacements when creating a new commit.

## The Missing Piece

Here are the pieces in place now:

1) A manual save, created by providing file components, adds a commit to a dataset in our repository
2) A transform, which when executed will act as a state transition to a new commit. Cannot be composed with (1)
3) FSI's full replacement save, which converts the contents of the working directory to a new commit

FSI was designed with a lot of thought put into how it would work in relation to manual saves. However, the combination of transforms within FSI has not been clearly defined. We worked under the assumption that transform authors would use them only with non-checked out datasets. But as time has passed, we've found this no longer to be the case.

One major issue is the assumption that transforms make that they cannot be composed with a manual save operation, so no component files can be provided when the transform executes. But under FSI, since we're no longer performing patch operations, the full set of component files are always there in the working directory. Transforms don't know how to handle this, which frequently leads to confusing error messages and irreconcilable states.

Getting down to it, there's a fundamental mismatch with how transforms are defined, and how FSI assumes it may operate. Before FSI, there was a one-to-one correlation between running a transform and creating a commit, hence why it made sense to tie that execution to the "save" command. The `--dry-run` flag was acting as a development tool. But with FSI, that original assumption no longer applies. A transform should be able to be executed as a way to modify the body without commiting, in fact this is probably going to be the expected behavior for any new user coming to qri.

# Proposal

Transforms should not be thought of as equivalent to a state transition to a new commit. They should be viewed literally as a *transform*, from some input to some output, and only sometimes is that performed as creating a commit.

Let's say we add a command `qri apply`, which runs a transform to arrive at some output; but how that output is applies depends upon the context of the command.

1) When operated in the context of a manual save, `apply` can write to stdout, replacing the need for `--dry-run`. In this case, we know that no commit is written, so we can change `save` to write to stdout in other cases.
2) A transform as a state transition can be thought of as a `qri apply` operation piped into the `qri save` command, with appropriate syntacic sugar.
3) For FSI, `qri apply` can modify the body in the working directory, letting the user see the results without messing with the `save` and `--dry-run` comamnds. When they eventually do run `qri save`, we should stash the current body file, apply the transform and verify one of the following:
  * The stashed body matches the output of the transform
  * The stashed body is unchanged from the previous commit, replace it with the transforms' output
  * Otherwise throw an error because the user may be making a mistake
4) Not mentioned above, but once Desktop has better support for transform, we probably will eventually want some interactive development environment. Perhaps transforms can be incrementally compiled and applied, resulting in fast turnaround.

To expand on this idea, applying a transform could be generalized out to other tasks that execute some sort of data pipeline. For example, if we bring back viz, generating those visuals could also be seen as a type of apply operation, with the script being some sort of imaging library, and the output being static images. Or, our `qri update` feature could be somehow incorporated into this. Any sort of task that computes using a dataset to produce some view or artifact could be folded into this command. Better to have a generalized tool than create a one-off command that solves only a narrow problem.

More details of the implementation should wait until the feature is further discussed and some rough concensus can be reached.
