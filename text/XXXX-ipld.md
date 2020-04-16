- Feature Name: ipld
- Start Date: 2019-06-01
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Switch Qri from writing IPFS 

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Terms used in this document:
* **[IPLD](https://github.com/ipld/ipld)**

IPLD & unixfs There's a tension here that that in the end refuses to resolve. ipld doesn’t distinguish between  persistence graph and the representation graph

Dataset bodies are unbounded 

Ndjson sneaks an extra-textual delimeter. 

Use diso writer to add raw ipld data blocks. 

This would be a lossy process. We’re losing the archival qualities qri has now. 

What about head & tail matter? Dsio could get a new set of methods read/write head and read/write tail.

——

While switching from go-ipfs 0.4.18 to 0.4.21, the plumbing for creating unixfs v1 DAGs flipped to using mfs under the hood. This causes some problems for our current approach to keeping Qri datasets on IPFS.

We’ve favored unixfs for two major reasons

During the process of DAG formation, qri has been using the output of Add events to edit the primary dataset document, connecting each file-added event with its input path by using a predefined file layout. Add events deliver the root hash of a file. We would listen to these events, updates the dataset document, and when all expected documents are accounted for we add the dataset document to the dag, and finalize it, wrapping all files in a directory.

Versions of IPFS after 0.4.18 use a different approach to creating DAGs, which  enforces assumptions that break our in-flight modification trick. Without substantial modification to the underlying dag creation mechanisms, we can’t use our old approach.

The problem boils down to the (reasonable) assumption that the data you’re adding to IPFS is finalized before the dag creation process begins. We’ve been exploiting this implicit feature, and it’s no longer feasible.

So, we have two main options:
* Write our own code to keep our current behavior
* Embrace IPLD for dataset definition

Both have trade offs. 

Writing our own code is more things we have to maintain.

IPLD DAGs cannot be rendered to HTML representations by an IPFS gateway.

This is simulating very “ipld like” behavior, while maintaining our requirements of having a nice native presentation of a qri dataset. Ultimately it’s “just files”.

We do this because of the way the gateways render unixfsv1 DAGs.

As an intermediate step I think we should focus on expressing everything except the body, viz, and transform scripts as IPLD.

Datasets will no longer be stored as Unixfs directories, instead the root of a dataset will be an IPLD representation of the former dataset.json object.

To maintain backwards compatibility, requesting rootHash/dataset.json should be transparently re-routed to the root, and converted to JSON representation. For this reason we should reserve the /dataset path in spec. (In the future we may want to turn this into a conversion feature using file extensions. For example: dataset.cbor

The biggest question is how to accomplish this change. Intersecting the assumptions of CAFS & IPLD, things seem to line up mainly because IPLD was designed to accommodate this use case. “Ipfs dag put [data]” has an interface for declaring links embedded in the data provided. Links are an object with a “/“ key who’s value is a hash string. 

I’m not sure I’ll ever understand the rationale that went into bucking the rest of the linked-data community, which uses “$ref” to refer to links, but we’re here to build bridges, so let’s move on.

This link declaration syntax should allow cafs to maintain the core aspects of its current interface. The seam between cafs and dsfs is where the most change will occur.

I would like to implement the switch to IPLD with some high level changes:
* add a new
* add a new package to `github.com/qri-io/dag` called `ld` that defines generic interfaces for reading & writing linked data. it will contain the definition of a `Link` struct 
* add a new package in linkded data:  `github.com/dag/ld/ipld`. `ipld` will implement the `ld` interface



There are two places most heavily affected by this change: dsfs & cafs.

The biggest area in need of change is [`dsfs.WriteDataset`](https://github.com/qri-io/dataset/blob/c5d509df5415c13ef5eae5a7b3d1591b0514132d/dsfs/dataset.go#L502)

```go
// WriteDataset writes a dataset to a cafs, replacing subcomponents of a dataset with path references
// during the write process. Directory structure is according to PackageFile naming conventions.
// This method is currently exported, but 99% of use cases should use CreateDataset instead of this
// lower-level function
func WriteDataset(store cafs.Filestore, ds *dataset.Dataset, pin bool) (string, error) {
  // ... 185 lines of code
}
```

This function is the only place in our codebase that makes use of the `qfs.Adder

cafs is our abstraction for content addressed file systems. it's part of qfs, our family of file system-related code

While we're here, we might as well fix a few things.

I have concerns about IPLD. If I understand correctly the motivation for the project came from the 

There is a relationship between the persistence graph and representation graph that IPLD falls on the other side of. Unixfs intentionally constrained the representation graph to a common, tractable form: 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

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

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
