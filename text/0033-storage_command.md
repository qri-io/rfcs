- Feature Name: storage command
- Start Date: 2020-04-21
- RFC PR: [#50](https://github.com/qri-io/qri/pulls/50)
- Issue: 

# Summary
[summary]: #summary

Create a new command `storage` for managing qri's disk usage.

# Motivation
[motivation]: #motivation

We're missing a command for managing storage. Work on _retention strategies_ from another RFC has forced the issue, as we can't move on to building those without first creating manual tools for inspecting and manipulating stored dataset versions. We need a robust command for managing storage locally, and a way to manually reach out to a remote to make storage management asks of them. This RFC sets the stage for moving toward retention strategies with commands for manually viewing & deleting storage.

the current `remove` command in qri v0.9.8 is currently covering _some_ of this territory for us, but not nearly enough. Adding storage management to `remove` would be overloading `remove`'s primary purpose of manipulating history. So we need a new command!

This RFC proposes starting with two subcommands under a new command: `qri storage`:
* `qri storage stat`
* `qri storage drop`

These commands should provide the most vital action required for any retention strategy: removing stored datasets independent of history manipulation, and set the stage for better understanding storage cost with a stats command.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Here's the proposed description for the storage command:

```
$ qri storage --help
storage manages disk usage. The default storage location is your local disk, but
most commands allow specifying a remote

Available Commands:
  drop       purge versions from storage, potentially reclaiming disk space
  stat       calculate disk usage for a dataset or the entire data store
```

And a description of the stat command:

```
$ qri storage stat
Stat calculates storage statistics for datasets. Showing how much space a
dataset, version, or versions consumes.

"storage stat" calculates four types of statistic:
* storage cost: how much disk space a stored dataset history consumes. Cost is 
  the sum of size of the set of blocks retained in a history
* commit overhead: how much unique space a commit takes in relation to other 
  stored commits. Overhead is th sum of blocks who's reference count is 
  1 within the set of blocks
* net overhead: sum of blocks in each DAG for _all_ commits in a history.
  Overhead is storage cost, if all versions were retained
* gross overhead: how much disk space the total set of versions represents.
  Gross overhead is the sum of listed dataset sizes, without any deduplcation.

Usage:
  qri storage stat [DATASET...]

Examples:
  # calculate storage stats across your entire datastore
  $ qri storage stat

  # show storage stats for a dataset
  $ qri storage stat b5/world_bank_population

  # show overlap between two different dataset histories
  $ qri storage stat b5/world_bank_population b5/country_codes

  # show storage stats for a dataset stored on the registry:
  $ qri storage stat b5/world_bank_population --remote registry

  
Flags:
  -f, --format          output data format. (pretty|json|json-pretty)
      --remote          show stats for dataset stored on a remote
```

We want stats to calculate a few things here, throwing out some initial things we want to know:

| term | description | formula description |
| ---- | ----------- | ------------------- |
| storage cost | how much disk space a stored dataset history consumes | sum of size of the set of blocks retained in a history |
| commit overhead | how much unique space a commit takes in relation to other stored commits | sum of blocks who's reference count is 1 within the set of blocks |
| net overhead | sum of blocks in each dag for _all_ commits in a history | 
| gross overhead | how much disk space the total set of versions represents | sum of listed dataset sizes |

* storage cost: how much space is this dataset taking up?
* commit overhead: Which commit should I delete to get the most space back?
* net overhead: how much space would it take to store this entire dataset?
* gross overhead: how much space would it take up to write all of these versions to separate files?

The stats should present as much of this info as possible, scoped to the arguments provided.


### drop: remove stored dataset versions without affecting history


```
$ qri storage drop --help
"storage drop" deletes dataset version data, freeing up disk space. Drop does 
not edit or remove version histories. To edit version history, use the delete 
command. To remove pulled datasets entirely, use the higher-level "qri drop"
command.

Dataset versions take up disk space, and at some point need to be managed.
Becase datasets and storage capacity vary dramatically the need for this

You can also use "storage drop" to request a remote drop dataset versions.
This can be helpful for staying under a storage cap by manually dropping
versions on the remote.

Usage:
  qri storage drop DATASET [flags]

Examples:
  # drop the 3rd-newest revision of a dataset:
  $ qri storage drop user/dataset~2

  # drop a stored version from the registry
  $ qri storage drop me/dataset@/ipfs/QmHashOfVersion --remote registry

Flags:
  -a, --all               purge all versions, overrides revisions
      --force             force qri to perform a dangerous drop operation
  -r, --revisions         purge a number of versions from the latest commit
      --remote            perform this drop on a speficied remote
  -t, --tail              purge stored versions starting from the earliest
```

For now, only one version can be dropped at a time. Attempting to drop HEAD should come with a warning, and require the `--force` flag to proceed. Attempting to drop a version that isn't stored is a no-op.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The purpose of stats are to answer a few primary questions. All of them deal with a _set of blocks_. 

Most of these changes will originate with the `dag` package. The foundation for all of this work should be a `dag.Info` & `dag.Manifest`. Info provides the block size information, and manifests provide the . `dag.Info` currently looks like this:

```go
// Info is os.FileInfo for dags: a struct that describes important
// details about a graph. Info builds on a manifest.
//
// when being sent over the network, the contents of Info should be considered gossip,
// as Info's are *not* deterministic. This has important implications
// Info should contain application-specific info about a datset
type Info struct {
	// Info is built upon a manifest
	Manifest *Manifest      `json:"manifest"`
	Labels   map[string]int `json:"labels,omitempty"` // sections are lists of logical sub-DAGs by positions in the nodes list
	Sizes    []uint64       `json:"sizes,omitempty"`  // sizes of nodes in bytes
}
```

Here's the current Manifest data structure:
```go
type Manifest struct {
  Links [][2]int `json:"links"` // links between nodes
  Nodes []string `json:"nodes"` // list if CIDS contained in the DAG
}
```

Manifests assume they describe one graph with a single root. To represent multiple graphs in the same manifest, we'll need to make one change:

```go
type Manifest struct {
  Graphs [][][2]int `json:"links"` // links between nodes
  Nodes []string `json:"nodes"` // list if CIDS contained in the DAG
}
```


### Changes to dsync
* With multiple roots in the same manifest, we can adjust `dsync` to send multiple roots in the same sessio
* We should add a new hook: `postReceiveManifest` that fires  _after_ the manifest has been sent, but before any blocks have moved.
* dsync should do a check to confirm there is sufficient blockstore space for any new blocks, and reject before any calling any hooks if not
* dsync should be adding per-block integrity checks that confirm the size of the block matches the stated size in the manifest. Any mismatch should abort the receive session.

### Calculating storage stats
The process for calculating storage stats should generally work like this:

1. Load logbook info for a dataset
2. Build a `dag.Manifest` of all stored roots, keeping a slice of block sizes while you're at it
3. Turn that into a `dag.Info`
4. Call methods on `dag.Info` that produce storage stats

# Drawbacks
[drawbacks]: #drawbacks

### Lots of plumbing
The storage command is largely plumbing, and ideally we aren't directing many users to this command at all.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Skip directly to automated methods
One major alternative to this would be to skip a `storage` subcommand and straight to a command like `cleanup` instead, without any nesting. My concern here is we're supposed to be building up toward retention strategies, and adding manual tools should help diagnose and develop these strategies safely.

### Use language other than "drop"
We have another RFC that details a top-level `drop` command that removes both history and dataset versions. it may be confusing to see the same command listed in two places. Personally, I'm a fan of the repition. In both cases `drop` is the word for removing redundant data.

# Prior art
[prior-art]: #prior-art

* [docker system prune](https://docs.docker.com/engine/reference/commandline/system_prune/)
* [brew cleanup](https://discourse.brew.sh/t/why-does-brew-cleanup-work-so-well/1819)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Knowing which datasets depend on a block
One of the problems this RFC doesn't tackle properly is _knowing_ if removing a block will free up storage space, because we aren't leveraging pin reference counting. In an ideal world we'd have a low-level command like this:

```
qri storage deps: for a given block, show which dataset versions depend on that that block
```

If I understand this correctly, that'd require going from a many-to-one in the wrong direction. Expensive.

### Automated Cleanup
Tools like `docker` and `brew` offer an automated way to clean up data that is safe to remove. The commands here don't cover such a thing, but instead form a foundation for adding a command like `qri storage cleanup` or `qri storage prune` that removes extraneous versions. 

### How Storage interacts with other qfs.Filestores
This RFC doesn't deal with how `qri storage drop` would affect other, non-IPFS filesystems. Ideally the `qfs.Filesystem` interface would be expanded to support this API any storage methods that opt into an interface necessary for running deletes.

### Specifying version ranges
The version presented here only permits dropping one commit at a time. Ideally we could specify a _range_ of versions to drop at once. git has the `..` syntax for this. We should add a spec for doing version ranges in a future RFC.