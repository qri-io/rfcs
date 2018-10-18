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
> -- @mbattifarano

What I'm hearing is Qri doesn't have clear mental model of what will happen when the user runs `qri new` or  `qri save`. In some ways this makes sense, Qri is brand new. For that same reason it's important to introduce a consistent, digestable mental model as early as possible.

The ambiguity around what goes in the transform and what goes in the dataset.yaml is difficult to decipher when it's not clear which is "processing first". We should make it clear that the details of dataset.yaml are processed onto the dataset _before_ the skylark file, but validation happens after.

We need to clearly document _how outside data gets into Qri_ in a way that becomes natural with practice. Ideally this process has as few exceptions & "gotchas" as possible. While defining such a mental model, we've come up with changes that have 

Currently, there are three ways to get a dataset into your Qri repo:
* `new` - create a new dataset
* `save` - write an update to an existing dataset
* `add` - download a peer's dataset

Besides the distinction of requiring network access to function, `add` doesn't _ingest_ data into Qri from an external source, instead moves grabs a dataset from the network. This RFC ignores the `add`  command, implying that `add` should remain a distinct thing that's about getting datasets from a peer. I think this is correct because `add` is both technically and cognitively distinct from `new` and `save`. _(NB: the name `add` may be changed in the future)_

### Proposed Change:
**This RFC proposses merging "new" and "save", making all commands bring outside data into Qri use the same mental model and codepath**. By merging `new` and `save` into just `save` command, we can write documentation that builds up a single mental model of how external data gets into Qri, while also simplfying our code. Users can leverage this mental model to predict what Qri will do, even in a novel use. Users familiar with git should begin to think of `qri save` as `git commit`. A simple, frequently used process that behaves consistently.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order to get a "clear mental model" I think we have to back all the way up, comparing Qri's versioning model to git. By building off a git metaphor, it's easier for someone familiar with Git to understand how Qri is different. Much of this isn't new, but what should be ratified in this RFC is the _quality of the explanation_. By reading this someone familiar with git should be able to understand the process Qri undertakes to create a dataset. Hopefully this explanation can be copy-pasted into our docs if this RFC is accepted.

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

With Qri we're making a concerted effort to reduce the complexity associated with version control, getting it into a form that's palettable to an audience that doesn't consider themselves "technical". This is why we use `save` instead of `commit` (if you're all about git, don't worry, `commit` is aliased to `save` on the command line).

> _Note: There is currently no way to merge a dataset that's been forked. This is bad. We're working on it. Our current thinking is it'll work as closely as possible as git branch merging: `qri merge peer/dataset me/dataset` should have the effect of stitching together the two specified histories into a single entity at `me/dataset`. Unlike git all datasets share a data model, so this should work even with completely different histories._

### Comparing Qri Save to Git Commit

| Qri                           | git                             |
| ----------------------------- | ------------------------------- |
| `cd project`                  | `cd project`                    |
| `qri save --file=dataset.yml` | `git add -A`                    |
|                               | `git commit -m "changed stuff"` |
| `qri publish`                 | `git push origin master`        |

#### Both Qri and git use the local filesystem & `pwd`
Before we get started, it's worth noting that both git and qri use the present-working-directory (`pwd`) to dereference filepaths. Git stores commits in a `.git` directory, one for each repo. Qri instead stores commits in a "content-addressed file system", which is stored in one place on your hard drive. Instead of one `.git` directory in each repository, there is one `.qri` directory for your entire file system,

_It is possible to have multiple installations of Qri on your machine at once by manipulating the `$QRI_PATH` value. At time of writing (Oct. 2018), we haven't documented this properly._

#### Git Commit Stores a snapshot, Qri Save Computes a snapshot
while `qri save` and `git commit` are conceptually comparable, they're fairly different animals under the hood.

Git is a _generic_ version control system. It's possible to version any binary data with git, with varying (incredible) degrees of success. The assumptions git makes about the information being versioned are very light (for example, git assumes many differences will occur across newline breaks).

Git carries the assumption that the data provided to it is the _source of truth_. Git's job is to version things, the user's job is to tell git _what to version_.

Like git, Qri's jobs is to version things. However, Qri is _not_ a generic version control system. Instead Qri only makes versions of a predefined dataset model. This association narrows the number of use cases in which Qri will function, but  delivers the benefit of making Qri _semantically versioned_. Because Qri is programmed to understand the intent & uses of various components of the dataset model, Qri can compute many important values on the user's behalf. This "acting on a user's behalf" comes from a long line of data management practices. When managing data there is often too much information for the user to feasibly review each row

Through the use of transform scripts, the user has tools to extend & control how a dataset is computed. Qri knows to run the transform in the first place because datasets have exactly one designated place for putting a transform script.

#### Qri has no staging area
While it is true that "qri save" is doing one less step than the git variant, git users will quickly point out that `git commit -am "changed stuff"` consolidates `git add` and `git commit` into a single step, so it's not much of a time-saver from a keystrokes perpective. Instead I've broken `git add` and `git commit` into two commands to show that Qri has no concept of a staging area.

Instead of a staging area, Qri uses the concept of a _dry run_ to see what would happen without actually committing anything. 

#### Qri registries are like git remotes
With Qri it's possible to just run `qri save` and follow that with `qri connect`, and now others will be able to see and consume your dataset.

Qri Publish is a far less necessary step on 

-- --

