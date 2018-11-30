- Feature Name: dataset manifests
- Start Date: 2018-10-25
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Define the different names, limitations, and use cases for subsets of dataset information

# Motivation
[motivation]: #motivation

As Qri develops, we're running into situations where we need tools for quantifying a _subset_ of a dataset. Some examples from recent work:

* storing only a "preview" of a dataset on Qri registries, where a preview is a dataset without the body
* producing _size-bounded_  datasets. Datasets can theoretically be any size, because of this, we need some way to get meaningful _subsets_ of a dataset. 
* tracking the progress of transferring a dataset from one place to another, possibly resuming that transfer part-way-through on a failure

Datasets in Qri are organized into _immutable graphs_ of content addressed blocks. This places restrictions on what it means to "have" a dataset. Currently you either have the entire dataset graph, or you have to . Anything in between is "ephemeral" in the sense that 

By combining a graph with a manifest, we can make _meaningful subsets_ of datasets.

Because we intend to use datasets in a peer-2-peer context, manifests need to contend with _trust_. The design of manifests and techniques that rely on manifests must take abusive use into account.

Manifests get consumers of a subset of a graph out of an information-poor context. If a peer has only part of a graph, but also has a manifest, they know _which_ part they have

Manifests have a few key use cases for us:
* 


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
  * The element after highest index present in the `links` array is also the split point between leaf and branch nodes
* supplying the same dag to the manifest function must be deterministic: `manifest_of_dag = manifest(dag)`
  * Therfore: `hash(manifest_of_dag) == hash(manifest(dag))`
* In order to generate a manifest, you need the full DAG, or at least a trusted source of DAG IDs and links

Manifests are intentionally limited in scope to make them easier to prove, faster to calculate, the list of nodes can be used as a base other structures can be built upon. by keeping manifests at a minimum they are easier to verify, forming a foundation for other.

Based on some light testing, if leaves have a max size of 256KB, a 100KB manifest encoded as CBOR can represent somewhere in the vicinity of 1GB of content. Having manifests that are roughly 1:100 in size makes them a reasonable choice for preflighting a DAG transfer over a network.

I'd like to introduce two structures that build on Manifests: `DAGInfo` and `Completion`:

### DAGInfo
DAGInfo is intended to work like `os.FileInfo` for graph-based storage: a struct that describes important details about a graph. In terms of trues, the contents of DAGInfo should be considered gossip. DAGInfo's are *not* deterministic, and are intended to fill the role of application-specific needs when describing DAGs. 

For Qri's purpos DAGInfo contains two fields: `sizes` and `paths`. `sizes` lists the size of each node in bytes, where each array index corresponds to the ID specified in `manifest.Nodes`. `sizes` is useful for a number of things, the most obivious being getting the the total size of the DAG by summing of all elements in the `sizes` array.

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
    "dataset.json": 2
  },
  "sizes": [150,145,2340,256000,3404]
}
```

Setting a `paths` property key `dataset.json` to `2` lets us leverage `links` to construct the subgraph starting at index 2 to construct the subraph at `dataset.json`.


### Completion
Completion tracks the presence of blocks described in a manifest. Completion can be used to store transfer progress, or be stored as a record of which blocks in a DAG are missing each element in the slice represents the index a block in a manifest.Nodes field, which contains the hash of a block needed to complete a manifest the element in the progress slice represents the transmission completion of that block's _locality_ (weather or not it's on your hard drive). Completion values must be a number from 0-100, where 0 = nothing locally, 100 = block is local.

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

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art

* Manifests started as a proposal for inclusion in the IPFS GraphSync protocol: https://github.com/ipld/specs/pull/66
* Bittorrent Torrent files: https://en.wikipedia.org/wiki/Torrent_file

# Unresolved questions
[unresolved-questions]: #unresolved-questions