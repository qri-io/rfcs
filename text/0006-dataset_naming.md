- Feature Name: dataset_naming
- Start Date: 2017-08-14
- RFC PR: [#6](https://github.com/qri-io/rfcs/pull/3)
- Repo: https://github.com/qri-io/qri

_Note: This RFC was created as part of an initial sprint to adopt the RFC
process itself to help clarify our existing work. As such, sections of this
document are less complete than we'd expect from a new RFC.
-:heart: the qri core team_

# Summary
[summary]: #summary

Define the qri naming system, conventions for name resolution, and related jargon.

# Motivation
[motivation]: #motivation

As a decentralized system, qri must confront the Zooko's Triangle problem which is establishing a way of referring to datasets that is:

* human readable
* decentralized
* unique

Because Qri is assumed to be built atop a content-address file system as its storage layer, the properties of being decentralized & unique are already present. The Qri naming system maps a human readable name to the newest version of a dataset.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

It’s possible to refer to a dataset in a number of ways. It’s easiest to look 
at the full definition of a dataset reference, and then show what the “defaults” are to make sense of things. The full definition of a dataset reference is as follows:

    dataset_reference = handle/dataset_name@profile_id/network/version_id

an example of that looks like this:

    b5/comics@QmYCvbfNbCwFR45HiNP45rwJgvatpiW38D961L5qAhUM5Y/ipfs/QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y

In a sentence: b5 is the handle of a user, who has a dataset named comics, and its hash on the ipfs network at a point in time was `QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y`

The individual components of this reference are:

* handle - The human-friendly name that the creator is using to refer to theirself, somewhat analagous to a username in other systems. We need handles so lots of people can name datasets the same thing.
* dataset_name - The human-friendly name that makes it easy to remember and refer to the dataset.
* profile_id - A unique identifier to let machines uniquely refer to datasets created by this user, regardless of whether their handle is renamed.
* network - Protocol name that stores distributed data, defaulting to "ipfs".
* version_id - A unique identifier hash to refer to a specific version of a dataset, from an exact point in time.

### default to latest on ipfs

Now, having to type `b5/comics@QmYCvbfNbCwFR45HiNP45rwJgvatpiW38D961L5qAhUM5Y/ipfs/QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y`
every time you wanted a dataset would be irritating. So we have two defaults. 
The default network is `ipfs`, and the default hash is the lastest known version of a dataset. We say latest known because sometimes things can fall out of sync. If you're only working with your own local datasets, this won’t be an issue.

Anyway, that means we can cut down on the typing if we just want the latest 
version of b5’s comics dataset, we can just type:

    b5/comics

In a sentence: “the latest version of b5’s dataset named comics.”

### the me keyword

What if your handle is, say, `golden_pear_ginger_pointer`? First, why did you pick such a long handle? 
Whatever your answer, it would be irritating to have to type this every time, so we give you a special way to refer to yourself: `me`. So if you have a dataset named comics, you can just type:

    me/comics

In a sentence: “the latest version of my dataset named comics.” Under the hood, we’ll re-write this request to use your actual handle instead.

### drop names with hashes

Finally, it’s also possible to use just the hash. This is a perfectly valid dataset reference:

    @/ipfs/QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y

In this case we’re ignoring naming altogether and simply referring to a dataset by its network and version hash. Because IPFS hashes are global, we can do this across the entire network. If you’re coming from git, this is a fun new trick.

To recap:

All of these are ways to refer to a dataset:

* handle/dataset_name (user-friendly reference)

      b5/comics
    
* network/hash (path)

      /ipfs/QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y
      
* handle/dataset_name@profile_id/network/version_id (canonical reference)

      b5/comics@QmYCvbfNbCwFR45HiNP45rwJgvatpiW38D961L5qAhUM5Y/ipfs/QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The struct that stores a dataset reference is `DatasetRef`. Its fields correspond directly to the definition specified above.

```
// DatasetRef encapsulates a reference to a dataset.
type DatasetRef struct {
	// Handle of the dataset creator
	Handle string `json:"handle,omitempty"`
	// Human-friendly name for this dataset
	Name string `json:"name,omitempty"`
	// ProfileID of dataset creator
	ProfileID profile.ID `json:"profileID,omitempty"`
	// Content-addressed path for this dataset, network and version_id
	Path string `json:"path,omitempty"`
	// Dataset is a pointer to the dataset being referenced
	Dataset *dataset.DatasetPod `json:"dataset,omitempty"`
}
```

The Dataset pointer optionally points to a dataset itself, once it has been loaded into memory.

The most important function for working with `DatasetRef`s is `CanonicalizeDatasetRef`, which converts from user-friendly references and path references to canonical references.

```
func CanonicalizeDatasetRef(r Repo, ref *DatasetRef) error {
  ...
}
```

This function handles replacements like converting `me` to the user's `handle`, and fills in PeerID and Path. It returns the error `repo.ErrNotFound` if the dataset does not exist in the user's repo, which means the dataset is not local, but may exist remotely. Callers of this function should respond appropriately, by contacting peers if a p2p connection exists.


# Drawbacks
[drawbacks]: #drawbacks

Creating a naming scheme carrying with it many issues around backwards compatibility, but this is somewhat mitigated by having the network name in the reference. If, for some reason, a new distributed network needs to be supported in the future, Qri can adapt without breaking old references.

Renames are difficult to get right, since it means that code cannot assume that all hashes correlate to the same user-friendly string.

There exist subtle differences between types of dataset references, due to the structure being stateful. Code must carefully handle DatasetRefs not knowing from their type alone whether they are user-friendly, or canonical, or only a path.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Some naming scheme is absolutely necessary to refer to distributed dataset in a user-friendly way.

# Prior art
[prior-art]: #prior-art

Similar concepts exist in Git, which uses sha1 hashes the way Qri uses IPFS hashes. Git also uses branch and remote names similar to how Qri uses dataset names.

Bittorrent solves similar problems by encapsulating hash information in binary files that users load from their native interface.

# Unresolved questions
[unresolved-questions]: #unresolved-questions
