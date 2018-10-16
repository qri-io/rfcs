- Feature Name: Define Dataset Creation
- Start Date: 2018-11-15
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Simplify "new" and "save" into one algorithm that defines the high-level steps for creating a dataset, making learning Qri easier.

# Motivation
[motivation]: #motivations

An early Qri user recently gave us some feedback:

> when I was writing a transformation, it wasn't super clear what needed to happen in the sky file and what needed to happen in the dataset.yml. I started out with both and had to deal with a bunch of errors which stemmed from (I think) me accidentally specifying things in both places or in the wrong place. In particular I wasn't sure what runs first. For example, does the ds in the transformation know the schema specified in dataset.yml or does qri check the output of transform against the specified schema? 

To me the root of this problem concern is not having a clear mental model of what will happen when the user runs `qri new` or  `qri save`. In some ways this makes sense, because Qri is brand new. For that same reason it's important to introduce a consistent, digestable mental model as early as possible. To properly respond to this feedback, I think we need to clearly document _how outside data gets into Qri_ in a way that becomes natural with practice, but has as few exceptions & "gotchas" as possible.

Currently, there are three ways to get a dataset into your Qri repo:
* `new` - create a new dataset
* `save` - write an update to an existing dataset
* `add` - download a peer's dataset

Besides the distinction of requiring network access to function, `add` is distinct from `new` and `save` in the sense that no data is being _ingested_ into Qri from an external source. 

This RFC proposses merging "new" and "save", making all commands that put outside data into Qri happen in one place.  I think `add` should remain a distinct thing that's about getting datasets from a peer, an operation that is both technically and cognitively different.

By merging the two commands, we can write documentation that builds up a single mental model of how external data gets into Qri. Users can leverage this mental model to predict what Qri will do, even in a novel use. Those familiar with git should begin to think of `qri save` as `git commit`. A simple, frequently used process that behaves consistently.

While I think it's important to unify new & save into a simple, digestable model, I do think there's room for "advanced" tools that circumnavigate this mental model. The issue this RFC grapples with is establishing a simple process to use as a starting point.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To get this right it's worth backing all the way up, comparing Qri's versioning model to git. To paint a complete picture. Hopefully this explanation can be copy-pasted into our docs if this RFC is accepted.

### Comparing Commits: Qri, GitHub, Git
Qri builds on ideas popularized in both git and GitHub.

| Qri        | GitHub       | git        |
| ---------- | ------------ | ---------- |
| profile    | profile      | user       |
| repository | repositories |            |
| dataset    | repository   | repository |
|            | branch       | branch     |
| commit     | commit       | commit     |

<!-- I view much of Qri's decentralization work as the process of moving features from "GitHub" to "git". .  does onto peers. -->

Qri intentionally has no functionality akin to git branches. Qri is built in a post-GitHub era where the notion of a _profile_ has become an intrensic part of the git experience. Feel free to mentally replace "GitHub" with "GitLab" or "GitService" if you like, but GitHub _did_ popularize these concepts.

<!-- The proliferation of GitHub -->

Qri is different because data has a different,broader audience than software devs. Microsoft excel has long since surpassed 950 million downloads, GitHub has roughly ~10 million users, git has, I dunno, double that?

With Qri we're making a concerted effort to reduce the complexity associated with version control, getting it into a form that's palettable to an audience that doesn't consider themselves "technical". This is why we use `save` instead of `commit` (don't worry, `commit` is aliased to `save`, plz don't get hyphee devs).

> Note: There is currently no way to merge a dataset that's been forked. This is bad. We're working on it. Our current thinking is it'll work as closely as possible as git branch merging: `qri merge peer/dataset me/dataset` should have the effect of stitching together the two specified histories into a single entity at `me/dataset`. Unlike git all datasets share a data model, so this should work even with completely different histories.

### Comparing Qri Save to Git Commit

