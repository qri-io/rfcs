- Feature Name: Distinguish Manual & Scripted Tranforms
- Start Date: 2018-11-05
- RFC PR: https://github.com/qri-io/rfcs/pull/30
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Enforce a mutually-exclusive distinction between manual and scripted transforms to ensure reproducible dataset histories, reduce confusion and remove overlapping responsibilities.

# Motivation
[motivation]: #motivation

Qri has a goal of being as simple as possible, simple can be measured in two ways: 

1. How _concise_ something is: a thing having few steps, requiring fewer instructions
2. How _clear_ something is: knowing what will happen when you interact with that thing.

Here's some feedback from @mbattifarano an early Qri user. We've cited this in [another other rfc](https://github.com/qri-io/rfcs/blob/master/text/0017-define_dataset_creation.md), but I don't think we're done mining it for cues to improve:

> The role of dataset.yml was a bit confusing to me because it seems to have a few responsibilities some of which are not unique to it. Two concrete examples: first, when I was writing the transformation to create me/fare_per_pax, it wasn't super clear what needed to happen in the sky file and what needed to happen in the dataset.yml. I started out with both and had to deal with a bunch of errors which stemmed from (I think) me accidentally specifying things in both places or in the wrong place. In particular I wasn't sure what runs first. For example, does the ds in the transformation know the schema specified in dataset.yml or does qri check the output of transform against the specified schema?

@dustmop pointed directly at the issue this feedback raised [while we were working on that rfc](https://github.com/qri-io/rfcs/pull/25#discussion_r225723031):

> The feedback also points to a possible ambiguity between transform and dataset.yml, since both can do similar things. This is a distinct (but admittedly related) problem from the semantics of `qri save`. I understand it's hard to answer the original question without cleaning up our mental model, but I think it's also important to address the original question directly.

Both are right. Words like "confusion" and "ambiguity" mean lack of clarity. Qri is emphasizing brevity too much, and needs to be more clear, even if it means adding a step or two. To do this properly, each added step should reinforce the mental model of _applying changes to a dataset_.

There's also  a technical motivation for making this change as well What's worse, this "too many things" is costing us reproducibility. Because multiple things can act on a dataset to get form one snapshot to another, we can't accuractly reproduce histories by re-playing transformations. 

## Proposal distinguish between "Manual" and "Scripted" transforms
To me, the root of the problem here is we're being concise as the cost of clarity. it's clear that Qri is doing too many things in a single step, and allowing multiple things to affect the outcome of a `qri save`.  To solve this, we'll enforce some new rules and ratchet up some definitions. I want to start by adding formality to the the term "transform":

### A "transform" is a forward transition from one dataset snapshot to another.
* transforms must mutate one or more non-computed fields of a dataset
* only one type of transform can be applied to any field per transform
* transforms can use one or more types of mutations to determine the next snapshot

A transform is the opposite of a history, moving forwards in snapshots instead of backwards. A history is _reproducible_ when you can start at the first snapshot and re-execute each mutation described in the next snapshot. By enforcing the mutually-exculsive mutations, each snapshot is a deterministic record of both state and _how to arrive at that state_ from the previous snapshot.

#### Manual Transforms
@ramfox had the lightbulb moment of conceiving of changes a user makes as a "human transform". The phrase "human transform" invites us to concieve of people manually changing things as a category of transformation, and that's exactly how we'll model it.

Manual transforms work by providing values directly to Qri, such as with a `dataset.yaml` file:
```yaml
meta:
  title: rational number series
body: 1,2,3,4,5,6,7,8,9,10]
```

And a CLI command:
```
$ qri save --file=dataset.yaml me/rational_numbers
saved dataset b5/rational_numbers
```

We can conceive of the above as a _manual transform_: a series of assignments in a single function call, with no calculations or business logic. The above example is telling Qri to do the following:

```python
def human_transform(ds):
  ds.set_meta("title", "rational number series")
  ds.set_body([1,2,3,4,5,6,7,8,9,10])
```

#### Scripted Transforms
With a scripted transform, instead of making manual mutations, an algorithm _automates_ changes to fields of a dataset with programmatic instructions:

```python
def transform(ds, ctx):
  ds.set_meta("title", "rational number series")
  # use the range function to automate setting the dataset body
  ds.set_body(range(1,11))
```

### Transform types are mutually exclusive
Both types of transform are acting on the same components and fields of a dataset (`meta` and `body` components, a `title` field and body `rows`). To ensure reproducibility, we need a new rule: **only one type of transform can mutate a field between two snapshots**. 

With this rule in place, we can finally simplify the question "what is the passed in value of `ds` in `transform(ds,ctx)`?". The answer: the previous snapshot.

In the past, this was complicated. We can now simplify the story because it's an error to have two transform types act on a single field.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Here's an example flow:

`dataset.yaml`:
```yaml
meta:
  title: Prime Ministers of Canada
  description: A list of Canadian Prime Ministers
structure:
  format: json
  schema:
    type: array
    items:
      type: object
      properties:
        name:
          type: string
body: body.json
```

Let's run that through Qri save:
```
# create a new dataset, dataset.yaml contains no transform
$ qri save --file=dataset.yaml me/ca_prime_ministers
created new dataset b5/ca_prime_ministers
```

Pretty quickly we realize that manually constructing the body is a pain, so we write a transform script that grabs this data from a trusted source:

`transform.star`:
```python
load("http.star", "http")

def download(ctx):
  res = http.get("http://canada.ca/prime-ministers-list.json")
  return res.json()

def transform(ds, ctx):
  ds.set_body(ctx.download)
```

So we update our dataset.yaml to specify the transform script:

`dataset.yaml`:
```yaml
meta:
  title: Prime Ministers of Canada
  description: A list of Canadian Prime Ministers
structure:
  format: json
  schema:
    type: array
    items:
      type: object
      properties:
        name:
          type: string
transform:
  script: transform.star
body: body.json
```

But re-running save gives us an error:
```shell
$ qri save --file=dataset.yaml me/ca_prime_ministers
error: transform script and user-supplied datsaet are both trying to set:
  body

please adjust either the transform script or remove the supplied body
```

So we do what the error tells us, and remove the `body` field from dataset.yaml, and re-save. This time it works, and the transform runs:

```shell
$ qri save --file=dataset.yaml me/ca_prime_ministers
🤖 executing transform
✅ transform complete
saved dataset b5/ca_prime_ministers
```

Some time later we want to get fresh data, so we run an update:

```shell
$ qri update me/ca_prime_ministers
🤖 executing transform
✅ transform complete
updated dataset b5/ca_prime_minsters
```

Dope. Now we realize that it's important to add themes to our metadata, to classify this info as being about government. In this case we're only trying to set meta, not get a new version of the data. So this time we trust that the data we've already specified is in qri, so we can delete all the stuff about structure in `dataset.yaml`, and just focus on the meta component:

`dataset.yaml`:
```yaml
meta:
  title: Prime Ministers of Canada
  description: A list of Canadian Prime Ministers
  theme:
  - govenment
  keywords:
  - canada
  - government
  - prime ministers
```

And we save the changes:

```shell
$ qri save --file=dataset.yaml me/ca_prime_ministers
saved dataset b5/ca_prime_minsters
```

The new part here is the transform didn't run, because _save only runs transforms the first time they're provided_. Further proof of this comes from the fact that the transform is now missing from the most recent snapshot:

```
$ qri get transform me/ca_prime_ministers
null
```

Also, `qri update` now gives us a new error:

```
$ qri update me/ca_prime_ministers
error: no transform script in most recent dataset. There is a transform script 2 commits back that adjusts:
  body

to run an update using the most recent transform, run:
  qri update --recall-tf me/ca_prime_ministers
```

This missing transform is vital for reproducibility reasons, the missing transform indicates that no tranform script was executed to get from the snapshot that had the old meta to the new meta.

Resurrecting the transform is relatively easy, we follow the instructions from the error:
```
$ qri update --recall-tf me/ca_prime_ministers
🤖 executing transform
✅ transform complete
updated dataset b5/ca_prime_minsters
```

Finally, if we want to both re-run the existing transform _and_ change things about a dataset in a single commit, we can still do that so long as the transform doesn't affect any fields we're trying to manually change by running `qri save with --recall-tf`. In this example we'll adjust the schema to add more specificity, so we adjust dataset.yaml:

`dataset.yaml`:
```yaml
structure:
  format: json
  schema:
    type: array
    items:
      type: object
      properties:
        name:
          type: string
        description:
          type: string
          maxLength: 140
```

And run save with `--recall-tf`:

```
$ qri save --file=dataset.yaml --recall-tf me/ca_prime_ministers
🤖 executing transform
✅ transform complete
updated dataset b5/ca_prime_minsters
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Human and Machine Transforms that affect the same field is now an error

While this RFC places the definition at the field level, our initial implemenation will operate at the component level for simplicity.

This does mean we'll need tools to check which components & fields of a dataset a transform affects. Thanks to the way starlark works (and our implementation), this'll be pretty simple.

Ways to resolve the error:
* adjust your commit so that it doesn't affect
* `--drop-transform` will remove the transform moving forward

We'll use the phrase _control_ to distinguish who's doing what. Errors should read `body is controlled by a transform script. Avoid adding a body with `

## The job of a transform script is to generate the next state snapshot from the existing snapshot
This means the dataset passed into `transform(ds,ctx)` will be the existing dataset in Qri. Because human and machine transforms are mutually exclusive, script authors only need to consider which fields they wish to remain editable by users.


## `save` is for humans, `update` is for machines
Save is now the more powerful of the two commands. 

By default save will _not_ recall and re-run transforms, but will run provided transforms. running `qri save` without providing a tra

Local updates will need some refinement. Running `qri update me/dataset` will now only succeed if a 
Update gets a new flag: `--recall-last`

This is a little light, but at a bare minimum we'll need to do the following:
* add 

# Drawbacks
[drawbacks]: #drawbacks

The main reason for not doing this is we haven't vetted this approach well enough.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The alternative is what we're currently doing, which is a sort of simultaneous three-way merge between the previous commit, user input, and script results. A more realistic alternative would be further refinements on this model, which we'll hopefully figure out through use.

# Prior art
[prior-art]: #prior-art

[Git Hooks](https://githooks.com/) were mentioned as an example of prior art. There isn't much to go on here.
There is no analog for a transform script in git. Git hooks are the closest concept, but only allow the user to receive notifications and interrupt the commit process, not affect the commit itself

# Unresolved questions
[unresolved-questions]: #unresolved-questions

There's some unfinished work I'd like to do in this area, which is document the advantages that come out of this:
### Maintain history as an anchor point
This rfc refines our mental model, centering it around state transitions, which is what's happening in the best of times when using git. I'd like to write more about this.

### Separate human transforms and machine transforms into common use cases