## Save creates a dataset snapshot
A dataset commit is a _snapshot_ of a dataset at a given point in time. The following oulines the steps that goes into computing a snapshot

### Steps
* **Prepare**
  * screen for incorrect input
  * check for required values
  * if `ds.name` isn't provided, infer a name from provided `dataset.yml` bodyFile
  * if no `prev` value is supplied, confirm the supplied name isn't taken
  * turn a _patch_ into an _update_:
    * load any previous version
    * merge provided patch onto previous version
    * remove any values set to `null` by the provided patch
    * remove stale computed values like commit.title & commit.message
* **Run Transform**
  * transform script is passed the prepared update from the user (eg: things supplied in dataset.yaml)
  * previous verion of input dataset (if any) is accessible via a helper function: `qri.load_previous(ds.name)`
* **Compute Automatic Values**
  * if no `ds.structure.schema` is supplied but body data is provided, a `ds.structure.schema` is inferred from body data. often defaulting to one of `{ "type" : "array" }` or `{ "type" : "object" }` 
  * calculate `ds.structure.errorCount` by running `ds.structure.schema` against `ds.body` & counting returned number of errors
  * calculate `ds.structure.checksum` a checksum of `ds.body`
  * calculate `ds.structure.length`, the length of the dataset in bytes
  * calculate `ds.structure.entries`, the number of top-level entries in the dataset
  * set `ds.commit.timestamp` to the current time
  * if `ds.commit.title` is empty, calculate a string representation of the difference since last version
  * calculate `ds.commit.signature` using the peer private key
* **Create Snapshot**
  * write the data to a content-addressed file system (IPFS)

### User supplied Patches vs. Updates
In Qri we want to support users supplying both "patches" and "updates". For now, Qri should accomplish this by assuming all input is a "patch", as a "user supplied update" can be interpreted as supplying _all_ values of a dataset.

The most common instances of user-supplied updates will be when using the frontend (our frontend is configured to supply call the API with _all_ values), and when running `save` with an exported dataset archive.

Difference between a update and a patch:

* updates are a _complete_ dataset snapshot
  * `qri save --file=dataset.zip` is an example of a command-line update
  * stale commit messages are ignored
* patches are a _set of changes_
  * to get an update from a patch, we must populate missing values from the previous version
  * `qri save --file=dataset.yaml` is an example of a patch
  * http `PATCH` requests make the API behave like the command-line 
  * when in a patch, supply `null` to zero-out a field

<!-- ### Turning Patches into Updates:
One of the toughest things to sort out with snapshots

The Patch Value Cascade:
* User Supplied Values
* Scripts
* Previous Entries
* Computed Values -->

#### stale commit details are always ignored
There is one exception to this rule: commit `title` and `message`.

git requires commit information to be provdided via API interaction. Qri allows you to supply commit information in the dataset.yml document. Qri also _exports_ commit information along with a dataset. To smooth out the process of re-importing, any time commit details exactly match a previous version are considered stale, and are removed in the prepare step.

#### supplied files are always full replacements
This comes from git. If you give us a file, Qri will replace the previous version with the provided one. providing `null` removes the previous file in the new snapshot if one existed.

here's all the places we currently accept files:
```yaml
viz:
  scriptpath: ""
transform:
  scriptpath: ""
bodypath: ""
```

### saving a dataset you don't own creates a fork
GitHub introduces the concept of a "fork" which clones a repository from one user's namespace to another.

In Qri it's perfectly acceptable to run `qri save`, with a `previous` value that is someone else's dataset. This will "fork" the dataset into your namespace, keeping the previous user's version histry.

### `new` is just `save` without a previous reference
Creating a commit doesn't require a previous reference. Voilà! No more need for the `new` command.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Here's the new high-level sketch of call stack for a successful `save` operation, with the corresponding creation step on the right:

```
lib.DatasetRequests.Save                Prepare
  lib.DatasetRequests.validateSave      |
  actions.SaveDataset                   |
    actions.prepareDataset              |
      if ds.Prev:                       |
        core.PrepareDatasetUpdate       |
      else:                             |
        core.PrepareDatasetNew          |
    core.CreateDataset                  |
      if ds.Transform:                  Transform
        core.ExecTransform              |
      dsfs.CreateDataset                Compute Automatic Values
        dsfs.prepareDataset             |
        dsfs.writeDataset               Create
```

# Drawbacks
[drawbacks]: #drawbacks

Merging these two commands will pile up the number of command line flags & API params, which is kinda irritating.

More crucially, there are lots of caveats here. It may be worth working harder now to simplify this model before merging

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

While I think it's important to unify new & save into a simple, digestable model, I do think there's room for "advanced" tools that circumnavigate this mental model. The issue this RFC grapples with is establishing a simple process to use as a starting point. Later on I'd like to introduce a new command `qri patch` that allows changing dataset components without running `save`. I'm hoping the "patch' info described here will evolve into the atomic unit used for this command, but it'll take some time to develop.

# Prior art
[prior-art]: #prior-art

Um, yeah, [git](https://git-scm.com).

Kubernetes Configuration files are another source of inspiration for Qri "computed datasets".

This builds on previous splitting of add described in [qri-io/notes#3](https://github.com/qri-io/notes/issue/3).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

There's still latent tension around naming that isn't yet (and may never be) solved. In my opinion these commands aren't obviously disctinct from the outside:
* `qri save`
* `qri add`
* `qri get`
Ideally these would be named in a way that is both "instinctual" and easy to tell apart. I'm not sure that's possible.
