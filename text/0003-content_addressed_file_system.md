- Feature Name: content_addressed_file_system
- Start Date: 2017-08-03
- RFC PR: [#3](https://github.com/qri-io/rfcs/pull/3)
- Issue: NA

# Summary
[summary]: #summary

Content-Addressed File System (CAFS) is a generalized interface for working with 
filestores that names content based on the content itself, usually
through some sort of hashing function.
Examples of content-addressed file systems include include git, bittorrent, IPFS, 
the DAT project, etc.

# Motivation
[motivation]: #motivation

The long-term goal of CAFS is to define an interface for common filestore 
operations between different content-addressed filestores that serves the
subset of features qri needs to function.

This package doesn't aim to implement everything a given filestore can do, 
but instead focus on basic file & directory i/o. CAFS is in its very early days, 
starting with a proof of concept based on IPFS and an in-memory implementation. 
Over time we'll work to add additional stores, which will undoubtedly affect 
the overall interface definition.

A tacit goal of this interface is to manage the seam between graph-based
storage systems, and a file interface.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are two key interfaces to CAFS. The rest are built upon these two:

### File
File is an interface based largely on the `os.File` interface from golang `os` 
package, with the exception that files can be _either a file or a directory_.
This file interface will have many dependants in the qri ecosystem.

### Filestore
Filestore is the interface for storing files & directories. The "content 
addressed" part means that `Put` operations are in charge of returning the name
of the file.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The File Interface:
```golang
  // File is an interface that provides functionality for handling
  // files/directories as values that can be supplied to commands. For
  // directories, child files are accessed serially by calling `NextFile()`.
  type File interface {
    // Files implement ReadCloser, but can only be read from or closed if
    // they are not directories
    io.ReadCloser

    // FileName returns a filename associated with this file
    FileName() string

    // FullPath returns the full path used when adding this file
    FullPath() string

    // IsDirectory returns true if the File is a directory (and therefore
    // supports calling `NextFile`) and false if the File is a normal file
    // (and therefor supports calling `Read` and `Close`)
    IsDirectory() bool

    // NextFile returns the next child file available (if the File is a
    // directory). It will return (nil, io.EOF) if no more files are
    // available. If the file is a regular file (not a directory), NextFile
    // will return a non-nil error.
    NextFile() (File, error)
  }
```

The Filestore interface:
```golang

  // Filestore is an interface for working with a content-addressed file system.
  // This interface is under active development, expect it to change lots.
  // It's currently form-fitting around IPFS (ipfs.io), with far-off plans to generalize
  // toward compatibility with git (git-scm.com), then maybe other stuff, who knows.
  type Filestore interface {
    // Put places a file or a directory in the store.
    // The most notable difference from a standard file store is the store itself determines
    // the resulting key (google "content addressing" for more info ;)
    // keys returned by put must be prefixed with the PathPrefix,
    // eg. /ipfs/QmZ3KfGaSrb3cnTriJbddCzG7hwQi2j6km7Xe7hVpnsW5S
    Put(file File, pin bool) (key datastore.Key, err error)

    // Get retrieves the object `value` named by `key`.
    // Get will return ErrNotFound if the key is not mapped to a value.
    Get(key datastore.Key) (file File, err error)

    // Has returns whether the `key` is mapped to a `value`.
    // In some contexts, it may be much cheaper only to check for existence of
    // a value, rather than retrieving the value itself. (e.g. HTTP HEAD).
    // The default implementation is found in `GetBackedHas`.
    Has(key datastore.Key) (exists bool, err error)

    // Delete removes the value for given `key`.
    Delete(key datastore.Key) error

    // NewAdder allocates an Adder instance for adding files to the filestore
    // Adder gives a higher degree of control over the file adding process at the
    // cost of being harder to work with.
    // "pin" is a flag for recursively pinning this object
    // "wrap" sets whether the top level should be wrapped in a directory
    // expect this to change to something like:
    // NewAdder(opt map[string]interface{}) (Adder, error)
    NewAdder(pin, wrap bool) (Adder, error)

    // PathPrefix is a top-level identifier to distinguish between filestores,
    // for exmple: the "ipfs" in /ipfs/QmZ3KfGaSrb3cnTriJbddCzG7hwQi2j6km7Xe7hVpnsW5S
    // a Filestore implementation should always return the same prefix
    PathPrefix() string
  }
```

# Drawbacks
[drawbacks]: #drawbacks

Getting this interface right will be difficult & full of odd edge-cases that'll 
need to be handled carefully.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could skip the notion of _files_ entirely at this level, and instead choose 
to focus on _graph_ structures.

# Prior art
[prior-art]: #prior-art

There isn't much here in the way of prior art.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How do we properly handle the distinction between network & local operations?
