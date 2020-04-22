- Feature Name: Storage command
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Create a new command `storage` for managing qri's disk usage.

# Motivation
[motivation]: #motivation

We're missing a command for managing storage. Work on _retention strategies_ from another RFC has forced the issue. We need a robust command for managing storage locally, and a way to manually reach out to a remote to make storage management asks of them.

Remove is currently covering _some_ of this territory for us, but not nearly enough. Adding storage management to `remove` would be overloading `remove`'s primary purpose of manipulating history. So we need a new command!

This RFC proposes starting with two subcommands under a new command: `qri storage`:
* `qri storage stat`
* `qri storage drop`

These commands should provide the most vital action required for any retention strategy: removing stored datasets independent of history manipulation, and set the stage for better understanding storage cost with a stats command.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

```
$ qri storage --help
storage manages disk usage. The default storage location is your local disk, but
most commands allow specifying a remote

Available Commands:
  drop       purge versions from storage, potentially reclaiming disk space
  stat       calculate disk usage for a dataset or the entire data store
```

```
$ qri storage stat
Overview calculates a summary of your 

1. A list of dataset versions that are 

Usage:
  qri storage stat [REMOTE] [DATASET...]

Examples:
  # calculate storage stats across your entire datastore
  $ qri storage stat

  # show storage stats for a dataset
  $ qri storage stat b5/world_bank_population

  # show overlap between two different dataset histories
  $ qri storage stat b5/world_bank_population b5/country_codes

  # show storage cost of a specific commit:
  $ qri storage stat 

  
Flags:
  -f, --format          output data format. (pretty|json|json-pretty)
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
* gross overhead: how much space would it take up to write all of these versions to seperate files?

The stats should present as much of this info as possible, scoped to the arguments provided.


### purge: remove stored datasets without affecting history
```
$ qri storage purge --help
Purge deletes dataset version data, freeing up disk space. Purge does not edit 
history. To edit version history, use the remove command.

Dataset versions take up disk space, and at some point need to be managed.
Becase datasets and storage capacity vary dramatically the need for this

You can also use purge to request a remote drop version data. This is often

Usage:
  qri purge [REMOTE] DATASET [flags]

Examples:
  # drop

Flags:
  -a, --all               purge all versions, overrides revisions
  -r, --revisions         purge a number of versions from the latest commit
  -t, --tail              purge stored versions starting from the earliest

```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The purpose of stats are to answer a few primary questions. All of them deal with a _set of blocks_. 


The foundation for all of this work should be a `dag.Info` & `dag.Manifest`. Info provides the block size information, and manifests provide the . `dag.Info` currently looks like this:

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

Calculating storage states
1. Load logbook info for a dataset
2. Build a `dag.Manifest` of all stored roots, keeping a slice of block sizes while you're at it
3. Turn that into a `dag.Info`
4. Call methods on `dag.Info` that produce storage stats

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art



# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Knowing which datasets depend on a block
One of the problems this RFC doesn't tackle properly is _knowing_ if removing a block will free up storage space, because we aren't leveraging pin reference counting. In an ideal world we'd have a low-level command like this:

```
qri storage deps: for a given block, show which dataset versions depend on that that block
```

If I understand this correctly, that'd require going from a many-to-one in the wrong direction. Expensive.