- Feature Name: starlark load_dataset
- Start Date: 2018-03-15
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Introduce a global function `load_dataset` to starlark scripts both loads the dataset and declares it as a dependency. Deprecate all `qri.load_dataset` functions, making `load_dataset` the only way to depend on an external dataset.

This RFC assumes all datasets are public. In the future `load_dataset` will be expanded, accepting optional arguments to scope the kinds of access a script is requesting.

# Motivation
[motivation]: #motivation

Qri's goal is to build an environment where datasets are as a first class citizen. Qri currently monitors any call to `qri.load_dataset` and records the loaded dataset as a dependency. This results in a complete dependncy graph.

There are (at least) three problems with this approach: 

* **scattered requirements**
  Dependencies are not explicit in code. The only way to know what datasets a script depends on is to read the entire script. We've learned this lesson many times with software dependencies, and know to explicitly require dependency-forming statements to be at the top of a code file. There's no reason this shouldn't apply to depending on datasets.
* **inexpressive usage**
  `qri.load_dataset` functions are is not expressive enough to capture all the uses for a dataset, building up patternas around loading pieces of a dataset inside special functions like `transform` is missing the point of thinking of datasets as _documents_.
* **expanding adds to `qri` module, not dataset documents**
  Currently the `qri` module returns plain values from methods like `load_dataset_body`. If we wanted to add a function that, say, only selected specific parts of a dataset body, that function would be added to the `qri` module. This issue will only be exacerbated when we try to overlay access control onto datasets. It makes more sense to define all ways a dataset will be used at the point of import, and have Qri return a `dataset` document that is scoped to that use.

For these reasons, we should make datasets a first class citizen in starlark scripts by creating a global `load_dataset` statement that can be expanded with permissions scopes in the future. This statement should return a _dataset document_ that we can expand the functionality of over time. 


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

```python
load("http.star", "http")
# load a dataset into a variable named "fhv"
load_dataset("b5/nyc_for_hire_vehicles", "fhv")

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

Knowing that things like private datasets are planned for the future, we need an explicit way to load a dataset that we can build upon. While we can't yet define what those nuances will be, we do need to define _how they will be expressed_. Here are examples of possible scoping statements, expressed as optional arguments: 

```python
# scope a dataset load to only meta & structure components
load_dataset("b5/city_residents", "residents", components=["md", "st"])

# only allow access to "residents" during the "transform" step. "residents" will 
# be equal to None during all other steps
load_dataset("b5/city_residents", "residents", steps=["transform"])

# apply differential privacy to all statements that read from the body
load_dataset("b5/city_residents", "residents", anonymized=True)

# require that the permissions of the dataset that results from this transform
# match the permissions of "b5/city_residents"
load_dataset("b5/city_residents", "residents", forward_permissions=True)
```

These are suggestions only. We need lots of time to think through how these options will work, this rfc only adds the assumption that those statements will be expressed as options on `load_dataset`. We can build on the already-present assumption that loaded datasets are read-only, making all options pertain to read access.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The general form of `load_dataset` is as follows:

```
def load_dataset(reference, target_name)
```

**reference** is a string reference to a dataset, as defined by our dataset naming conventions elsewhere. Any valid dataset reference should work with one exeption: no "me" statements or context-dependant references are allowed:

```python
# this will error:
load_dataset("me/population", "pop")
# this will work:
load_dataset("b5/population", "pop")
```

this is to keep transform scripts portable by freeing them from dependencies on execution context.

**target_name** is the name this script will use to refer to this dataset in the script. The supplied string must be a valid starlark variable name.

This variable-style import makes sense for datasets, who's names are often lengthy. Unlike package imports, we anticipate it will be very common for dataset names to collide, being distinguished only be peername. this required assign-to-name pattern will help here as well.

# Drawbacks
[drawbacks]: #drawbacks

### differences with `load`
As currently proposed, there's a subtle-but-important distinction between `load` and `load_dataset` that may cause confusion. Here's an example:
```python
# load the `http` component of the http "package object"
load("http.star", "http")
# load a dataset into a variable named "fhv"
load_dataset("b5/nyc_for_hire_vehicles", "fhv")
```

In the `load` statement, the value `http` is defined by the module. writing `load("http.star", "cats")` would error because `cats` is not defined on the http module.

On the other hand `load_dataset` assumes that only _one_ value is available, and it will be assigned to a second, _user-supplied_ argument. calling `load_dataset("b5/nyc_for_hire_vehicles", "cats")` would _not_ error, instead loading the dataset document into a global variable named `cats`. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### return a value from `load_dataset`
One alternative would be to have `load_dataset` return a value that could be assigned. As an example:
```python
fhv = load_dataset("b5/nyc_for_hire_vehicles")
```

I'd prefer to avoid this approach because in the future we will need to make alterations that will make the return value behave less like a variable. If the following option existed:
```python
# only allow access to "residents" during the "transform" step. "residents" will 
# be equal to None during all other steps
fhv = load_dataset("b5/nyc_for_hire_vehicles", steps=["transform"])
```
we need to mutate the value of `fhv` at runtime. I find it easier if the user think of a dataset as a kind of module that behaves differently from a value set to a variable.

### alternate nameas
We can also consider alternative names for `load_dataset`. To me the requirements are as follows:
* show a dependency is being created
* differ from `load`, which refers to requiring code
* prefer concise names



# Prior art
[prior-art]: #prior-art

We're taking a lot of inspiration for the security model for imports & dependencies from the [deno project](https://github.com/denoland/deno)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
