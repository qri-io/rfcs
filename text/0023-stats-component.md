- Feature Name: Stats Component
- Start Date: 03-22-2019
- RFC PR: 
- Issue:

# Summary
[summary]: #summary

Define "Stats", an optional dataset component that contains statistical metadata about the body of a dataset.


# Motivation
[motivation]: #motivation

Qri is missing statistical metadata! Of the [five common types](https://en.wikipedia.org/wiki/Metadata) of metadata, Qri's dataset definition already has four:

descriptive metadata (stored in the Meta component)
structural & reference metadata (Structure component)
administrative metadata (Commit component)
This rfc proposes adding a new component to dataset for storing statistical metadata.

Because Qri strives to be a generic dataset management tool, it's difficult to infer a set of meaningful statistics without introducing subject-matter specific information. Instead we focus on conventions for structuring statistical metadata, providing users flexibility to define stats metadata that suit their needs. We only default to inferring stats in common scenarios that introduce minimal cost overhead, like tabular data.

The stats component is one part in a push to get dataset creators to an interesting, publishable insight quicker. The ideal path, using the default Stats generator and the default template would be: the creator adds a dataset with a tabular body. On dataset creation, Qri calculates a stats component, and uses this stats component to render a visualization from the default template. The creator can check out that template (or the stats component itself), to see if there are any interesting insights. The creator may choose to publish this dataset then and there, exposing the dataset and the rendered findings to the world, or create a new version of the dataset exploring the findings further.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Let's split this section in to three parts. 
First we will talk about what a default stats component might looks like and what stats we want to generate. Then let's talk about future possibilities, namely what a default stats component might look like for high dimentional data and what custom stats might look like and what we need to do to get there.

## Default Stats component:
To start, Qri can create a default stats component on tabular (two dimentional) data. Because we want the cost of creating a stats component to be low, we only want to generate stats that can be calculated on a rolling basis.

The different types of data that we can generate stats on are number (including integer and floating point), string, and boolean.

We can generate different stats depending on the type of that column. Note, "valid" here means the result of running the given jsonschema structure against this cell of data.

Number:
  valid - the number of cells in that column that contain valid numbers. Validity is based on the the json 
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
    "valid": 4,
    "error": 0,
    "missing": 0,
    "minLength": 2,
    "maxLength": 2
  },
  {
    "title":"students",
    "type":"number",
    "valid": 4,
    "error": 0,
    "missing": 0,
    "min": 17,
    "max": 22,
    "avg": 19.5,    
  },
  {
    "title":"can accept more students",
    "type":"boolean",
    "valid": 4,
    "error": 4,
    "missing": 4,
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
    "valid": 4,
    "error": 0,
    "missing": 0,
    "minLength": 2,
    "maxLength": 2
  },
  "students": {
    "title":"students",
    "type":"number",
    "valid": 4,
    "error": 0,
    "missing": 0,
    "min": 17,
    "max": 22,
    "avg": 19.5,    
  },
  "can_accept_more_students": {
    "title":"can accept more students",
    "type":"boolean",
    "valid": 4,
    "error": 0,
    "missing": 0,
    "trueCount": 2,
    "falseCount": 2,
  }
}
```

In summary, qri can generate some basic stats on a dataset body, if the body is tabular. We also need a structure with a schema in order to understand what stats to be calculating.

## high dimentional data

Calculating statistics on high dimensional data is out of scope for this RFC. however we do provide a way to arrange statistical metadata into hierarchies by nesting stats object with a key: stats. Given a sample of high-dimensional data:

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
    }
  ]
```
Here's an example of "stats that contain stats"
```json
{
  // other dataset components ...
  "stats":  [
    { "title": "type", "type" : "string" /* same as 2D string stat */ },
    {
      "title": "geometry",
      "type" : "object",
      "error" : 0,
      "valid" : 2,
      "missing" : 0, 
      "stats": {
        "geometry": { 
          "type":"string", /* same as 2D string stat */
          "coordinates": {
             "type" : "geopoint",
             "avg" : [-74.00, 41.00]
          }
        }
      }
    }
  ]
}


What's important to note here is stats is a recursive data structure that uses the keyword "stats" to build up statistical hierarchies. The actual calculation of these statistics is out of scope, but their presentation can be validated, mainly by reserving the stats keyword within a stats object.

```
## Custom stats

This is out of the scope of this current RFC, but Qri should implement a way for users to add their own custom stats during a transform. We should create a starlark module called `stats`, and a function on the dataset object `ds.set_stats`.

