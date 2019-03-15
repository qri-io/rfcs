- Feature Name: dataset manifests
- Start Date: 2018-10-25
- RFC PR: [#32](https://github.com/qri-io/rfcs/pull/32)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Define dataset manifests: a building block for managing subsets of a dataset and point-to-point block syncing.

# Motivation
[motivation]: #motivation

At Qri we've run into a pretty serious snag: sending datasets from point-to-point in a manner that's roughly comparable with a centralized service in terms of speed. While working on `qri publish`, we want to pin the blocks in a dataset on an IPFS node that is always available so others can get to it. This requires getting the dataset from a local machine to a publically-hosted Qri registry. This is the rough equivalent of `git push origin master` for Qri.

Without the Qri version of `git push origin master`, Qri's usefulness is degraded. We've written a test that uses out-of-band communication to ask a remote node to `ipfs pin add` while the local machine is on the distributed web, and it's taking anywhere from 5-7 minutes to pin a ~200kb dataset, even with an explicit request to directly connect the two nodes making the transfer.

While we're most certainly doing a few things wrong in terms of network configuration, this "ambient pinning" approach isn't producing a consistent-enough user experience. After initially [proposing some upstream changes](https://github.com/ipld/specs/pull/66) in IPFS, it's become clear that it'll be more useful to others to solve our own pain points with an eye toward generalization. If we come up with a solution that works well, it might be useful in other situations.

There is also a related-but-separate set of problems we've encountered when working with dataset transmission that all involve situations where we need tools for quantifying a _subset_ of a dataset. Some examples from recent work:

* storing only a "preview" of a dataset on Qri registries, where a preview is a dataset without the body
* producing _size-bounded_  datasets. Datasets can theoretically be any size, because of this, we need some way to get meaningful, predefined subsets of a dataset. 
* tracking the progress of transferring a dataset
* comparing blocks of a dataset between two versions to examine deduplication

Because datasets in Qri are organized into _immutable graphs_ of content addressed blocks. There are two versions of what it means to "have" a dataset. In the first sense of the word, you have the entire graph of blocks that make up a dataset pinned locally. The other meaning of "having" is the result of reading the canonical hash and encoding them into an API, CLI, or exported dataset.

This second sense of "having" is obviously vital for Qri to work properly because the entire world doesn't read raw IPFS blocks, and never will. The problem is the first version is all-or-nothing. You either have the _entire_ dataset graph, or you have to consider the result an emphemeral byproduct of the canonical source.

Before we approach the point-to-point sync problem, I'd like to introduce **Manifests** as a method of describing all blocks that compose a dataset in a concise, repeatable way. By having a clear method of defining a complete dataset, we have the tools we need to have meaningful discussion about subsets of a dataset, and make progress on the all-or-nothing issue.

Because we intend to use datasets in a peer-2-peer context, manifests need to contend with _trust_. The design of manifests and techniques that rely on manifests must take abusive use into account.

Manifests get consumers of a subset of a graph out of an information-poor context. If a peer has only part of a graph, but also has a manifest, they know _which_ part they have, and more importantly, which parts they're missing.

Once manifests are in place, I'll follow up with an RFC on block-syncing that uses manifests as a building block for an rsync-like delta replication system.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Manifests
A manifest is a determinsitc description of a complete directed acyclic graph (DAG). Manifests contain no content, only a description of the _structure_ of a graph (nodes and links). Manifests are inspired by the `info` section of bittorrent "torrent" files. 

Manifests are built around a flat list of node identifiers (usually hashes) and an array of links. A link element is a tuple of `[from,to]` where `from` and `to` are indexes in the `nodes` array. Manifests always describe the **full** graph: a root node and all it's descendants. Here's an example graph with four nodes: `A,B,C,D` and three Links: `A->B`, `A->C`, `C->D`.
```
    A
   / \
  B   C
       \
        D
```

The manifest for this DAG written as JSON would be:
```json
{
  "nodes": ["A","C","B","D"],
  "links": [[0,1],[0,2],[1,3]]
}
```

A valid manifest has a few requirements:
* The array of nodes MUST be sorted by number of descendants. 
* When two nodes have the same number of descendants they MUST be sorted lexographically by node ID
* The list of links MUST be sorted from low-to-high tuple values. eg: `[0,1]`, `[1,1]`, `[1,2]`.

These requirements create a few nice properties:
* The means that for DAGs with a single root node, the root ID will always be index 0 of `nodes`
* Using this `nodes` organization, all leaf nodes will be grouped at the end of the list. 
  * The element after highest `From` index present in the `links` array is also the split point between leaf and branch nodes
* supplying the same DAG to the manifest function must be deterministic: `manifest_of_dag = manifest(dag)`
  * Therefore: `hash(manifest_of_dag) == hash(manifest(dag))`
* In order to generate a manifest, you need the full DAG, or at least a trusted source of DAG IDs and links

Manifests are intentionally limited in scope to make them easier to prove, faster to calculate, the list of nodes can be used as a base other structures can be built upon. By keeping manifests at a minimum they are easier to verify, forming a foundation for other.

Based on some light testing, if leaves have a max size of 256KB, a 100KB manifest encoded as CBOR can represent somewhere in the vicinity of 1GB of content. Having manifests that are roughly 1:10,000 in size makes them a reasonable choice for preflighting a DAG transfer over a network.

I'd like to introduce two structures that build on Manifests: `DAGInfo` and `Completion`:

### DAGInfo
DAGInfo is intended to work like `os.FileInfo` for graph-based storage: a struct that describes important details about a graph. In terms of trues, the contents of DAGInfo should be considered gossip. DAGInfo's are *not* deterministic, and are intended to fill the role of application-specific needs when describing DAGs. 

For Qri's purpose DAGInfo contains two fields: `sizes` and `paths`. `sizes` lists the size of each node in bytes, where each array index corresponds to the ID specified in `manifest.Nodes`. `sizes` is useful for a number of things, the most obvious being getting the the total size of the DAG by summing of all elements in the `sizes` array.

DAGInfo _paths_ associate labels with meaninful subsection of a manifest. They're represented as a map of strings to manifest node indicies, which indicate the "root" of the dag subgraph that constitutes the path. For example, we could associate the label `dataset.json` with it's root in the DAG:

```
    A
   / \
  B   C <- dataset.json
     / \
    D   E
```

We can construct the following DAGInfo:
```json
{
  "manifest": {
    "nodes": ["A","C","B","D","E"],
    "links": [[0,1],[0,2],[1,3],[1,4]]
  },
  "paths": {
    "dataset.json": 1
  },
  "sizes": [150,145,2340,256000,3404]
}
```

Setting a `paths` property key `dataset.json` to `2` lets us leverage `links` to construct the subgraph starting at index 2 to construct the subraph at `dataset.json`.


### Completion
Completion tracks the presence of blocks described in a manifest. Completion can be used to store transfer progress, or be stored as a record of which blocks in a DAG are missing each element in the slice represents the index a block in a `manifest.nodes` field, which contains the hash of a block needed to complete a manifest the element in the progress slice represents the transmission completion of that block's _locality_ (weather or not it's on your hard drive). Completion values must be a number from 0-100, where 0 = nothing locally, 100 = block is local.

Note that completion is not necessarily linear. For example the following is 50% complete:
```
manifest.Nodes: ["QmA", "QmB", "QmC", "QmD"]
progress:       [0, 100, 0, 100]
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

```go
type Manifest struct {
	Links [][2]int `json:"links"` // links between nodes
	Nodes []string `json:"nodes"` // list if CIDS contained in the root dag
	Root  int      `json:"root"`  // index if CID in nodes list this manifest is about. The subject of the manifest
}

type DAGInfo struct {
	Manifest *Manifest      // DAGInfo is built upon a manifest
	Paths    map[string]int // sections are lists of logical sub-DAGs by positions in the nodes list
	Sizes    []uint64       // sizes of nodes in bytes
}

type Completion []uint16
```

# Drawbacks
[drawbacks]: #drawbacks

* Manifests are additional stuff that needs to be shipped around, and by design include no actual dataset content. In this sense they're a kind of tax
* Manifests have to be generated, and likely cached. This is additional storage and code complexity

I do believe the benefits in terms of having a reliable assumption about DAG structure, and reduced code complexity from being able to leverage this assumption is well worth both of these drawbacks

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could attempt to implement something like _selectors_, which Protocol Labs is working on. This solution seems more general than our needs.

# Prior art
[prior-art]: #prior-art

* Manifests started as a proposal for inclusion in the IPFS GraphSync protocol: https://github.com/ipld/specs/pull/66
* Bittorrent Torrent files: https://en.wikipedia.org/wiki/Torrent_file

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* We still need to figure out how to store Manifests/DAGInfos/Completion
* This is too large to be sending to the frontend / over an API without being explicitly requested. We'll need to develop some sort of shorthand for representing completion