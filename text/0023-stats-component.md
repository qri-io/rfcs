- Feature Name: Stats Component
- Start Date: 03-22-2019
- RFC PR: 
- Issue:

# Summary
[summary]: #summary

Introducing a new component to our dataset model: the Stats component. The Stats component contains some statistical analysis on the dataset body. Qri can create a default stats component, if the dataset body has certain qualities (right now, if it is tabular). This rfc posits the first default stats that should be added to the component, the "standard" way the stats component should be shaped (including any keywords that should be reserved), and finally, it maps out a future in which a dataset creator can add their own custom stats to the stats component.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->
Qri is missing statistical metadata! We have descriptive metadata (stored in the Meta component), structural metadata (Structure component), and administrative metadata (Commit component), but no statisicial metadata. This rfc proposes the the path forward to statistical metadata stored in the Stats component. 

Creating and storing a Stats component will take a sliver of the computing power it takes to create and store the dataset itself (as this rfc will prove). For that small cost, the stats component will make the dataset creators life easier. It will be a quick way to learn about the full dataset body, and it will allow us to have visualizations without needing to inject the full body into the html. That not only reduces loading time for templates, but will make our dataset packages slimmer.

The stats component is one part in a push to get dataset creators to an interesting, publishable insight quicker. The ideal path, using the default Stats generator and the default template would be: the creator adds a dataset with a tabular body. On dataset creation, Qri calculates a stats component, and uses this stats component to render a visualization from the default template. The creator can check out that template (or the stats component itself), to see if there are any interesting insights. The creator may choose to publish this dataset then and there, exposing the dataset and the rendered findings to the world, or create a new version of the dataset exploring the findings further.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

concepts to expand on:
- example stats component

```
Stats: [
  {
    "title":"",
    "type":"number" // or decimal/integer? need to double check how this is stated in jsons schema
    "count":,
    "min":,
    "max":,
    "avg":,
  },
  {
    "title":"",
    "type":"string",
    "count":,
    "unique":, // can't cause not rolling
    "most_common":, // can't cause not rolling
    "minLength",
    "maxLength"
  },
  {
    "title":"",
    "type":"boolean",
    "count":,
    "trueCount":,
    "falseCount":,
  }
]
```
Can calculate stats on boolean, string, and numerical columns. 
For each type, qri can calculate, on default, a number of stats. Our main qualifications for adding a stat to the qri default, are that it can be expressed with a small amount of memory and be calculated on a rolling basis.

There are few specific things we are looking for for each type. The main qualifications are that it not be a large piece of data to save, and something that can be calculated on a rolling basis.

To start, we need to constrain the datasets onto which we calculate stats. For this first implimentation, we will only create default stats on two dimentional (tabular) data.

Not only does 3 dimentinal (or higher) data have a less apparent structure, it is also diffcult to reason about the best way to display stats for 3 dimentional data.
For example, with geojson:
- multiple dimensions, how do you describe which section is being used
- there is no guarentee that row the dataset has the same elements/patterns
- perhaps we can colapse columns e.g. for a coordiate in geojson:
[
  {
    "type": "Feature",
    "geometry": {
      "type": "Point",
      "coordinates": [125.6, 10.1]
    },
    "properties": {
      "name": "Dinagat Islands"
    }
  },
  {
    "type": "Feature",
    "geometry": {
      "type": "Point",
      "coordinates": [125.6, 10.1]
    },
    "properties": {
      "name": "Dinagat Islands"
    }
  },
  {
    "type": "Feature",
    "geometry": {
      "type": "Point",
      "coordinates": [125.6, 10.1]
    },
    "properties": {
      "name": "Dinagat Islands"
    }
  },
]
stats: [
  {
    "title":"type",
    "type":"string",
    "count": 3,
    "minLength: 7,
    "maxLength": 7
  },
  {
    "title":"geometry.type",
    "type":"string",
    "count": 3,
    "minLength: 5,
    "maxLength": 5
  },
  {
    "title":"geometry.coordinates.x",
    "type":"float",
    "count": 3,
    "min: 125.6,
    "max": 125.6,
    "avg": 125.6
  },
  {
    "title":"geometry.coordinates.y",
    "type":"float",
    "count": 3,
    "min: 10.1,
    "max": 10.1,
    "avg": 10.1
  },
  {
    "title":"properties.name",
    "type":"string",
    "count": 3,
    "minLength: 15,
    "maxLength": 15
  }
]

Do stats have to be an array, or should they be a dictionary, with the keys as the column names?
{
  "type": {
    "type":"string",
    "count": 3,
    "minLength: 7,
    "maxLength": 7
  },
  "geometry.type": {
    "type":"string",
    "count": 3,
    "minLength: 5,
    "maxLength": 5
  },
  "geometry.coordinates.y": {
    "type":"float",
    "count": 3,
    "min: 10.1,
    "max": 10.1,
    "avg": 10.1
  },
  "properties.name": {
    "type":"string",
    "count": 3,
    "minLength: 15,
    "maxLength": 15
  }
}

This could potentially no preserve order, which may be important. If we present as an array, order is maintained.
Need specific circumstances and formatting for default stats. Once a stats component is present, we want to be able to assume a lot about it, especially for use in our default template. If custom stats are added, we need to make it clear to the dataset creator that they should create a custom template as well.

We want the users to have abject flexibility to create stats, but the default should only be created under specific circumstances.
- what are the default stats
- explain why we are choosing those specific default stats.
- talk about needed to assume very little but allow a lot

<!-- Explain the proposal as if it was already included in the language and you were teaching it to a Qri _developer_. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Qri developer should *think* about the feature, and how it should impact the way they use Qri. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to a Qri developer vs a Qri _user_.

For implementation-oriented RFCs (e.g. for Qri codebase internals), this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->
- out of the scope of this rfc, but explain a bit about the idea for custom stats, how they might be added to qri, and 
- show potential starlark example of set_stats
- how would someone create those stats?
- set_stats and then actually add some custom things that we might want, ie, in an options or a config say each row add average, each row add median, for this specific row add median of text eg

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation\

- implimented at the dsio level
- adding a section to Dataset
- basically walking through what b5 has already added

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?
- we already have a meta that we can add arbitrary data too
- we already have a schema that talks about the structure of the data, and is going to look really similar
- potential dataset bloat?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
- the impact of not doing this => huge template bloat. So sliver of added bytes to the dataset, and template can get slimmer, cause you don't have to do calculations in the template.
- alternatives, use json schema somehow for stats... we are basically using json
schema to describe the stats anyway. 
- keep pushing the stats to a stats field in meta. However, since stats should be computer generated, this doesn't seem right. It's to easy right now to mess with the meta.
<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art
- json schema
- need to look at kaggle and other dataset creation tools to see what they use for default stats 
- on kaggle => 
for number => valid, mismatched, missing, the mean, standard deviation, and quantities for each percentile
for string => valid, mismatched, missing, # of unique entries, most common entries (and %)
date => valid, mismatched, missing, minimum, mean, maximum
boolean => valid, mismatched, missing, unique, most common
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
- where we go from here re: stats component and custom stats.
<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
