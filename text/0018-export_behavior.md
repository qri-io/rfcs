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

I believe the starting point on getting export to work as well as we want is to enumerate some use cases and figure out what the reasonable defaults are.

Here are some of the use cases that pop into mind:

* Getting the body as json or csv and writing to a file, the opposite of `save`.
* Exporting an entire dataset with metadata and body to analyze in a script. A single file "json" or "yaml", for example.
* Compressing into a zip file for storage / transportation / archive.
* Exporting with the intent to import later on.
* Converting to a binary format (like xlsx) for use in another program.

Considering these possible use cases, the one that most seems to fit the "vanilla" behavior of `export`, what happens when no options are specified, is to export a dataset with metadata and body as a single file.

`qri export me/my_dataset`

Should produce a json file containing the Dataset / DatasetPod definition, from github.com/qri-io/dataset/dataset.go, as well as the Body stored inline.

The options mentioned in the <a href="https://github.com/qri-io/rfcs/blob/master/text/0014-export.md#reference-level-explanation">previous rfc</a> apply mostly to this starting point. For example:

`qri export me/my_dataset --section body`

Can be used to only export the body itself.

The flag `--zip` can be used to output the full dataset or body as a zipped file instead:

`qri export me/my_dataset --zip`

`qri export me/my_dataset --zip --section body`


### Additional files

The dataset definition doesn't contain everything related to the dataset. Specifically, the `transform` and `viz` files are stored as separate files, and the dataset definition only links to their paths. In order to export these additional files, the command must use either the `--zip` or `--directory-of-files` options.

QUESTION: Should this be changed? Should it be possible to store Transform and Viz both inline in the Dataset? If so, should Transform and Viz be treated as "sections"?

### Output filename

An option not mentioned previously is a `-o` flag (long form `--output`), which can be used to set the output path.

This is a powerful option, as it actually has multiple effects:

* Decides the filename and also the path to which the export is written
* Can be used to infer the output format via file extension, such as "json" or "yaml"
* Can imply other options like `--zip` if extension is ".zip" or `--directory-of-files` if path ends with "/".

If not specified the default output path should use the file extension based upon the format. Since the default format is "json", this means an `export` with no options will output a ".json" file. It is an error to use both `--format` and `-o` if the output file extension contradicts the format flag.

In addition, the basename of the output path should be derived from the name of the dataset, and should be append a formatted timestamp of the most recent commit to the dataset. For example:

`qri export me/my_dataset`

Should be saved to "my_dataset_[formatted_timestamp_of_most_recent_commit].zip". This means exports of the same version of the dataset will have the same filename, while new versions will have different filenames.

When the `export` command is completed, the command-line should print a success message that includes the filename of the exported result.

### Body format

The currently existing flag `--body-format`, should be changed to `--format-body`, as specified in the previous rfc, in order to better lineup with other proposed options `--format-header` and `--format-file`.

The `--format-body` flag is special in that it requires an explicit conversion from one format that matches the dataset.structure.format to a different format. This means that if `--format-body` is supplied and causes this conversion to happen, the resulting body format will no longer be correctly described by the structure once it is exported.

Furthermore, this implies that when an exported dataset is used, either by an external tool or by a reimport back into qri, a mismatch between the dataset.structure.format field and the encoding format of the body should be taken to mean that a reconversion is required to turn the body back into its original format.

For example, say we have a dataset as follows:

```
{
  "structure"": {
    "format": "cbor",
    ...
  },
  "body": ... // cbor data
  ...
}
```

And a user runs:

`qri export me/my_dataset --zip --format-body json`

This will create "my_dataset-[timestamp].zip" which contains "dataset.json" and "body.json", despite dataset.json having the field dataset.structure.format == "cbor". When the zip is reimported into qri, we must convert the body from json back to cbor.

### Body format in single-file exports

It is an error to attempt to export a single dataset with an inline body if the body format does not match the dataset format. In order words, a dataset exported as a single json file (the default case described earlier in this document) should be assumed to have `--format-body json`. If some explicit value is set, it must match the top-level dataset format, otherwise export must fail with an error message.

Since there is both a `--format` option to talk about the export's top-level format, and also a `--format-body` flag to do the same for the body, if only the body is being exported (using `--section body`), the `--format` flag should be allowed as shorthand to refer to `--format-body`. In other words, `--section body --format json` actually implies `--format-body json`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Given the proliferation of options, the clearest way to understand how they are resolved is to have a high level view comparing the effects of important combinations. Here is a table view:

| options        | ds format | body format | output filename   | notes |
| -------------- | --------- | ----------- | ----------------- | ------- |
| default        | json      | json        | [repo]\_[ts].json | body convert to json |
| --section body | -         | st.format   | [repo]\_body\_[ts].[format] | - |
| --zip          | json      | st.format   | [repo]\_[ts].zip  | zipped |
| --zip --section body | -   | st.format   | [repo]\_body\_[ts].[format] | zipped |
| --format yaml  | yaml      | base64      | [repo]\_[ts].yaml | - |
| --format-body json | json  | json        | [repo]\_[ts].yaml | body convert to json |
| --zip --format-body csv | json  | csv    | [repo]\_[ts].zip  | ds.format != body.format |
| --section body --format json | - | json  | [repo]\_body\_[ts].json | body convert to json |
| --o out.yaml   | yaml      | base64      | out.yaml          | - |
| --o out.json   | json      | json        | out.json          | body convert to json |
| --o out.zip    | json      | st.format   | out.zip           | zipped |

The tricky part of the implemention is resolving the implied, derived, and explicit options.

TODO: Add pseudocode


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

IPFS block exporting.
