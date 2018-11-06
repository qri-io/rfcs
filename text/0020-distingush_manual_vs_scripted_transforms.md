- Feature Name: Distinguish Manual & Scripted Tranforms
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Distinguish between manual and scripted transforms to ensure reproducible dataset histories, reduce confusion and overlapping responsibilities.

# Motivation
[motivation]: #motivation


> The role of dataset.yml was a bit confusing to me because it seems to have a few responsibilities some of which are not unique to it. Two concrete examples: first, when I was writing the transformation to create me/fare_per_pax, it wasn't super clear what needed to happen in the sky file and what needed to happen in the dataset.yml. I started out with both and had to deal with a bunch of errors which stemmed from (I think) me accidentally specifying things in both places or in the wrong place. In particular I wasn't sure what runs first. For example, does the ds in the transformation know the schema specified in dataset.yml or does qri check the output of transform against the specified schema?


One of many not-so-nice surprises about the way Qri currently works comes from the passed-in dataset in a starlark transform:
```python
def transform(ds, ctx):
```
Currently, the `ds` passed into that function is 

Experienced users of git model changes 

There is no analog for a transform script in git. Git hooks are the closest concept, but only allow the user to receive notifications and interrupt the commit process, not affect the commit itself

@ramfox hit the lightbulb moment of conceiving of changes a user makes as a "human transform". 

We need to separate 


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Maintain history as an anchor point
### Separate human transforms and machine transforms into common use cases

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