| Qri                           | git                             |
| ----------------------------- | ------------------------------- |
| `cd project`                  | `cd project`                    |
| `qri save --file=dataset.yml` | `git add -A`                    |
|                               | `git commit -m "changed stuff"` |
| `qri publish`                 | `git push origin master`        |

#### Both Qri and git use the local filesystem & `pwd`

#### Git Commit "Stores a Snapshot", Qri Save "Computes a dataset"
while `qri save` and `git commit` are conceptually comparable, they're fairly different animals under the hood.

Git is a _generic_ version control system. It's possible to version any binary data with git, with varying (incredible) degrees of success. The assumptions git makes about the information being versioned are very light (for example, git assumes many differences will occur across newline breaks).

Qri ties a document model to a version control system. This association narrows the number of use cases in which Qri will function, but  delivers the benefit of making Qri _semantically versioned_. many important values to be inferred on the user's behalf. Through the use of transform scripts, the user has total control over how a dataset is computed.

#### Qri has no staging area
While it is true that "qri save" is doing one less step than the git variant, git users will quickly point out that `git commit -am "changed stuff"` consolidates `git add` and `git commit` into a single step, so it's not much of a "time saver". Git commands are broken out to show that Qri has no concept of a staging area.

Instead of a staging area, Qri uses the concept of a _dry run_ to see what would happen without actually committing anything. 


#### Qri registries are like git remotes
Qri Publish is a far less necessary step

### Save creates a dataset snapshot
An initial stab at that mental model

A dataset commit is a _snapshot_ of a dataset at a given point in time.

Prepare
Transform
Validate
Save

### Steps
* **Prepare**
  * screen for incorrect input
  * check for required values
  * remove unchanged input
  * turn a _patch_ into an _update_
* **Run Transform**
  * transform scripts receive a _full snapshot_
  * transform functions execute in a canonical order, calling download, then transform
  * to pass extra values from one step to the next, supply a tuple of return values
  * error by calling `error()`
  * validation is performed post-transformation
    * use the "qri.validate" method to perform validation within a transform (currently not implemented)
  * access versions with `qri.load_previous(ds)`
* **Validate**
* **Create Snapshot**

### Patches vs. Updates

### Turning Patches into Updates:
One of the toughest things to sort out with snapshots

The Patch Value Cascade:
* User Supplied Values
* Scripts
* Previous Entries
* Inferred Values

#### stale commit details are always ignored
There is one exception to this rule: commit `title` and `message`.

#### supplied files are always full replacements
This comes from the git

### saving a dataset you don't own creates a fork


### `new` is just `save` without a previous reference


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Patch vs. Update
* updates assume a _full snapshot_
  * `qri save --file=dataset.zip` is an example of a command-line update
  * stale commit messages are ignored
* patches mean "infer previous values"
  * `qri save --file=dataset.yaml` is an example of a patch
  * http `PATCH` requests make the API behave like the command-line 
  * when in a patch, supply `null` to zero-out a field

Here's the new high-level call stack of a successful `save` operation:

```
lib.DatasetRequests.Save            // Prepare
  lib.DatasetRequests.validateSave  // |
  actions.SaveDataset               // |
    actions.prepareDataset          // |
      if ds.Prev:                   // |
        core.PrepareDatasetUpdate   // |
      else:                         // |
        core.PrepareDatasetNew      // |
    core.CreateDataset              // |
      if ds.Transform:              // Transform
        core.ExecTransform          // |
      dsfs.CreateDataset            // Validate
        dsfs.prepareDataset         // |
        dsfs.writeDataset           // Create
```

PrepareDatasetUpdate

# Drawbacks
[drawbacks]: #drawbacks

Merging these two commands will pile up the number of command line flags & API params, which is kinda irritating.

More crucially, there are lots of caveats here. It may be worth working harder now to simplify this model

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives


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

There's still latent tension around naming that isn't yet (and may never be) solved. In my opinion these commands aren't obviously disctinct from the outside:
* `qri save`
* `qri add`
* `qri get`
