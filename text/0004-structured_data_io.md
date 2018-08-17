- Feature Name: structured_io
- Start Date: 2017-10-18
- RFC PR: [#4](https://github.com/qri-io/rfcs/pull/4)
- Repo: [dataset](https://github.com/qri-io/dataset)

_Note: This RFC was created as part of an initial sprint to adopt the RFC
process itself, as such sections of this document are less complete than
we'd hope, or less complete than we'd expect from a new RFC.
-:heart: the qri core team_

# Summary
[summary]: #summary

Structured I/O defines interfaces for reading & writing streams of parsed 
 _entries_, which are elements of dynamic-yet-structred data such as JSON, CBOR,
or CSV documents. Structured reader & writer interfaces combine a byte stream, 
data format and schema to create entry readers & writers that produce & consume 
entries of parsed primtive types instead of bytes. Structured I/O streams can be 
composed & connected to form the basis of rich data communication capable of 
spanning across formats.

# Motivation
[motivation]: #motivation

One of the prime goals of qri is to to be able to make any dataset comparable to
another dataset. Datasets are also intended to be a generic-yet-structured
document format, able to support all sorts of data with varying degrees of
quality. These requirements mean datasets must be able to define their own 
schemas, and may include data that violates that schema.

Our challenge is to declare a clear set of abstractions that leverage _key
assumptions_ enforced by the dataset model, and leverage those assumptions
to combine arbitrary data at runtime.
 
Structured I/O is the primary means of parsing data, abstracting away the
underlying data format while also delivering a set of expectations about 
parsed data based on those key assumptions. Those expectations are parsing
to a predetermined set of types (`int`, `bool`, `string`, etc.), and delivering
a _schema_ that includes a definition of valid data structuring.

As concrete examples, all of the following require tools for data parsing:
- Creating a dataset from a JSON file
- Converting dataset data from one data format to another
- Printing the first 10 rows of a dataset
- Counting the number of entries in a dataset

All of these tasks are basic things that we'd like to be able to do with one
or more datasets.

Structured I/O is intended to be a robust set of primitives that
underpin these tasks. Structured _streams_ (readers & writers) wrap a 
raw stream of bytes with a parser that tranform raw bytes into _entries_ made
of a standard set language-native types (`int`, `bool`, `string`, etc.)
Working with entries instead of bytes allows the programmer to avoid thinking
about the underlying format & focus on the semantics of data instead of
idiosyncrocies between encoding formats.

Orienting our primitives around _streams_ helps manage concerns created by both 
network latency and data volume. By orienting qri around stream programming 
we set ourselves up for success for programming in a distributed context.

Structured I/O builds on foundations set fourth in the _structure_ portion of
the dataset definition. For any valid dataset it must be possible to create
a Structured Reader of the dataset body, and a Writer that can be used to 
compose an update.

Structured I/O is intended to underpin lots of other functionality. Doing new
things with data should be a process of composing and enhancing Structured I/O 
streams. Want only a subsection of a dataset's body? use a `LimitOffsetReader`. 
Want to convert JSON to CBOR? Pipe a JSON-formatted `EntryReader` to a 
CBOR-formatted writer.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Creating a structured I/O stream requires a minimum of three things:
- a stream of raw data bytes
- the _data format_ of that stream (eg: JSON)
- a data schema

The Format & Schema are specified in the passed-in structure, the byte stream
is an io.Reader or io.Writer primitive. Here's a quick example of creating a 
reader from scratch & reading it's values:
```golang
  import (
    "strings"

    "github.com/qri-io/dataset"
    "github.com/qri-io/dataset/dsio"
    "github.com/qri-io/jsonschema"
  )

  // the data we want to stream, an array with two entries 
  const JSONData = `["foo",{"name":"bar"}]`

  st := &dataset.Structure{
    Format: dataset.JSONDataFormat,
    Schema: jsonschema.Must(`{"type":"array"}`),
  }

  // created a Structured I/O reader:
  str, err := dsio.NewEntryReader(st, strings.NewReader(JSONData))
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
  fmt.Println(ent.Value) // {"name":"bar"}

  _, err := str.ReadEntry()
  fmt.Println(err.Error()) // EOF
```

### Stream & Top Level Data
A _stream_ refers to refer to both a _reader_ and a _writer_ collectively.

_Top Level_ refers to the first entry in a discrete set of data.
This data's top level is an _array_:
```json
[
  {"a": 1},
  {"b": 2},
  {"c": 3}
]
```

This Data's top level is a _string_:
```json
"foo"
```

### Entries
Traditional "unstructured" streams often use byte arrays as the basic unit that
is both read and written. Structured I/O works with _entries_ instead
An _entry_ is the fundamental unit of reading & writing for a Structured stream.
Entries are themselves a small abstraction that carries the `Value` (parsed 
data), `Index` and `Key`. Only one of `Index` and `Key` will be populated at a
given point, depending on weather an array or object is being read.

### Value Types
Qri is built around a basic set of types, which forms a crucial assumption when
working with Structured I/O, which build on this assumption. These assumptions
are inherited from JSON, with the addition of byte arrays.

All entries will conform to one of the following types:
```golang
// Scalar Types
nil
bool
int
float64
string
[]byte

// Complex Types
[]interface{} // array
map[string]interface{} // object
```

When examining an `Entry.Value` it's type is `interface{}`, performing a 
[type switch](https://tour.golang.org/methods/16) that handles all of the above 
types will cover all possible cases for a valid entry. Using such a type switch 
recursively on complex types provides a robust, exhaustive method for inspecting
any given entry.

Its important to note that these garuntees are only enforced for basic 
Structured I/O streams. Abstractions on top of Structured I/O may introduce
additional types during processing. A classic example is timestamp parsing.
Implementers of streams that break this type assumption are encouraged to define
a more specific interface than structured I/O to indicate to consumers this
assumption has been broken.

### Corrupt Vs. Invalid Data
Structured I/O must distinguish between data that is _corrupt_ and data that is
_invalid_. Corrupt data is data that doesn't conform to the specified format.
As an example, this is corrupt json data (extra comma):
```json
["foooo",]
```
Structured I/O will error if it enounters any kind of corrupt data.

_Invalid_ data is data the doesn't conform to the specified _schema_ of
structured I/O. Structured I/O streams _are_ expected to gracefully handle
invalid data by falling back to _identity schemas_, discussed below.

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

Data who's top level does not conform to one of these two schemas is 
considered corrupt.

These "Identity Schemas" form a _fallback_ the stream will revert to if the data
it's presented with is invalid. For example, if a schema specifies a top level
of "object", and the stream encounters an array, it will silently revert to the
array identity schema & keep reading.

The rationale for such a choice is emphasizing _parsing_ over strict adherence
to schema definitions. One of the primary use cases of a dataset version control
system is to begin with data that is invalid according to a given schema, and
correct toward it.

For this reason, consumers of structured I/O streams are encouraged to 
prioritize parsing based on type switches as mentined above, unless the codepath
they are operating in presumes _strict mode_.

### Strict Mode
Fallbacks are intended to keep data reading at all costs. However many use cases
will want explicit failure when a stream is misconfigured. For this purpose
streams provide a "strict mode" that errors when invalid data is encountered,
instead of using silent identity-schema fallbacks.

When a stream operating in Strict mode encouters an entry that doesn't match, it
will return `ErrInvalidEntry` for that entry. In this case the stream will
remain safe for continued use, so that invalid entries do not prevent access
to subsequent valid reads/writes.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Entry is a "row" of a dataset:
```golang
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

EntryWriter is a generalized interface for writing structured data:
```golang
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

EntryReader is a generalized interface for reading Ordered Structured Data:
```golang
type EntryReader interface {
  // Structure gives the structure being read
  Structure() *dataset.Structure
  // ReadVal reads one row of structured data from the reader
  ReadEntry() (Entry, error)
}
```

EntryReadWriter combines EntryWriter and EntryReader behaviors:
```golang
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

Given that we've already written this, the time for considering alternatives
should be in future a RFC.

# Prior art
[prior-art]: #prior-art

[golang's io package](https://godoc.org/io) is _the_ source of inspiration here.


### OpenAPI
[OpenAPI](https://swagger.io/docs/specification/about/) Structured I/O can be 
seen as a strict extension on OpenAPI. In fact, we use the jsonschema spec that
grew out of OpenAPI!

From OpenAPI's docs:
> The ability of APIs to describe their own structure is the root of all 
awesomeness in OpenAPI. Once written, an OpenAPI specification and Swagger tools 
can drive your API development further in various ways...

Qri datasets are analogous to self-contained OpenAPI specifications & data in 
one combined document. Structured I/O is kinda like the thing that turns such a 
document back into an "API".


# Unresolved questions
[unresolved-questions]: #unresolved-questions

How do we handle data that _doesn't_ conform to the structure, such as invalid
data. Should we implement a "strict mode" that requires data to be valid?

A Structured Reader connects a single schema to a Data Stream, it's a common
use case that entries need only a the subsection of the schema that the entry