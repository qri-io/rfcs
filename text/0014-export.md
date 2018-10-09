- Feature Name: Export Dataset
- Start Date: 2018-08-27
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Users need to be able to get data not only into Qri, but out of it as well.

# Motivation
[motivation]: #motivation 

In order for Qri to be useful to a large swath of folks, we need to allow users to get datasets out of it, thus providing an easy way to interop with other tools.

Our two methods for doing so are the `/export` api endpoint and the `qri export` command. Both methods should have the same options, level of control, and outputs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Exporting from Qri needs to provide a wide array of flexible options, in order to maximize the quantity of other tools it can interoperate with. As with any system, as the number of options increases, the complexity of how options combine with each other grows exponentially, so it's especially important that we define how each option works individually.

In addition, we wish to provide high-level "intelligent" options that work like "presets", setting reasonable low-level options in order to cover common cases of what users will often want to accomplish.

The options that `export` should allow include:

* What component of a dataset gets exported
	* header (meta, structure, commit)
	* body (full, or part)
	* transform
	* viz
* Format for the exported dataset header
	* json
	* yaml
	* none (no header)
* Format for the exported dataset body 	
    * json
    * yaml
	* csv
	* cbor
	* a non-qri format like xlsx, pdf, html
* Choices around encoding of data
    * line-endings
    * charset (usually utf-8)
* Miscellaneous options
    * file-format specific metadata (like putting the header into xlsx)
    * outputting a directory instead of zip (only for cmd-line)
    
### Component naming

The components of a dataset use a standard naming scheme. Exporting a single component will use this name by default.

| Component | Filename |
| ------------ |  --------- |
| header | "dataset.json" |
| body | "body.json" |
| transform script | "transform.sky" |
| viz script | "viz.html" |
| reference | "ref.txt" |

The "dataset.json" file contains the header components that describe the dataset: the meta, structure, and commit. The "ref.txt" file contains the path of the dataset on the distributed web, as this value clearly can't be stored in the dataset itself, but it necessary for re-importing.

### Zip format

If more than one file needs to be exported, Qri will create a single zip file containing a directory with the standardized filenames. This is pretty similar to how many modern file formats work, such as `.ods`.

### Embedded metadata

Some file formats include a location for including metadata. For example, html has the `<head>` element with `<meta>` tags, jpeg has EXIF chunks, and xlsx has custom properties. One of Qri's export options should be to write the metadata of a dataset into one of these file-specific locations, assuming the format is in use.

This would allow exporting a dataset to a single file, without losing any important metadata.

In such cases, it is also assumed that the metadata will include a cafs hash to the exported dataset, which will allow for easy parenting if the result gets reimported.

### Saving

The command-line tool will require a filename to save the export to. The frontend will prompt with a standard "Save As" dialog box.

### Blank

The command-line tool will also allow a "blank export" using the `--blank` flag to create an empty file of the requested type.

  - `qri export --blank header`
  - `qri export --blank transform`
  - `qri export --blank viz`
  
Blank files will be outputted to the standard name for their component. If a file with that name already exists, an error will be displayed.

### Importing

As explicit goal of the export process is that an exported dataset can be reimported in the future (possibly with modifications), and not lose any fidelity in the process. For example, all metadata should still be present, and a file that is not modified at all should be treated as an exact duplicate of the dataset that was exported.

Of course this won't always be possible if certain options are used, such as if the user requests the header not be exported, but assuming this isn't the case, information should not be lost.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The top-level `Export` method will be passed a struct that contains a number of other structs, each roughly corresponding to the categories of options mentioned above.

```
type ExportOptions struct {
  Section  ExportSectionOptions
  Format   ExportFormatOptions
  Encoding ExportEncodingOptions
  Misc     ExportMiscOptions
}
```

The exact contents of each of these substructs will be determined during development.

The command-line arguments corresponding to these substructs are either simple string values, or use a hyphenated name if the struct has multiple fields. Here are some examples:

```
--section all
--section meta,body
--format-header yaml
--format-body json
--format-file xlsx
--encoding-charset utf-8
--metadata-in-file true
--directory-of-files true
```

The final example "metadata-in-file" is part of the `Misc` struct.

### URL encoding

The process for converting from command-line arguments to a URL used by the frontend is as simple as separating each argument with an ampersand, and putting an equals sign between each key and value.

```
qri export --sections all \
           --format-header yaml \
           --format-body json
```

would have this corresponding URL:

```
https://localhost:2503/export?sections=all&format-header=yaml&format-body=json
```

### Presets

For the high-level "preset" style flags, the export command will use the `preset` flag, like this:

```
qri export me/my-dataset --preset html save_to.html
qri export me/my-dataset --preset excel save_to.xlsx
```

There is no struct corresponding to the `--preset` flag. Rather, a given value of this flag will translate into a specific instance of the ExportOptions structure, using reasonable defaults for the requested output. These instances are encoded in sourcefiles, so using `--preset excel` will deserialize this instance and pass it to the Export method.

For instance, `--preset excel` may be treated like:

```
qri export --sections all \
           --format-file xlsx \
           --option-metadata-in-file true
```

# Drawbacks
[drawbacks]: #drawbacks

There is inherent complexity around this feature. However, we believe the benefits of having a strong interop story outway the cost of the added complexity. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative to this feature would be to just give users raw data, in a Qri-specific format, and ask them to write their own conversion tools. We have chosen not to take this approach in order to hopefully encourage others to use Qri.

# Prior art
[prior-art]: #prior-art

Export is a common feature is nearly all document editing and data handling applications. Often it may use the name "Save As" with conversion options.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

How many options will need to be supported in the "Misc" section?

Is there a better way to divide up the structure that represents all ExportOptions?