The starlark module `stats`, should have a `defaultStats` function that would return the exact same default stats that would occur during dataset creation.

Dataset creators should be well informed that when they create custom stats, the default template may not render a useful visualization. If you create custom stats, you should also create a custom template to match.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation\

- implimented at the dsio level
- adding a section to Dataset
- basically walking through what b5 has already added

## Stats component size calculations

Calculations on the size of a stats component are based on worst case senario. In our worse case senario each stat would be a boolean stat (the default with the largest ratio between stat size and data size), with a 200 character title, and a 19 digit number for each of count, trueCount, and falseCount. *note: this is disengenuous, because the count trueCount and falseCount max out at the number of rows, but because we are looking at the worst case senario let's keep it as high as possible*

A single stat in the worst senario is around 342 bytes.

In contrast, a dataset with one column and one row, whose only cell contains a a boolean, only takes up 8 bytes. If this same dataset, with one column, would need around 40 rows to even equal the size of the stats component. 

We know, however, that the size of the Stat component will only increase if we add another column to the dataset. In fact, it increases linearly, proportional to the number of columns. So if there are 10 columns, then the worst case stats component will be 3.4 kilobytes. Any added rows to the dataset do not add to the size of the Stats component. If the number of columns remains the same, the stats component will change very little. And, better yet, as the number of rows increase in a dataset, the Stats component becomes more and more valuable.

Our worst case scenario dataset would have a huge number of columns, and only one row. While a dataset structure like that is possible, it is unlikely. Far more likely are datasets with 10-30 columns, and over 50 rows. 

Let's go to an extreme. Let's say we have a 400Mb dataset, with 400 columns. That dataset's Stats component is going to be around 137kbs. The stats would be around 3000 times smaller then the actual data itself. Even if we had 4000 columns, and the Stats component was 1.4MBs, that's still 235 times smaller then the data itself. And it gets more valuable the fewer the number of columns.

Let's take a typical example. A dataset that is around 200Mbs with about 10 columns. The stats component would be about 3.4kbs, making the stats component 58823 times smaller than the data itself. In our typical and extremely large dataset examples, the stats component is of a negligable size.

# Drawbacks
[drawbacks]: #drawbacks

Adding a stats component would disrupt what has become a relatively settled dataset model. 

Although the default stats component should only add a small amount of data to our dataset package, it is still adding extra kilobytes. Also, when custom stats are available, if a dataset creator doesn't think through the stats they are adding, they may actually add much more than a few kilobytes of glut.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
If we do not add a stats component, we will run into issues as we move forward with fleshing out templates for visualizations. We do not want dataset creators to have to inject the entire dataset body into a template in order to render an interesting visualization. With a stats component, the user can do some analysis on the dataset, save it to the stats component, and inject the the statistical data it needs to display in a rendered viz. Using the stats component rather then the body, will save an order of magnatude of space, rendering, and loading time.

We can add stats to the structure component. This would most make sense in place of or added onto the json schema. However, we definitely get a lot of wins by using json schema, so we really don't want to replace it. As for adding to what json schema already gives us, for tabular data, this is actually probably possible. However, we also want to allow custom stats and a way forward for stats on high dimentional data, this seems much more difficult. The problem of presenting stats on high dimentional data is difficult enough without having to also add the complexity of describing high dimentional data. Having the stats in a separate location feel better then adding it to structure.

We already have a meta component that describes the dataset, we could potentially just add a `stats` section there, or just have dataset creators add stats as an arbitrary field each time they need them. However, the Meta component is very easy to mess with, and there are very few bounds on what you can add. Part of our guarentee with the Stats component, is that the Stats reflect the body, because they were generated from of that body. Adding to an arbitrary field in Meta does not give us that guarentee. 


# Prior art
[prior-art]: #prior-art

Since stats relies on json schema, and because json schema's main function is to describe data structures, we looked at what we could copy from json schema.

Kaggle is a dataset project that shows some basic stats for each column of data. Our proposed list of stats and their list of stats has alot of overlap. Theirs also includes a few that we deemed too computationally heavy to being with:
for number => valid, mismatched, missing, the mean, standard deviation, and quantities for each percentile
for string => valid, mismatched, missing, # of unique entries, most common entries (and %)
date => valid, mismatched, missing, minimum, mean, maximum
boolean => valid, mismatched, missing, unique, most common


# Unresolved questions
[unresolved-questions]: #unresolved-questions
What does representing high dimentional data look like?
What are the exact specifications of the starlark `stats` module, and dataset object `ds.set_stats` function?
