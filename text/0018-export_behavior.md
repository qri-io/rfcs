- Feature Name: Export Behavior
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Export functionality exists in qri, but is lacking many options and pieces of functionality.

# Motivation
[motivation]: #motivation

An <a href="https://github.com/qri-io/rfcs/blob/master/text/0014-export.md">earlier RFC</a> began describing the export functionality. However, it left a number of details unspecified, which led to too much uncertainty to move forward with implementation. While it did mention many of the options for export, it left out discussions of defaults, the interactions between options, and basically how `export` is expected to work. This RFC hopes to remedy this situation so that implementation can proceed without hesitation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Export is a piece of functionality that will generate the version of a dataset which is external to qri storage, so that it can live instead on a local filesystem. The goal is to have a document defintion of the structured data that has been, up until now, represented only internally by qri. Here are some of the use cases that `export` is meant to handle:

* Getting a local binary copy of a dataset, such that it can be safetly deleted from qri, and later imported without any loss of fidelity or history.
* Exporting an entire dataset with metadata and body to analyze in a custom script. A single file "json" or "yaml", for example.
* Converting to a binary format (like xlsx) for use in another program like Excel or OpenOffice.
* Deciding to stop using qri, and wanting to get all of your data out in a high-fidelity form, with both machine and human readable formats.

These should all be doable using `export`.

Some non-goals:

* Retrieving only a single component of a dataset, like the title.
* Converting a dataset's body to a different format than it is being stored as (csv to json or cbor).

These types of operations should be added to `get` instead, which already has similar functionality.

## Formats

Each of the above listed use cases for `export` corresponds to a different export format. Here they are discussed in more detail.

### Native export

The default option is a native export. This consists of a zip file containing two things, a `manifest.cbor` file, then a list of binary blocks containing the data that is directly read from IPFS. The manifest simply enumerates these blocks. This native format is designed to work best with reimports; using it will recreate the dataset with no loss of information.

### Foreign export

An `export` can also be foreign, using the `--format` flag to choose a specific file format. For example:

```
qri export --format json me/my_dataset
```

```
qri export --format xlsx me/my_dataset
```

A file created by these commands still contains as much data about the original dataset, but reimporting it is not guaranteed to end up with the exact same result as the dataset it was exported from.

For the case of "json", the result will be a single file, with the body stored inline in the document.

### Full fidelity export

If a user wishes to stop using qri, and wants to get a fully detailed record of the data that was involved in their usage of the system, they can request a full export. This produces a file that contains all the raw binary data from qri, as well as human-readable versions, so that they can work with their exported data without needing qri anymore to process it.

One proposal for this flag:

```
qri export --takeout me/my_dataset
```

The `--takeout` flag can be combined with `--format` to decide what the format of the human-readable document should be, defaulting to "json".

## Output filename

An option not mentioned previously is a `-o` flag (long form `--output`), which can be used to set the output path.

This is a powerful option, as it actually a few effects:

* Decides the filename and also the path to which the export is written
* Can be used to infer the output format via file extension, such as "json" or "xlsx" or "zip"

If not specified the default output path should use the file extension based upon the format. Since the default format is "native", this means an `export` with no options will output a ".zip" file. It is an error to use both `--format` and `-o` if the output file extension contradicts the format flag.

In addition, the basename of the output path should be derived from the name of the dataset, and should be append a formatted timestamp of the most recent commit to the dataset. For example:
 
`qri export me/my_dataset`
 
Should be saved to "my_dataset_[formatted_timestamp_of_most_recent_commit].zip". This means exports of the same version of the dataset will have the same filename, while new versions will have different filenames.
 
When the `export` command is completed, the command-line should print a success message that includes the filename of the exported result. This should be done whether the output filename is derived from the dataset name and timestmap, or if its specified using the `-o` flag.

## Transform and viz

For foreign exports, the transform and viz fields (if present in the dataset), will just be represented as paths. The flags `--inline-transform` and `--inline-viz` can be used to store them as base64 encoded files instead.

## All revisions

Using the flag `--revision=all` or `--all` will include all past versions of a dataset. This feature will not be implemented at first, but will be added in a follow up change.

## Body format in single-file exports

When a dataset is being export as a "json" document, it will be necessary to convert the body to also be json. In addition, the Structure.Format field should be updated to reflect this new json format.

It may be desired to export a document as "yaml", or to have a body as a "csv" or other format. In these cases, the body would need to be byte-encoded, so that it could be properly represented in the host format.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A brief summary of the main flags, and their effects:

| flag          | type    | raw   | document |
| ------------- | ------- | ----- | -------- |
| (default)     | native  | yes   | none     |
| --format json | foreign | no    | json     |
| --format yaml | foreign | no    | yaml     |
| --format xlsx | foreign | no    | xlsx     |
| --takeout     | native  | yes   | json     |

Other flags:

```
-o (--output) Filename to output, file extension implies --format
--revisions   How many revisions to export, "all" for all
--all         Same as --revisions=all
```

# Drawbacks
[drawbacks]: #drawbacks

No good reasons not to do this.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

More detail and specification is needed before implementation can go forward.

# Prior art
[prior-art]: #prior-art

[https://github.com/qri-io/rfcs/blob/master/text/0014-export.md](https://github.com/qri-io/rfcs/blob/master/text/0014-export.md)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The `--revisions` flag needs a bit more exploration before it is implemented. The plan is to first all the basic export functionality, and come back to revisions later on.


