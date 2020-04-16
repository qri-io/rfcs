- Feature Name: formalizing qri data types
- Start Date: 2019-08-02
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Formalize the data types we deserialize to and from for all data formats we support. Introduce complex types as an optional superset of types that blend behaviours with data.

# Motivation
[motivation]: #motivation

I'd like to take some time to return to first principles, formally defining the data types we operate upon. By defining a universe of expected types we can constrain the surface area of our work.

With these basic types formalized, this RFC introduces

Future iterations of Qri



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Define a new package `github.com/qri-io/vals`

We've had time to build up a baseline set of data formats we'd like to support and interoperate with. We've settled on these four as a starting point:

* JSON
* CSV
* XLSX
* CBOR

For some time we've been using JSON's self-describing types as a 


### Standard Data Types

Standard data types have no behaviour beyond a programming language

| #  | type           | example            | description  |
| -- | -------------- | ------------------ | ------------ |
| 1  | null           | `null`             | the explicit zero-value to signify non-existence |
| 2  | boolean        | `false`            | the set of boolean values: true and false |
| 3  | byte           | `0x00`             | the set of all unsigned 8-bit integers. Range: 0 through 255. |
| 4  | int            | `23094`            | the set of all 64-bit whole number value ranging from -/+ 9,223,372,036,854,775,807 |
| 5  | number         | `25.234`           | the set of all [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) 64-bit floating-point numbers |
| 6  | bytes          | `0xEF8`            | the set of all byte strings |
| 7  | string         | `"hai"`            | the set of all strings of 8-bit bytes, conventionally but not necessarily representing UTF-8-encoded text |
| 8  | array          | `[3,false]`        | the set of all ordered lists containing any of the other values |
| 9  | map            | `{"foo": false}`   | the set of all  |

Standard data types come in two forms:

* _scalar_ types: types which have no
* _compound_ types: types that are composed of other types

### Complex Types

In addition to standard types, this RFC defines a set of _complex_ types.

complex types are optional, it is the duty of implementations to document weather or not their package accepts & works with complex types

| #  | type           | example            | description  |
| -- | -------------- | ------------------ | ------------ |
| 10 | link           | `"http://qri.io"`  | a string value that can be _resolved_, yeilding one of the other values |
| 11 | ByteReader     | `0xEF895670990`    | a stream of bytes |
| 12 | ValueReader    | `[3,4,2,false,{}]` | |
| 13 | mapValueStream | `{"a":3,"next":4}` |  |

### Values

### Default Values & Zero Values

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


| type      | example  | go type   | description  |
| --------- | -------- | --------- | ------------ |
| null      | `null`   | `nil`     | |
| boolean   | `false`  | `bool`    | |
| byte      | `0x0`    | `byte`    | |
| int       | `23094`  | `int`     | |
| number    | `25.234` | `float64` | |
| string    | `"hai"`  | `string`  | |
| bytes     | `0xEF8`  | `[]byte`  | |
| type      | example          | go type         | description  |
| --------- | ---------------- | --------------- | ------------ |
| array     | `[3,false]`      | `[]interface{}` | |
| map       | `{"foo": false}` | `map[string]interface{}` / `map[interface{}]interface{}` | |

In addition to the `array` and `map` types, the vals package provides two interfaces that can be used 

## Complex Types

| type           | example            | go type               | description  |
| -------------- | ------------------ | --------------------- | ------------ |
| link           | `http://qri.io`    | `vals.Link`           | |
| byteReader     | `0xEF895670990`    | `vals.ByteStream`     | |
| byteWriter     | `0xEF895670990`    | `vals.ByteStream`     | |
| valueReader    | `[3,4,2,false,{}]` | `vals.ValueStream`    | |
| mapValueStream | `{"a":3,"next":4}` | `vals.MapValueStream` | |

```go
type ByteStream interface {
  
}
```

```go
```

### Type Switching

### Regarding `map[interface{}]interface{}`


### Link Resolution

### ValueStream

```go
type ValueReader interface {
  Read() (value interface{}, err error)
  ReadKey() (key string, value interface{}, err error)
  IsMap() bool
  IsOrdered() bool
  Close() error
}
```


```go
type ValueWriter interface {
  Write(value string) (err error)
  WriteKey(key string, value interface{}) (err error)
  IsMap() bool
  IsOrdered() bool
  Close() error
}
```


# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Time value
This go-round we've opted to not include a time value, and instead reserve it for future expansion. At present datetimes should be stored as either strings or seconds-since-epoch by convention.

### Iterators

```go
type Iterator interface {
  Next(val *interface{}) (more bool)
}
```

The problem with iterators comes with error handling. Often valueStreams are

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
