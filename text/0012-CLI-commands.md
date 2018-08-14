- Feature Name: CLI commands
- Start Date: 2018-08-13
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

The Qri CLI is how users can interact with Qri. It allows you to add, update, 
and explore datasets, as well as connected to the Qri network to interact with other Qri peers.

# Motivation
[motivation]: #motivation

This RFC is a place to document the prefered behaviors of the Qri CLI. By 
knowing the prefered behaviors, the Qri team can better and faster design new 
features, squash bugs, and answer questions. 

Each command will be documented with summary, flags, and examples.

Most commands require a dataset reference. A dataset reference is a naming 
convention for datasets. You can read about it in [RFC: dataset naming](/text/0006-dataset_naming.md).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Qri CLI commands are as follows:

* [qri add](#qri_add)  - Add a dataset from another peer
* [qri body](#qri_body)  - Get the body of a dataset
* [qri config](#qri_config)  - Get and set local configuration information
* [qri connect](#qri_connect)  - Connect to the distributed web, start a local API server
* [qri diff](#qri_diff)  - Compare differences between two datasets
* [qri export](#qri_export)  - Copy datasets to your local filesystem
* [qri get](#qri_get)  - Get elements of qri datasets
* [qri help](#qri_help) - Lists all the qri commands 
* [qri info](#qri_info)  - Show summarized description of a dataset
* [qri list](#qri_list)  - Show a list of datasets
* [qri log](#qri_log)  - Show log of dataset history
* [qri new](#qri_new)  - Create a new dataset
* [qri peers](#qri_peers)  - Commands for working with peers
* [qri registry](#qri_registry)  - Commands for working with a qri registry
* [qri remove](#qri_remove)  - Remove a dataset from your local repository
* [qri rename](#qri_rename)  - Change the name of a dataset
* [qri render](#qri_render)  - Execute a template against a dataset
* [qri save](#qri_save)  - Save changes to a dataset
* [qri search](#qri_search)  - Search qri
* [qri setup](#qri_setup)  - Initialize qri and IPFS repositories, provision a new qri ID
* [qri use](#qri_use)  - Select datasets for use with other commands
* [qri validate](#qri_validate)  - Show schema validation errors
* [qri version](#qri_version)  - Print the version number

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<a id="qri_add"></a>
## qri add

Add a dataset from another peer

### Synopsis

Add retrieves a dataset owned by another peer and adds it to your repo. 
The dataset reference of the dataset will remain the same, including 
the name of the peer that originally added the dataset.

```
qri add <dataset_reference>
```

### Examples

```
  add a dataset named their_data, owned by other_peer:
  $ qri add other_peer/their_data
```

### Options

```
  -h, --help   help for add
```

<a id="qri_body"></a>
## qri body

Get the body of a dataset 

### Synopsis

`qri body` gets records from a dataset. Default is 50 records, starting 
from the beginning of the body. You can using the `--limit` and `--offset` 
flags to iterate through the dataset body. 

```
qri body [flags] <dataset_reference>
```

### Examples

```
  show the first 50 rows of a dataset:
  $ qri body me/dataset_name

  show the next 50 rows of a dataset:
  $ qri body --offset 50 me/dataset_name

  save the body as csv to file
  $ qri body -o new_file.csv -f csv me/dataset_name
```

### Options

```
  -a, --all             read all dataset entries (overrides limit, offest)
  -f, --format string   format to export. one of [json,csv,cbor] (default "json")
  -h, --help            help for body
  -l, --limit int       max number of records to read (default 50)
  -s, --offset int      number of records to skip
  -o, --output string   path to write to, default is stdout
```


<a id="qri_config"></a>
## qri config

Get and set local configuration information

### Synopsis


`qri config` encapsulates all settings that control the behaviour of qri.
This includes all kinds of stuff: your profile details; enabling & 
disabling different services; what kind of output qri logs to; 
which ports on qri serves on; etc.

Configuration is stored as a .yaml file kept at $QRI_PATH, or provided 
at CLI runtime via command a line argument.

For details on each config field checkout the [config README](https://github.com/qri-io/qri/blob/master/config/readme.md)

### Examples

```
  # get your profile information
  $ qri config get profile

  # set your api port to 4444
  $ qri config set api.port 4444

  # disable rpc connections:
  $ qri config set rpc.enabled false
```

### Options

```
  -h, --help   help for config
```

### SEE ALSO

* [qri config get](#qri_config_get)  - Get configuration settings
* [qri config set](#qri_config_set)  - Set a configuration option


<a id="qri_config_get"></a>
## qri config get

Get configuration settings

### Synopsis

`qri config get` outputs your current configuration file with private 
keys removed by default, making it easier to share your qri configuration 
settings.

You can get the entire config object, or subsections of the object.
You can get the particular parts of the config by using dot notation to
traverse the config object. For details on each config field checkout 
the [config README](https://github.com/qri-io/qri/blob/master/config/readme.md)

The --with-private-keys option will show private keys.
PLEASE PLEASE PLEASE NEVER SHARE YOUR PRIVATE KEYS WITH ANYONE. EVER.
Anyone with your private keys can impersonate you on qri.

```
qri config get [flags]
```

### Examples

```
  # get the entire config
  qri config get

  # get the config profile
  qri config get profile

  # get the profile description
  qri config get profile.description
```

### Options

```
  -c, --concise             print output without indentation, only applies to json format
  -f, --format string       data format to export. either json or yaml (default "yaml")
  -h, --help                help for config get
  -o, --output string       path to export to
      --with-private-keys   include private keys in export
```

### SEE ALSO

* [qri config](#qri_config)  - Get and set local configuration information


<a id="qri_config_set"></a>
## qri config set

Set configuration options

### Synopsis

`qri config set` allows you to set configuration options. You can set 
particular parts of the config by using dot notation to traverse the 
config object. 

While the `qri config get` command allows you to view the whole config,
or only parts of it, the `qri config set` command is more specific.

If the config object were a tree and each field a branch, you can only
set the leaves of the branches. In other words, the you cannot set a 
field that is itself an object or array. For details on each config 
field checkout the [config README](https://github.com/qri-io/qri/blob/master/config/readme.md)

```
qri config set [flags]
```

### Examples
```
  # set a profile description
  qri config set profile.description "This is my new description that I
  am very proud of and want displayed in my profile"

  # disable rpc communication
  qri config set rpc.enabled false
```

### Options

```
  -h, --help   help for set
```

### SEE ALSO

* [qri config](#qri_config)  - Get and set local configuration information


<a id="qri_connect"></a>
## qri connect

Connect to the distributed web by spinning up a Qri node

### Synopsis


While it’s not totally accurate, connect is like starting a server. 
Running connect will start a process and stay there until you exit the 
process (ctrl+c from the terminal, or killing the process using tools 
like activity monitor on the mac, or the aptly-named “kill” command). 
Connect does three main things:
- Connect to the qri distributed network
- Connect to IPFS
- Start a local API server

When you run connect you are connecting to the distributed web, 
interacting with peers & swapping data.

```
qri connect [flags]
```

### Options

```
      --api-port int           port to start api on
      --disable-api            disables api, overrides the api-port flag
      --disable-rpc            disables rpc, overrides the rpc-port flag
      --disable-webapp         disables webapp, overrides the webapp-port flag
      --disconnect-after int   duration to keep connected in seconds, 0 means run indefinitely
  -h, --help                   help for connect
      --read-only              run qri in read-only mode, limits the api endpoints
      --registry string        specify registry to setup with. only works when --setup is true
      --rpc-port int           port to start rpc listener on
      --setup                  run setup if necessary, reading options from environment variables
      --webapp-port int        port to serve webapp on
```

<a id="qri_diff"></a>
## qri diff

Compare differences between two datasets

### Synopsis


Diff compares two datasets from your repo and prints a representation 
of the differences between them.  You can specifify the datasets
either by name or by their hash. You can compare different versions of 
the same dataset.

```
qri diff [flags] <dataset_reference_1> <dataset_reference_2>
```

### Examples

```
  show diff between two versions of the same dataset:
  $ qri diff me/annual_pop@/ipfs/QmcBZoEQ7ot4UYKn1JM3gwd4LHorj6FJ4Ep19rfLBT3VZ8 
  me/annual_pop@/ipfs/QmVvqsge5wqp4piJbLArwVB6iJSTrdM8ZRpHY7fikASrr8

  show diff between two different datasets:
  $ qri diff me/population_2016 me/population_2017
```

### Options

```
  -d, --display string   set display format [reg|short|delta|detail]
  -h, --help             help for diff
```

<a id="qri_export"></a>
## qri export

Copy datasets from qri to your local filesystem

### Synopsis


Export gets datasets out of qri. By default it exports only the dataset body. 

To export to a specific directory, use the --output flag.

If you want an empty dataset that can be filled in with details to create a
new dataset, use --blank.

```
qri export [flags] <dataset_reference>
```
### Examples

```
  # export dataset
  qri export me/annual_pop

  # export without the body of the dataset
  qri export --no-body me/annual_pop

  # export to a specific directory
  qri export -o ~/new_directory me/annual_pop
```

### Options

```
      --blank                export a blank dataset YAML file, overrides all other flags except output
      --body-format string   format for dataset body. default is the original data format. options: json, csv, cbor
  -f, --format string        format for all exported files, except for body. yaml is the default format. options: yaml, json (default "yaml")
  -h, --help                 help for export
  -b, --no-body              don't include dataset body in export
  -o, --output string        path to write to, default is current directory
  -d, --peer-dir             export to a peer name namespaced directory
```


<a id="qri_get"></a>
## qri get

Get elements of qri datasets

### Synopsis

Get the qri dataset (except for the body). You can also get portions of 
the dataset: meta, structure, viz, transform, and commit. To narrow down
further to specific fields in each section, use dot notation. The get 
command prints to the console in yaml format, by default.

You can get pertinent information on multiple datasets at the same time
by supplying more than one dataset reference.

Check out the [dataset documentation](https://qri.io/docs/reference/dataset/) to learn about each section of the 
dataset and its fields.

```
qri get [flags] <dataset_reference> <dataset_reference2> ...
```

### Examples

```
  # print the entire dataset to the console
  qri get me/annual_pop

  # print the meta to the console
  qri get meta me/annual_pop

  # print the dataset body size to the console
  qri get structure.length me/annual_pop

  # print the dataset body size for two different datasets
  qri get structure.length me/annual_pop me/annual_gdp
```

### Options

```
      --concise         print output without indentation, only applies to json format
  -f, --format string   set output format [json, yaml] (default "yaml")
  -h, --help            help for get
```

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art

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

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could b