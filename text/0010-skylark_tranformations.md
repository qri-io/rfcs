- Feature Name: skylark_transformations
- Start Date: 2018-05-23
- RFC PR: [#12](https://github.com/qri-io/rfcs)
- Repo: https://github.com/qri-io/skytf

# Summary
[summary]: #summary

Skylark transformations implement a turing-_incomplete_ scripting language to
automate the generating a dataset document. Skylark transformations include 
tools for creating documents from things like a stream of input data, 
HTTP resources, or other qri datasets. Transformation scripts are embedded into 
the dataset document, allowing datasets to "self update" by re-executing the
script & adding a snapshot to a dataset's version history.

# Motivation
[motivation]: #motivation

Qri ("query") is about datasets. Transformations are repeatable scripts for 
generating a dataset. Skylark is a scripting langauge from Google that feels a 
lot like python.
Skylark tranformations are about as close as one can get to the full power of a 
programming language as a transformation syntax. Often you need this degree of 
control to generate a dataset.

Typical examples of a skylark transformation include:

- combining paginated calls to an API into a single dataset
- downloading unstructured or structured data from the internet to extract
- re-shaping raw input data before saving a dataset

We're excited about skylark for a few reasons:

- python syntax - many people working in data science these days write python, 
we like that, skylark likes that. dope.
- deterministic subset of python - unlike python, skylark removes properties 
that reduce introspection into code behaviour. things like while loops and 
recursive functions are ommitted, making it possible for qri to infer how a 
given transformation will behave.
- parallel execution - thanks to this deterministic requirement (and lack of 
global interpreter lock) skylark functions can be executed in parallel. Combined
with peer-2-peer networking, we're hoping to advance tranformations toward
peer-driven distribed computing. More on that in the coming months.

One of the primary concerns with running arbitrary code is ensuring it's
properly sandboxed against malicious intent. Chosing a turing-incomplete
scripting syntax sets us on a path that allows code analysis

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


### Data Functions
Data Functions are the core of a skylark transform script. Here's an example of 
a simple data function that returns a constant result:

```python
def transform(qri):
  return ["hello","world"]
```

Here's something slightly more complicated that modifies a previous dataset by 
adding up the length of all of the elements:

```python
def transform(qri):
  body = qri.get_body()
  count = 0
  for entry in body:
    count += len(entry)
  return [{"total": count}]
```

The `transform` function is analagous to a `main` function in other languages.
It's the final function that is called by the host environment. Also like 
languages that support zero or more `init` functions, there are other data
functions. All data functions have a few things in common:
- Data functions *always* return an array or dictionary/object, representing 
the new dataset body
- When you define a data function, qri calls it for you
- All transform functions are optional (you don't _need_ to define them), _but_
- A transformation must have at least one data function
- Data functions are always called in the same order
- Data functions often get a `qri` parameter that lets them do special things

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

It is crucial to strike a balance between utility & feature-bloat when it comes
to the API & module design of skylark transformations. There is a natural 
temptation here to reinvent the entire world of data processing inside of
skylark, which would be a mistake if not cared for properly.

Indeed one of the prime motivators for the RFC process in the first place is to
enact a mechanism for carefully deciding the design of the skylark syntax.

We should aim for a standard suite of skylark modules that 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The other major alternative here is web assembly (WSM.js), which is popular in
the cryptocurrency space. We've made provisions for qri datasets to support
both of these syntaxes, but given our committment to data science, a python-like
syntax is a clear winner here.

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO
