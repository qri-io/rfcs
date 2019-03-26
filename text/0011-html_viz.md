- Feature Name: html_viz
- Start Date: 2018-08-14
- RFC PR: [#13](https://github.com/qri-io/rfcs/pull/13)
- Issue: <!-- (leave this empty) -->

_Note: We never properly finished the HTML rendering RFC, which we started work on in August 2018.
Instead of creating a new one we just "finished what we started" in March 2019. As such this RFC
contains references to RFCS developed in the period between August 2018 & March 2019.
-:heart: the qri core team_

# Summary
[summary]: #summary

HTML vizualizations are instructions embedded in a dataset rendering a dataset as a standard HTML document. 

# Motivation
[motivation]: #motivation

Qri has a syntax-agnostic `viz` component that encapsulates the details required to visualize a dataset. This RFC proposes the first & default visualization syntax should be rendering to a single HTML document using a template processing engine.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


### `qri render` & default template

A command will be added to both Qri's HTTP API ('API' for short) & command-line interface (CLI) called `render`, which adds the capicty to execute HTML templates against datasets. When called with a specified dataset `qri render` will load the dataset, assume the viz syntax is HTML, and use a default template to write an HTML representation of a dataset to `stdout` on the CLI, or the HTTP response body on the API:
`qri render me/example_dataset`

The default template and output path can be overridden with the `--template` and `--output` flags respectively. the output is on the CLI only:
`qri render --template template.html --output rendered.html me/example_dataset`

The default template must be carefully composed to balance the size of the resulting HTML file in bytes against readability & utility of the resulting visualization. It should also include a well-constructed citation footer that details the components of a dataset in a concise, unobtrusive manner that invites users to audit the veracity of the dataset in question. These defaults should encourage easy reading and invite verification on the part of the reader.

### Vizualizations in datasets

Saving a dataset will by default execute the default template to a file called `index.html` & place it in the IPFS merkle-DAG. When an IPFS HTTP gateway receives a request for a DAG that is a directory containing `index.html`, it returns the HTML file by default. This means when a user visits a dataset on the d.web–completely outside the Qri system of dataset production–they are greeted with a well-formatted dataset document by default.

While care will be taken to keep `index.html` files small, users may understandably want to disable them entirely. To achieve this we'll add a new flag to `qri save` and `qri update`: `--no-render`. No render will prevent the execution of any viz template. This will save a few KB from version to version at the cost of usability.

Users can _override_ the default template by supplying their own custom viz templates either by specifying a `viz.scriptPath`:

`dataset.yaml`:
```yaml
name: example_dataset
# additional fields elided ...
viz:
  syntax: html
  scriptPath: template.html
```

and running save:
```
$ qri save --file dataset.yaml
```

Or by running `qri save` with an `.html` file argument:
`qri save --file template.html me/example_dataset`

Since the above example provided no additional configuration details for the `viz` component in `dataset.yaml`, the two calls will have the same effect.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Template API

Introducing HTML template execution requires defining an API for template values. This API will need to be documented & maintained just like any other API in the Qri ecosystem.

The template implementation will use the [go language html template package](https://golang.org/pkg/html/template)

#### Dataset is exposed as `ds`

HTML template should expose a dataset document at `ds`. By exposing the document as `ds`, it matches our for referring to a dataset in starlark., and allows access to all defined parts of a dataset. ds should use _lowercase_ fields for component accessors. eg:

```html
{{ ds.meta.title }}
```

Undefined components should be defined as empty struct pointers if null. For example a dataset _without_ a transform the following template should print nothing:
```html
{{ ds.transform }}
```
And this should error:
```html
{{ ds.transform.script }}
```

Having default empty pointers prevents unnecessary `if` clauses, allowing a skip to tests for existence on component fields:
```html
{{ if ds.transform.script }}
{{ end }}
```

### Template functions

Top level functions should be loaded into the template `funcs` space to make rendering templates easier. The go html template package comes with [predefined functions](https://golang.org/pkg/text/template/#hdr-Functions), all of which are included in this RFC. An example prints the length of the body:

```html
<h3> The Body has {{ len ds.get_body }} elements </h3>
```

In addition to the stock predefined functions the following should be loaded for all templates to make templating a little easier:

| name        | description              |
| ----------- | ------------------------ |
| timeParse   | parse a timestamp string, returning a golang *time.Time struct |
| timeFormat  | convert the textual representation of the datetime into the specified format |
| default     | Allows setting a default value that can be returned if a first value is not set. |


#### future dataset document API

We have reserved future work for a "dataset API" that will expand the default capabilities of a dataset document to include convenience functions for doing things like loading named columns or sampling rows from the body. We've intentionally left this API undefined thus far to understand how it will work in different contexts. One such context is this template API.

The one exception to this is exposing body data through a function on `ds`: `ds.get_body`. This is because there's a _very_ high chance we'll want to export `ds.body` as an object with methods in the future. If we simply load the entire body & drop it into `ds.body`, adding methods to `ds.body` will require breaking the document API.

### The default template

Our standard template should actually be a collection of pre-defined partials, looking something like this:

```html
<!DOCTYPE html>
<html>
  <head>
  {{ partial "stylesheet" }}
  </head>
  <body>
  {{ partial "header" ds }}
  {{ partial "summary" ds }}
  {{ partial "stats" ds }}
  {{ partial "citations" ds }}
  </body>
</html>
```

Users can then swap in these predefined partials to make partially-custom templates a thing, and ease the transition to fully-custom visualizations through progressive customization. The predefined partials are as follows:

| name       | purpose |
| ---------- | ------- |
| stylesheet | default css styles |
| header     | dataset title, date created & author in a `<header>` tag |
| summary    | overview of dataset details |
| stats      | template of stats component that prints nothing if the stats component is undefined |
| citations  | `<footer>` tag that prints hash content & links to in-package files |

In practice I'm hoping the `citations` partial will get a lot of use, making it easier to drop properly-formatted citations into markup with little thought.

# Drawbacks
[drawbacks]: #drawbacks

One of the biggest drawbacks to the proposed implementation comes from a natural tendency to use the following to template the contents of a dataset body in to a dataset:

```html
<script type="text/javascript"> window.data = {{ ds.get_body }} </script>
```

Using templating this way prints the entire body into the template. If the body is large, this will cause _all body data to be duplicated in `index.html`_. I propose we fight this through documentation, because there are many scenarios where this code is completely appropriate (for example, when the dataset body is an aggregation of some other data source).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

there are more common templating styles that may make more sense. [Mustache](https://mustache.github.io/mustache.5.html) comes to mind. All of these will be more work to integrate than the standard go package. I've tried to focus on removing the "go-specific" aspects that often show up in templates like Title.Case.Accessors and using `.` to refer to the current template data.

# Prior art
[prior-art]: #prior-art

* [Hugo's template functions](https://gohugo.io/functions/)
* [Mustache templates](https://mustache.github.io/mustache.5.html)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Implementation in another language
It's unclear if this RFC is specific enough to make templates created in this context & executed in go-qri function in another language. Future work should scope _down_ the allowed syntax, with an eye toward portability.

### Dataset API
We need to put time into the dataset API, but it's also unclear weather the template engine should have full parity with that forthcoming API. Template execution is supposed to be limited, and smart decisions will need to be made to keep users from writing too much business logic into their templates.