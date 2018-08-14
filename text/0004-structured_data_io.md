- Feature Name: structured_io
- Start Date: 2017-10-18
- RFC PR: [#4](https://github.com/qri-io/rfcs/pull/4)
- Issue: N/A
- Repo: [dataset](https://github.com/qri-io/dataset)

# Summary
[summary]: #summary

Structured I/O defines interfaces for reading & writing streams of _entries_,
which are elements of a composite data structure such as an array or object.
Structured reader & writer interfaces combine a byte stream, data format 
and schema to create entry readers & writers that produce & consume entries
instead of bytes. Structured I/O streams can be composed & connected to form
the basis of rich data communication capable of spanning across formats.

# Motivation
[motivation]: #motivation

Structured I/O is the primary means of _doing things with data_. All of the
following require tools for semantic interpretation of data:
- Creating a dataset from a JSON file
- Converting dataset data
- Printing the first 10 rows of a dataset
- Counting the number of entries in a dataset

All of these tasks are basic things that we'd like to be able to do with a
dataset. Structured I/O is intended to be a robust set of primitives for
performing these tasks. Structured _streams_ (readers & writers) abstract away 
the data formats, automatically parsing raw bytes into parsed _entries_ of data.
Working with entries instead of bytes allows the programmer to avoid thinking
about the underlying format & focus on the semantics.

Orienting our primitives around _streams_ helps manage concerns created by both 
network latency and data volume. Stream-based programming reduces the

Structured I/O builds on foundations set fourth in the _structure_ portion of
the dataset definition. For any valid dataset it must be possible to create
a Structured Reader of the dataset body, and a Writer that can be used to 
compose an update.

"Doing things with data" should be a process of composing Structured streams.
Want only a subsection of a dataset's body? use a `LimitOffsetReader`. Want
to convert JSON to CBOR? Pipe a JSON-formatted `EntryReader` to a CBOR-formatted 
writer.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

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
    Schema: jsonschema.Must(`{"type":"array"}`),
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

  _, err := str.ReadEntry()
  fmt.Println(err.Error()) // EOF
```

Creating a structured I/O stream requires a minimum of three things:
- a stream of raw data bytes
- the _data format_ of that stream (eg: JSON)
- a data schema

The Format & Schema are specified in the passed-in structure, the byte stream
is an io.Reader or io.Writer primitive

### Stream
In this document we'll use _stream_ to refer to refer to both a _reader_ and a
_writer_ collectively.

### Entries
Traditional "unstructured" streams often use byte arrays as the basic unit that
is both read and written. Structured I/O works with _entries_ instead
An _entry_ is the fundamental unit of reading & writing for a Structured stream.
Entries are themselves a small abstraction that carries the `Value` (parsed 
data), `Index` and `Key`. Only one of `Index` and `Key` will be populated at a
given point, depending on weather an array or object is being read.

### Identity Schemas & Fallbacks
Because schemas _must_ be defined on complex types, and the only complex types 
we support are objects and arrays, there are two "identity" schemas that 
represent the minimum possible schema definitions that specify the top level of
a data stream be either an array or an object:

**Array Identity Schema**
```json
{ "type" : "array" }
```

**Object Identity Schema**
```json
{ "type" : "object" }
```

Data that does not conform to one of these two schemas is considered corrupt.

These "Identity Schemas" form a _fallback_ the stream can revert to if the data
it's presented with is invalid. For example, if a schema specifies a top level
of "object", and the stream encounters an array, it will silently revert to the
array identity schema & keep reading.

### Strict Mode
Fallbacks are intended to keep data reading at all costs. However many use cases
will want explicit failure when a stream is misconfigured. For this purpose
streams should provide a "strict mode" that errors when invalid data is
encountered instead of using silent fallbacks.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


```golang
// Entry is a "row" of a dataset
type Entry struct {
  // Index represents this entry's numeric position in a dataset
  // this index may not necessarily refer to the overall position within 
  // the dataset as things like offsets affect where the index begins
  Index int
  // Key is a string key for this entry
  // only present when the top level structure is a map
  Key string
  // Value is information contained within the row
  Value interface{}
}
```

```golang
// EntryWriter is a generalized interface for writing structured data
type EntryWriter interface {
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
// EntryReader is a generalized interface for reading Ordered Structured Data
type EntryReader interface {
  // Structure gives the structure being read
  Structure() *dataset.Structure
  // ReadVal reads one row of structured data from the reader
  ReadEntry() (Entry, error)
}
```

```golang
// EntryReadWriter combines EntryWriter and EntryReader behaviors
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

The drawbacks of this is it's a new interface that needs to be built, 
rationalized & maintained, which is to say this is work, and we should avoid
doing work when we could instead be doing something else.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Given that we

# Prior art
[prior-art]: #prior-art

[golang's io package](https://godoc.org/io) is _the_ source of inspiration here.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

How do we handle data that _doesn't_ conform to the structure, such as invalid
data. Should we implement a "strict mode" that requires data to be valid?

A Structured Reader connects a single schema to a Data Stream, it's a common
use case that entries need only a the subsection of the schema that the entry