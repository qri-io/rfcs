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

Let's split this section in to three parts. 
First we will talk about what a default stats component might looks like and what stats we want to generate. Then let's talk about future possibilities, namely what a default stats component might look like for high dimentional data and what custom stats might look like and what we need to do to get there.

## Default Stats component:
To start, Qri can create a default stats component on tabular (two dimentional) data. Because we want the cost of creating a stats component to be low, we only want to generate stats that can be calculated on a rolling basis.

The different types of data that we can generate stats on are number (including integer and floating point), string, and boolean.

We can generate different stats depending on the type of that column.

Number:
  valid - the number of cells in that column that contain valid numbers
  error - the number of cells in that column that contain invalid numbers
  missing - the number of cells in that column that do not contain data
  min - the minimum number found
  max - the maximum number found
  avg - the average value

String
  valid - the number of cells in that column that contain valid strings
  error - the number of cells in that column that contain invalid string
  missing - the number of cells in that column that do not contain data 
  minLength - the shortest length of a string in that column
  maxLength - the longest length of a string in that column

Boolean
  valid - the number of cells in that column that contain valid boolean values
  error - the number of cells in that column that contain invalid boolean values
  missing - the number of cells in that column that do not contain data 
  trueCount - the number of cells that contain a true value
  falseCount - the number of cells that contain a false value

The structure of the default stats component depends on the structure of the data itself. If each row of the dataset body is expressed as an array, the stats component will be an array of objects. If each row of the dataset body is expressed as an object, the stats component will be an object of objects. One important thing to note, we must have a structure with a schema that describes the title and the type in order to create stats.

Let's look at two examples of a small dataset body, and the resulting stats object.

```json
// this is a small dataset body expressed as a 2D array. It is a list of 
// classes in an elementary school. The columns are the class name (string),
// the current number of students (number), and whether the class can accept 
// more students
[
  ["1A", 20, false],
  ["1B", 22, false],
  ["1C", 17, true],
  ["1D", 19, true]
]

// the resulting stats section would look like this:
[
  {
    "title":"class name",
    "type":"string",
    "count": 4,
    "minLength": 2,
    "maxLength": 2
  },
  {
    "title":"students",
    "type":"number",
    "count": 4,
    "min": 17,
    "max": 22,
    "avg": 19.5,    
  },
  {
    "title":"can accept more students",
    "type":"boolean",
    "count": 4,
    "trueCount": 2,
    "falseCount": 2,
  }
]
```
Note that the data we expressed as a 2D array, and so the stats component is presented as an array.

Let's look at this same example, but with the data presented as an array of objects:

```json
[
  {
    "class_name":"1A",
    "students":20, 
    "can_accept_more":false
  },
  {
    "class_name":"1B",
    "students":22, 
    "can_accept_more":false
  },
  {
    "class_name":"1C",
    "students":27, 
    "can_accept_more":true
  },
  {
    "class_name":"1D",
    "students":19, 
    "can_accept_more_students":true
  }
]

// stats:
{
  "class_name": {
    "title":"class name",
    "type":"string",
    "count": 4,
    "minLength": 2,
    "maxLength": 2
  },
  "students": {
    "title":"students",
    "type":"number",
    "count": 4,
    "min": 17,
    "max": 22,
    "avg": 19.5,    
  },
  "can_accept_more_students": {
    "title":"can accept more students",
    "type":"boolean",
    "count": 4,
    "trueCount": 2,
    "falseCount": 2,
  }
}
```

In summary, qri can generate some basic stats on a dataset body, if the body is tabular. We also need a structure with a schema in order to understand what stats to be calculating.

## high dimentional data
Not only does 3 dimentinal (or higher) data have a less apparent structure, it is also diffcult to reason about the best way to display stats for 3 dimentional data.
The issue with high dimentional data, is there is no easy way to reason about the structure. With 2 dimentional data, if there are the same number of elements in each row, we know that the structure is consistent, so we have some assurance that when we calculate stats, the stats will make sense.
However, with higher dimentional data, it is harder to make those guarentees. 

However, if some data has a top level array and comes with a well defined schema, we then can at least know that the intention is for each row to have the same structured content. We can perhaps do stats on that data.

One example of data that has a defined structure, and has a top level array is geojson data. Let's look at a small array of geojson data:
```json

```
- multiple dimensions, how do you describe which section is being used
- there is no guarentee that row the dataset has the same elements/patterns
- perhaps we can colapse columns e.g. for a coordiate in geojson:

```json
[
    {
      "type": "Feature",
      "properties": {
        "name":"Brooklyn"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -74.00,
          40.63
        ]
      }
    },
    {
      "type": "Feature",
      "properties": {
        "name":"Manhattan"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -74.00,
          40.71
        ]
      }
    },
    {
      "type": "Feature",
      "properties": {
        "name":"Queens"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -73.90,
          40.71
        ]
      }
    },
    {
      "type": "Feature",
      "properties": {
        "name":"Staten Island"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -74.11,
          40.62
        ]
      }
    },
    {
      "type": "Feature",
      "properties": {
        "name":"Bronx" 
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          -73.92,
          40.82
        ]
      }
    }
  ]

// the json schema:
{
  "definitions": {},
  "type": "array",
  "items": {
    "required": [
      "type",
      "properties",
      "geometry"
    ],
    "properties": {
      "type": {
        "type": "string",
      },
      "properties": {
        "type": "object",
        "required": [
          "name"
        ],
        "properties": {
          "name": {
            "type": "string",
            "default": "",
            "examples": [
              "Brooklyn"
            ]
          }
        }
      },
      "geometry": {
        "type": "object",
        "required": [
          "type",
          "coordinates"
        ],
        "properties": {
          "type": {
            "type": "string",
            "examples": [
              "Point"
            ],
          },
          "coordinates": {
            "type": "tuple",
            "items": [
              {"type": "number", "title":"x"},
              {"type": "number", "title":"y"}
            ],
            "additionalItems": false
          }
        }
      }
    }
  }
}
// we can use this schema to infer the shape of each row, and determine a stats object
// based on that:
[
  {
    "title":"type",
    "type":"string",
    "count": 5,
    "minLength": 7,
    "maxLength": 7
  },
  {
    "title":"geometry.type",
    "type":"string",
    "count": 5,
    "minLength": 5,
    "maxLength": 5
  },
  {
    "title":"geometry.coordinates.x",
    "type":"float",
    "count": 5,
    "min": -74.11,
    "max": -73.90,
    "avg": -73.99
  },
  {
    "title":"geometry.coordinates.y",
    "type":"float",
    "count": 5,
    "min": 40.62,
    "max": 40.82,
    "avg": 40.70
  },
  {
    "title":"properties.name",
    "type":"string",
    "count": 5,
    "minLength": 5,
    "maxLength": 13
  }
] 
```

There maybe a few standards on which we can adopt default schemas, stats, and templates, that would play well together on dataset creation. GeoJSON would certainly be one candidate.

## Custom stats



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
