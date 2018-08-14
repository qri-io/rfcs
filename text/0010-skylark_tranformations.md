- Feature Name: <!-- (fill me in with a unique ident, my_awesome_feature) -->
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

TODO

# Motivation
[motivation]: #motivation

Qri ("query") is about datasets. Transformions are repeatable scripts for 
generating a dataset. Skylark is a scripting langauge from Google that feels a 
lot like python. This package implements skylark as a transformation syntax. 
Skylark tranformations are about as close as one can get to the full power of a 
programming language as a transformation syntax. Often you need this degree of 
control to generate a dataset.

Typical examples of a skylark transformation include:

- combining paginated calls to an API into a single dataset
- downloading unstructured structured data from the internet to extract
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

Skylark transformations have a few rules on top of skylark itself:
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

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO
