- Feature Name: structured_data_io
- Start Date: 2017-10-18
- RFC PR: [#4](https://github.com/qri-io/rfcs/pull/4)
- Issue: NA

# Summary
[summary]: #summary

Structured data io defines interfaces for reading & writing streams of data 
that have a defined structure.

# Motivation
[motivation]: #motivation

Defining primitives that introduce structure allows us to quantify datasets and
dataset components as _streams with structure_.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Structured reader relies on the _structure_ field of a dataset.


Here's a quick example of creating a reader from scratch & reading it's values:
```golang
  import (
    "strings"

    "github.com/qri-io/dataset"
    "github.com/qri-io/dataset/dsio"
    "github.com/qri-io/jsonschema"
  )

  st := &dataset.Structure{
    Format: dataset.JSONDataFormat,
    Schema: jsonschema.Must(`{ "type": "array"}`),
  }
  str, err := dsio.NewEntryReader(st, strings.NewReader(`["foo","bar"]`))
  if err != nil {
    panic(err)
  }

  ent, err := str.ReadEntry()
  if err != nil {
    panic(err)
  }
  fmt.Println(ent.Value) // "foo"

  ent, err := str.ReadEntry()
  if err != nil {
    panic(err)
  }
  fmt.Println(ent.Value) // "bar"

  ent, err := str.ReadEntry()
  if err != nil {
    panic(err) // panic: EOF
  }
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

```golang
// Entry is a "row" of a dataset
type Entry struct {
  // Index represents this entry's numeric position in a dataset
  // this index may not necessarily refer to the overall position within the dataset
  // as things like offsets affect where the index begins
  Index int
  // Key is a string key for this entry
  // only present when the top level structure is a map
  Key string
  // Value is information contained within the row
  Value interface{}
}
```

```golang
// StructuredWriter is a generalized interface for writing structured data
type StructuredWriter interface {
  // Structure gives the structure being written
  Structure() *dataset.Structure
  // WriteEntry writes one "row" of structured data to the Writer
  WriteEntry(Entry) error
  // Close finalizes the writer, indicating all entries
  // have been written
  Close() error
}
```

```golang
// StructuredReader is a generalized interface for reading Ordered Structured Data
type StructuredReader interface {
  // Structure gives the structure being read
  Structure() *dataset.Structure
  // ReadVal reads one row of structured data from the reader
  ReadEntry() (Entry, error)
}
```

```golang
// EntryReadWriter combines StructuredWriter and StructuredReader behaviors
type EntryReadWriter interface {
  // Structure gives the structure being read and written
  Structure() *dataset.Structure
  // ReadVal reads one row of structured data from the reader
  ReadEntry() (Entry, error)
  // WriteEntry writes one row of structured data to the ReadWriter
  WriteEntry(Entry) error
  // Close finalizes the ReadWriter, indicating all entries
  // have been written
  Close() error
  // Bytes gives the raw contents of the ReadWriter
  Bytes() []byte
}
```

# Drawbacks
[drawbacks]: #drawbacks

The drawbacks of this is it's a new interface that needs to be build, 
rationalized & maintained, which is to say this is work, and we should avoid
doing work when we could instead be doing something else.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

TODO 

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How do we handle data that _doesn't_ conform to the structure? Strict Mode?
