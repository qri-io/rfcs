- Feature Name: publish and update
- Start Date: 2018-10-24
- RFC PR: [#27](https://github.com/qri-io/rfcs/pull/27)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Add "publish" and "update" as patterns for broadcasting when a dataset is ready for public consumption, and an update pattern for grabbing the latest available version of a dataset.

# Motivation
[motivation]: #motivation

Qri makes promises to keep user's data fresh, but we're missing explicit tools & terminology for getting latest versions. This RFC introduces two initial tools for publishing datasets and keeping versions fresh. It's an important part of our story that I'd like to ship some code against to round out our initial user experience.

In regard to publishing, right now Qri is a little _too_ "public". Any time I add anything to Qri and go online, I have no tools for saying "I don't want people downloading this dataset", or "this dataset isn't ready for public consumption". This is creating "use paralysis", where I don't feel as comfortable as I should "practicing" on Qri, because any and everything I save is visible to the world. If I make a mistake, I have no clear way of telling another user "hey don't grab these updates!".

Speaking of updates, I don't have any clear mechanism for re-fetching datasets that's been updated since I "added" it. We need a quick and easy way to syncronize my remote copy of a dataset with the lastest version after it's been published.

Finally, much of the beauty of Qri's self-contained datasets is _not needing to understand code to make an update_. By providing users with an explicit "update" flow that re-executes transforms, that beauty is much easier to see.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC introduces two new concepts to Qri that will come with new commands and, will modify some existing behaviors. Publish and Update will work together to allow users to explicitly specify which datasets are ready for consumption.

## Publish
All datasets start as "unlisted", and the user must `publish` them to make visible to other users. Peers calling `list_datasets` now only see datasets you've designated as published. Any new commits to a published dataset will be available to others.

Whenever I choose, I may "unpublish" a dataset, which will remove it entirely from the list returned by `list_datasets`. It is true that others may still have a copy of my dataset. At present there's not much I can do about it.

#### Publish will integrate with Registries
Running `publish` with a configured registry should POST a dataset _summary_ to the registry. We haven't defined dataset previews, summaries, and documents yet (a story for another RFC), but once it's done publish will both list the dataset for p2p discovery and push the dataset to the registry.

#### There is only "publish" and "unlisted"
Currently there's no way to have one version of a dataset be public and another version be unlisted. Datasets are either completely unpublished or unlisted.

In the future we **might** build the notion of _branches_ into Qri. Branches were popularized by git. In this case branches would allow you to have a different version of a dataset published form the one you're working with locally. If we go this way, publishing will become an explicit step the user has to execute whenever they'd like to advance the current published version of a dataset. If we go this route, I'd like to make the common use patterns as simple as possible, to the point that `qri save` should come with a `publish` flag that combines saving an update & publishing it into one step.


## Update
Update brings your data to the latest known version of a dataset. Update comes in two flavors:
* Updating Someone else's dataset
* Updating A Dataset in your namespace
Both types of update will be accessed via the same command like and API interfaces, they only differ based on the dataset reference in question.

#### Updating a peer's dataset
Running update on a peer dataset checks to see if the peer is online and calls `get_dataset`, if the version is different, it'll grab that dataset. If the peer in question is offline we can't check for updates. Once `publish` also writes to registries, we can check the registry for updates to help aleviate the avilability problem.

Later on we'll want to do proactive checks for updates by submitting a list of dataset refs for which we'd like to grab updates to peers, which will let us display things like "your dataset is out of date, update?" in user interfaces.

#### Updating a dataset in your namespace
Running update on a dataset in your namespace will check if a transform script is specified, If so, it'll re-execute the transform & check for changes. If changes exist it'll create a commit.

Manual changes to a dataset are still run with `qri save`. Calling update on a dataset with no transform script is an error. That error should tell users to run `qri save`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

publish and update are both steps that require an existing dataset.

## Publish:
#### CLI:
```
$ qri publish --help

Description:
  Qri publish makes your dataset available to others. While online, peers that 
  connect to you can only see datasets and versions that you've published. 
  publishing a dataset always makes all previous history entries available.

  By default publish also publishes your dataset to all configured registries,
  to publish to only specific registries, use the --registries flag.

Examples:
  # publish the latest version of a dataset
  $ qri publish me/dataset

  # publish a dataset (and all previous versions) at a specific version
  $ qri publish me/dataset@/ipfs/QmFoo...

  # unpublish a dataset
  $ qri publish -d me/dataset

Flags:
  --delete -d       unpublish the specified dataset
  --no-registry -r  don't publish to the registry
  --registries      comma separated list of registries to publish to
```

#### API:
```
# publish a reference:
POST http://localhost:2505/publish/me/dataset

# publish a dataset (and all previous versions) at a specific version
POST http://localhost:2505/publish/me/dataset/at/ipfs/QmFoo...

# unpublish a dataset
DELETE http://localhost:2505/publish/me/dataset
```

## Update:
#### CLI:
```
$ qri update --help

Description:
  Update fast-forwards your dataset to the latest known version. If the dataset
  is not in your namespace (i.e. dataset name doesn't start with your peername), 
  update will ask the peer for any new versions and download them.

  Calling update on a dataset in your namespace will advance your dataset by 
  re-running any specified transform script, creating a new version of your 
  dataset in the process. If your dataset doesn't have a transform script, 
  update will error.

Examples:
  # get the freshest version of a dataset from a peer
  qri update other_person/dataset

  # update your local dataset by re-running the dataset transform
  qri update me/dataset_with_transform

  # supply secrets to an update, publish on successful run
  qri update me/dataset_with_transform -p --secrets=keyboard,cat

Flags:
  --publish, -p     publish successful updates
  --title, -t       supply a 
  --message, -m     supply a commit message for local updates
  --secrets, -s     specify transform secrets 
```

### API:
```
# get the freshest version of a dataset from a peer
POST http://localhost:2503/update/other_person/dataset

# update your local dataset by re-running the dataset transform
POST http://localhost:2503/update/me/dataset_with_transform

# supply secrets to an update via request headers, publish on successful run
$curl -X "POST" -H "secrets: keyboard,cat" http://localhost:2503/update/me/dataset_with_transform?publish=true
```

## Library changes
### lib:
two new methods:
* `lib.Publish`
* `lib.Update`
I'll work out how they work while coding.

ListDatasets will have to be adjusted to read from either `repo.PublishedRefs` or `repo.Refs` depending on who's asking.

### p2p:
* `list_datasets` should now return the list of _published_ datasets.
* `get_dataset` should only return the latest published version

### repo:
DatasetRef will now have a `Published` boolean field that indicates weather a dataset is published or unlisted.

### deletes:
removing a dataset that has been published should _not_ remove it from any registries, but should remove from local publication list. Deleting a published dataset should warn the user that it may still be published to

# Drawbacks
[drawbacks]: #drawbacks

The major drawback here is once information is within an IPFS repo, it's available on the open web, which is why "unpublished" data is better described as "unlisted". One could argue that we should wait until either the capacity to keep IPFS hashes private, or wait until Qri has private datasets.

This would be ideal, but it's not practically feasible on our current timeframe.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

A possible alternative would be to mimic git's `push` / `pull` system.

# Prior art
[prior-art]: #prior-art

Youtube. Vimeo. On many vid sharing platforms you can set a video to "unlisted", that's kinda what's going on here with unpublished datasets.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* We still need some sort of rebase / squash / merge system that allows users to edit & republish histories
* Publish currently ignores whatever the previous publication status was. We should be checking calls to publish against any existing history and require a `--force` flag if the new publish re-writes history
* We should also be warning users when they publish changes to a schema, possibly requiring a `--force` flag to publish schema changes. Schema changes have a habit of breaking transform scripts
* How publishing to multiple registries works. (I'm guessing a CLI example will involve giving registries names and adding a `--registry` flag that accepts a comma separated list of registries to publish to)

Syncing version histories is now a much bigger question that'll have to be addressed. This should be the subject of another RFC, ideally happening sooner-rather-than-later.