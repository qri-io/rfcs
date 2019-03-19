- Feature Name: starlark load_dataset
- Start Date: 2018-03-15
- RFC PR: [#38](https://github.com/qri-io/rfcs/pull/38)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Introduce a global function `load_dataset` to starlark scripts that both loads the dataset and declares it as a dependency. Deprecate all `qri.load_dataset` functions, making `load_dataset` the only way to depend on an external dataset.

This RFC assumes all datasets are public, and that names refer to a dataset the script author can resolve. In the future `load_dataset` will be expanded, accepting optional arguments to scope the kinds of access a script is requesting.

# Motivation
[motivation]: #motivation

Qri's goal is to build an environment where datasets are as a first class citizen. Qri currently monitors any call to `qri.load_dataset` and records the loaded dataset as a dependency. This results in a complete dependency graph.

There are (at least) three problems with this approach: 

* **scattered requirements**
  Dependencies are not explicit in code. The only way to know what datasets a script depends on is to read the entire script. We've learned this lesson many times with software dependencies, and know to explicitly require dependency-forming statements at the top of a code file. There's no reason this shouldn't apply to depending on datasets.
* **inexpressive usage**
  `qri.load_dataset` functions are not expressive enough to capture all the uses for a dataset, building up patterns around loading pieces of a dataset inside special functions like `transform` is missing the point of thinking of datasets as _documents_.
* **expanding adds to `qri` module, not dataset documents**
  Currently the `qri` module returns plain values from methods like `load_dataset_body`. If we wanted to add a function that, say, only selected specific parts of a dataset body, that function would be added to the `qri` module. This issue will only be exacerbated when we try to overlay access control onto datasets. It makes more sense to define all ways a dataset will be used at the point of import, and have Qri return a `dataset` document that is scoped to that use.

For these reasons, we should make datasets a first class citizen in starlark scripts by creating a global `load_dataset` statement that can be expanded with permissions scopes in the future. This statement should return a _dataset document_ that we can expand the functionality of over time. 


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `load_dataset`
This RFC proposes switching to a single method of loading a dataset that is analogous to loading a module. Dataset dependencies will be declared (by convention) grouped together at the top of a script, forming a clear delaration of the datasets this script depends on: 

```python
load("http.star", "http")
# load a dataset into a variable named "fhv"
fhv = load_dataset("b5/nyc_for_hire_vehicles")

def download(ctx):
  # use the fhv dataset to inform an http request
  vins = ["%s,%s" % (entry['vin'], entry['model_yearl']), for entry in fhv.body()]

  res = http.post("https://vpic.nhtsa.dot.gov/api/vehicles/DecodeVINValuesBatch/", form_body={
    'format': 'JSON', 
    'DATA': vins.join(";")
  })

  return res.json()

def transform(ds, ctx):
  ds.set_body(ctx.download)
```

In this example, the `load_dataset` statement supplies no optional arguments, it's assumed this dataset is loaded and available in all parts of the script. This RFC defines no optional arguments for `load_dataset`, so loading datasets for global use is the only option available. Considering all datasets on Qri are currently public, this is a sensible place to start.

### Access Control
Knowing that things like "private datasets" are planned for the future, we need an explicit way to load a dataset that we can build upon. While we can't yet define what those nuances will be, we do need to define _how they will be expressed_. Here are examples of possible scoping statements, expressed as optional arguments: 

```python
# scope a dataset load to only meta & structure components
residents = load_dataset("b5/city_residents", components=["md", "st"])

# only allow access to "residents" during the "transform" step. "residents" will 
# be equal to None during all other steps
residents = load_dataset("b5/city_residents", steps=["transform"])

# apply differential privacy to all statements that read from the body
residents = load_dataset("b5/city_residents", anonymized=True)

# require that the permissions of the dataset that results from this transform
# match the permissions of "b5/city_residents"
residents = load_dataset("b5/city_residents", forward_permissions=True)
```

These options are not giving additional permissions to scripts, as that would go against the no-need-to-audit goal, they are instead self-restricting what the loaded dataset may be used for. They exist for the benefit of the transform author, not for the security of the transform consumer.

These are suggestions only. We need lots of time to think through how these options could work, this rfc only adds the assumption that those statements will be expressed as options on `load_dataset`. We can build on the already-present assumption that loaded datasets are read-only, making all options pertain to read access.

We _do_ need to declare what the default `load_dataset` expression means in relation to permissions:

```python
residents = load_dataset("b5/city_residents")
```

The default load dataset should be taken to mean "load this dataset with the _maximum available permission_". If a dataset is open, the user is free to read from any and all parts of the document.

Combining a default of requesting maximum permission with _restrictive_ permission declared on the imported dataset supports our default stance of an open dataset ecosystem. The onus of declaring how a dataset can be used will fall to the owner of the dataset, and defaulting to max permissions forces the owner to assume that "if it's permitted, someone will use it", which is the correct assumption when attempting to restrict access.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`load_dataset` has a fairly simple definition:

```
def load_dataset(reference): # return: dataset object
```

* **reference** is a string reference to a dataset, as defined by our dataset naming conventions elsewhere.
* **dataset object** the returned value is a dataset object, as defined by ds.NewDataset in `qri-io/startf/ds`.

As mentioned earler, the plan is to expand this definition with options once we've had time to do more research. Theses examples of valid reference strings, with `QmProfile` and `QmHash` being standins for full base-58 encoded hashes:

```python
load_dataset("b5/dataset")
load_dataset("b5/dataset@QmProfile/ipfs/QmHash")
load_dataset("@/ipfs/QmHash")
```

### `load_dataset` must be top level scope
`load_dataset` statements will only be allowed within the primary scope of a transform script. Phrased another way: `load_dataset` _cannot_ be called within a function. This limits the scope of what is possible when composing dynamic import statements. This **won't** work:

```python
load("qri.star", "qri")

# create a dataset of all dataset titles
def transform(ds, ctx):
  # error: load_dataset must be called in top level scope
  datasets = [load_dataset(name) for name in qri.list_datasets()]
  ds.set_body([ds.get_meta('title') for ds in datasets])
```

This example is making use of a _privileged resource_ (the local qri node), which should be denied by default. For this assumption to hold, privileged resources must _only_ be available within special functions, an assumption we already enforce.

Statements that use logic to compose import strings should also be denied:

```python
# error: load_dataset must be called with a string value
pop_years = [load_dataset("b5/population_%s" % year) for year in range(2011,2017)]
```

This convenience function for creating legal string names breaks static analysis tools, making it impossible to determine a script's dependencies without executing code.

### Name Resolution
When the starlark execution environment encounters a `load_dataset` statement, it has to _resolve_ the name, connecting the string reference to a dataset object. This may or may not require network access, depending on if the user currently has all of the specified versions locally. All naming duties required by `load_dataset` will be offloaded to Qri's name resolution system, but a few interactions merit calling out here:

#### Default to fetching non-local dependencies
By default, any dataset referenced with `load_dataset` that isn't local should be fetched from the network, using whatever heuristics Qri's name resolution system is configured to use (currently: fetching from both the registry & p2p network). This frees the user from having to consider what data they do & do not have. Requiring `load_dataset` be called in top level scope means non-local fetching will run before any special functions (`download`, `transform`) are called. This has the natural effect of requiring dependency resolution before the "main" functions of a transform script can execute.

#### Name resolution errors
This interaction with network availability will need to be managed. If, for example, a script were run in conjunction with a hypothetical `--offline` flag like this:

```
$ qri save --offline --file transform.star me/example
```

Referencing any non-local dataset in `load_dataset` will raise a name resolution error. But if all depencies are locally satisfied, the transform should execute without error. If a dataset cannot be loaded, or doesn't exist anywhere, the script MUST exit. Providing a recovery mechanism would allow probing datasets to see whether they exist or not.

### Don't pin dependencies by default
The default behaviour should be to **not pin dependencies**. We should add a new flag to `save` and `update` commands: `--pin-deps` that pins dependencies locally. The reason for this choice relies on default settings in other parts of Qri. A "stock" installation of Qri defaults to storing datasets in an IPFS repository restricted to 10Gigs of space. In practice most datasets so far have been less than 100Mb in size, and most users coming to Qri either understand how to manually manage their IPFS storage, or are only using IPFS through Qri. These choices accumulate to lots of free space in the 10Gig repository that can go for some time before garbage collection is needed. If garbage collection _is_ required, the first thing that should go are transient dependencies required by previous transform scripts, which will be the case here. Future tools that perform graph analysis on versioned dependency trees can be introduced to add fine-grained pinning review & management. Did I mention that this work never stops?

### Relative "me" references aren't allowed:
Any valid dataset reference should work with one exeption: no "me" statements.

```python
# this will error:
pop = load_dataset("me/population")
# this will work:
pop = load_dataset("b5/population")
```

This is to keep transform scripts portable by freeing them from dependencies on execution context.

This variable-style import makes sense for datasets, who's names are often lengthy. Unlike package imports, we anticipate it will be very common for dataset names to collide, being distinguished only be peername. this required assign-to-name pattern will help here as well.

# Drawbacks
[drawbacks]: #drawbacks

### differences with `load`
As currently proposed users are expected to assign the result of `load_dataset`
```python
# load the `http` component of the http "package object"
load("http.star", "http")
# load a dataset into a variable named "fhv"
fhv = load_dataset("b5/nyc_for_hire_vehicles")
```

In the `load` statement, the value `http` is defined by the module, which can only be known by reading the source. I find starlark's `load` statement opaque, and don't consider it a pattern we should follow.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### load into a user-supplied string
One alternative would be to have `load_dataset` work a little closer to the way `load` does in starlark, having a user supply a second string argument that names a global variable:
```python
# load b5/nyc_for_hire_vehicles into a global variable named "fhv"
load_dataset("b5/nyc_for_hire_vehicles", "fhv")

# use fhv:
fhv.get_body()
```

I attempted to code this up, it was both difficult to execute, and felt too "magical" in practice.

### alternate names
We can also consider alternative names for `load_dataset`. To me the requirements are as follows:
* show a dependency is being created
* differ from `load`, which refers to requiring code
* prefer concise names


# Prior art
[prior-art]: #prior-art

We're taking a lot of inspiration for the security model for imports & dependencies from the [deno project](https://github.com/denoland/deno)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Loading from history
One aspect not yet covered by this RFC is how to load previous versions of the dataset a script is affecting. The closest I have so far is supplying some sort of relative ref with no name:

```python
prev = load_dataset("~1")
```

The current thinking is to _not_ do this because it's not safe to assume that "~1" is a dataset meant for public consumption. Loading historical entries clearly needs more thought, I'm hoping to cover this in a subsequent RFC that helps refine & clarify the distinction between _names_ & _selection_.

### Dataset object
This RFC adds pressure to the api defined around dataset documents in starlark. We're on a collision course with a `dataframe-like api` that'll need to get worked out in a separate RFC. I think we can survive on the current model until a subsequent RFC can be written with extensive research ondataframe-like APIs.

### Dynamic dependencies
This RFC explicitly denies building dynamic dependencies by forcing `load_dataset` to be top-level calls. 
 
I think we should instead follow up with a subsequent rfc to handle dynamic dependencies. I think dynamic dependency loading should happen in a new special function created expressly for this purpose. An example of this would be a `load_datasets` special function that allows dynamic dependencies:

```python
load("qri.star", "qri")

def load_datasets(ctx):
  return [load_dataset(name) for name in qri.list_datasets()]

# create a dataset of all dataset titles
def transform(ds, ctx):
  datasets = ctx.datasets
  ds.set_body([ds.get_meta('title') for ds in datasets])
```

I'd rather introduce this later after more concrete use-cases have been created so we can design a solution that addresses as many as possible while keeping the sandbox model intact. This solution, for example, would not allow http access to determine dependencies.