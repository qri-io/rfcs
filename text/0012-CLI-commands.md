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

We are using the cobra package to create each command.

Each command has a struct `[name_of_command]Options` (eg `ValidateOptions`)

Each option struct requires `Complete`, `Validate`, and `Run` methods.

The `Complete` method fills in any parameters needed for the command to run. It 
also adds the command arguments to the option struct.

The `Validate` methods ensures that the parameters added to the option struct
are all valid, and returns an error describing the problem if the options are
invalid.

The `Run` method calls to the more core Qri functions to run the command using
the parameters in the options struct.

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

<a id='qri_info'></a>
## qri info

Show summarized description of a dataset

### Synopsis

Info describes datasets. By default, it will return the peername, dataset name, 
the network, the dataset hash, the file size, the length of the datasets, 
and the validation errors.

Using the `--format` flag, you can get output in json. This will return a json
representation of the dataset, without the dataset body, identical to 
`qri get --format json`.

To get info on a peer's dataset, you must be running `qri connect` in a separate 
terminal window.

```
qri info [flags] <dataset_reference>
```

### Examples

```
  # get info for my dataset:
  qri info me/annual_pop

  # get info for a dataset at a specific version:
  qri info me@/ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn

  or

  qri info me/comics@/ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn

  # get info in json format
  qri info -f json me/annual_pop

  # to get info on a peer's dataset, spin up your qri node
  qri connect

  # then, in a separate window, request the info from peer b5
  qri info b5/comics
```

### Options

```
  -f, --format string   set output format [json]
  -h, --help            help for info
```



<a id='qri_list'></a>
## qri list

Show a list of datasets

### Synopsis


List shows lists of datasets, including names and current hashes. 

The default list is the latest version of all datasets you have on your local 
qri repository.

When used in conjuction with `qri connect`, list can list a peer's dataset. You
must have `qri connect` running in a separate terminal window.

```
qri list [flags]
```

### Examples

```
  # show all of your datasets:
  qri list

  # to view the list of your peer's dataset,
  # in one terminal window:
  qri connect

  # in a separate terminal window, to show all of b5's datasets:
  qri list b5
```

### Options

```
  -f, --format string   set output format [json]
  -h, --help            help for list
  -l, --limit int       limit results, default 25 (default 25)
  -o, --offset int      offset results, default 0
```



<a id='qri_log'></a>
## qri log

show log of dataset history

### Synopsis


log prints a list of changes to a dataset over time. Each entry in the log is a 
snapshot of a dataset taken at the moment it was saved that keeps exact details 
about how that dataset looked at at that point in time. 

We call these snapshots versions. Each version has an author (the peer that 
created the version) and a message explaining what changed. Log prints these 
details in order of occurrence, starting with the most recent known version, 
working backwards in time.

```
qri log [flags]
```

### Examples

```
  show log for the dataset b5/precip:
  $ qri log b5/precip
```

### Options

```
  -h, --help         help for log
  -l, --limit int    limit results, default 25 (default 25)
  -o, --offset int   offset results, default 0
```



<a id='qri_new'></a>
## qri new

Create a new dataset

### Synopsis


New creates a dataset from data you supply. Please note that all data added
to Qri is made public on the distributed web when you run qri connect

When adding data, you can supply metadata and dataset structure, but it’s not
required. qri does what it can to infer the details you don’t provide.
qri currently supports three data formats:
- CSV  (Comma Separated Values)
- JSON (Javascript Object Notation)
- CBOR (Concise Binary Object Representation)

Once you’ve added data, you can use the export command to pull the data out of
Qri, change the data outside of Qri, and use the save command to record those
changes to Qri.

Datasets added to Qri using `new` are published to the registry by default.
Use the `--no-publish` flag to not publish this dataset to the registry. For
more information on the registry and why it's important check out the [registry
section](#qri_registry).

```
qri new [flags]
```

### Examples

```
  create a new dataset named annual_pop:
  $ qri new --body data.csv me/annual_pop

create a dataset with a dataset data file:
  $ qri new --file dataset.yaml --body comics.csv me/comic_characters
```

### Options

```
  -b, --body string       path to file or url for contents of dataset
  -f, --file string       dataset data file in either yaml or json format
  -h, --help              help for new
  -m, --message string    commit message
      --private           make dataset private. WARNING: not yet implimented. Please refer to https://github.com/qri-io/qri/issues/291 for updates
  -p, --no-publish        do not publish this dataset to the registry
      --secrets strings   transform secrets as comma separated key,value,key,value,... sequence
  -t, --title string      commit title
```



<a id='qri_peers'></a>
## qri peers

Commands for working with peers

### Synopsis

The `peers` commands allow you to interact with other peers on the Qri network.
In order for these commands to work, you must be running a Qri node. This 
node allows you to communicate on the network. To spin up a Qri node, run
`qri connect` in a separate terminal. This will connect you to the network, 
until you choose to close the connection by ending the session or closing 
the terminal.

### Options

```
  -h, --help   help for peers
```



<a id='qri_peers_connect'></a>
## qri peers connect

Connect to a specific peer

### Synopsis

Connect to a peer using a peername, peer ID, or multiaddress. Qri will use this name, id, or address
to find a peer to which it has not automatically connected. 

You must have a Qri node running (`qri connect`) in a separate terminal. You will only be able 
to connect to a peer that also has spun up it's own Qri node.

A multiaddress, or multiaddr, is the most specific way to refer to a peer's location, and is therefore
the most sure-fire way to connect to a peer. 

```
qri peers connect <peername, peer_id, or multiaddr>
```

### Example

```
  # spin up a Qri node
  qri connect

  # in a separate terminal, connect to a specific peer using a multiaddr
  qri peers connect /ip4/192.168.0.194/tcp/4001/ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
```

### Options

```
  -h, --help   help for connect
```


<a id='qri_peers_disconnect'></a>
## qri peers disconnect

Explicitly close a connection to a peer

### Synopsis

Explicitly close a connection to a peer using a peername, peer id, or multiaddress. 

You can close all connections to the Qri network by ending your Qri node session. 

Use the disconnect command when you want to stay connected to the network, but want to 
close your connection to a specific peer. This could be because that connection is hung,
the connection is pulling too many resources, or because you simply no longer need an
explicit connection.  This is not the same as blocking a peer or connection.

Once you close a connection to a peer, you or that peer can immediately open another 
connection.

You must have `qri connect` running in another terminal.


```
qri peers disconnect <peername, peer_id, or multiaddr>
```

### Examples

```
  # disconnect from a peer using a multiaddr
  qri peers disconnect /ip4/192.168.0.194/tcp/4001/ipfs/QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn
```

### Options

```
  -h, --help   help for disconnect
```



<a id='qri_peers_info'></a>
## qri peers info

Get info on a Qri peer

### Synopsis


The peers info command returns a peer's profile information. The default
format is yaml.

Using the `--verbose` flag, you can also view a peer's network information.

You must have `qri connect` running in another terminal.

```
qri peers info [flags] <peername>
```

### Examples

```
  # spin up a Qri node in a separate terminal
  qri connect

  # then 
  # show info on a peer named "b5":
  qri peers info b5

  # show info in json:
  qri peers info b5 --format json
```

### Options

```
      --format string   output format. formats: yaml, json (default "yaml")
  -h, --help            help for info
  -v, --verbose         show verbose profile info
```



<a id='qri_peers_list'></a>
## qri peers list

List known qri peers

### Synopsis


Lists the peers to which your Qri node is connected. 

You must have `qri connect` running in another terminal.

To find include peers that are not online, but to which your node has previously been 
connected, use the `--cached` flag.

```
qri peers list [flags]
```

### Examples

```
  # spin up a Qri node
  qri connect

  # thenin a separate terminal, to list qri peers:
  qri peers list

  # to ensure you get a cached version of the list:
  qri peers list --cached
```

### Options

```
  -c, --cached           show peers that aren't online, but previously seen
  -h, --help             help for list
  -l, --limit int        limit max number of peers to show (default 200)
  -n, --network string   specify network to show peers from [ipfs]
  -s, --offset int       number of peers to skip during listing
```



<a id='qri_registry'></a>
## qri registry

commands for working with a qri registry

### Synopsis

Registries are federated public records of datasets and peers.
These records form a public facing central lookup for your datasets, so others
can find them through search tools and via web links. You can use registry 
commands to control how your datasets are published to registries, opting out
on a dataset-by-dataset basis.

By default qri is configured to publish to https://registry.qri.io,
the main public collection of datasets & peers. "qri new" and "qri update"
default to publishing to a registry as part of dataset creation unless run 
with the "no-publish" flag. Published datasets that are under 20MBs will be
backed up by the Qri registry.

Unpublished dataset info will be held locally so you can still interact
with it. And your datasets will be available to others peers when you run 
"qri connect", but will not show up in search results, and will not be 
displayed on lists of registry datasets.

Qri is designed to work without a registry should you want to opt out of
centralized listing entirely, but know that peers who *do* participate in
registries may choose to deprioritize connections with you. Opting out of a
registry entirely is better left to advanced users.

You can opt out of registries entirely by running:
$ qri config set registry.location ""

### Options

```
  -h, --help   help for registry
```



<a id='qri_registry_publish'></a>
## qri registry publish

Publish dataset info to the registry

### Synopsis

Publishes the dataset information onto the registry. There will be a record
of your dataset on the registry, and if your dataset is less than 20mbs, 
Qri will back your dataset up onto the registry.

Published datasets can be found by other peers using the `qri search` command.

Datasets are by default published to the registry when they are created.

```
qri registry publish [flags] <dataset_reference>
```

### Examples

```
  Publish a dataset you've created to the registry:
  $ qri registry publish me/dataset_name
```

### Options

```
  -h, --help   help for publish
```



<a id='qri_registry_unpublish'></a>
## qri registry unpublish

Remove dataset info from the registry

### Synopsis

Unpublish will remove the reference to your dataset from the registry. If 
you dataset was previously backed up onto the registry, this backup will 
be removed.

This dataset will no longer show up in search results. 

```
qri registry unpublish [flags] <dataset_reference>
```

### Examples

```
  Remove a dataset from the registry:
  $ qri registry unpublish me/dataset_name
```

### Options

```
  -h, --help   help for unpublish
```



<a id='qri_remove'></a>
## qri remove

remove a dataset from your local repository

### Synopsis


Remove gets rid of a dataset from your Qri node. After running remove, Qri will 
no longer list your dataset as being available locally. By default, remove frees
up the space taken up by the dataset, but not right away. The IPFS repo that’s 
storing the data will need to garbage-collect that data when it’s good & ready, 
which could be anytime. If you’re running low on space, garbage collection will 
be sooner. 

Keep in mind that by default your IPFS repo is capped at 10GB in size, if you
adjust this cap using IPFS, Qri will respect it.

Use the `--free` flag to force immediate removal of a dataset from both qri & 
IPFS to free up the space that was taken up by that dataset.

```
qri remove [flags] <dataset_reference>
```

### Examples

```
  remove a dataset named annual_pop:
  $ qri remove me/annual_pop
```

### Options

```
  -h, --help   help for remove
  -f, --free   immediately free up space that was taken up by this dataset
```



<a id='qri_rename'></a>
## qri rename

Change the name of a dataset

### Synopsis


Rename changes the name of a dataset.

Note that if someone has added your dataset to their qri node, and then
you rename your local dataset, your peer's version of your dataset will
not have the updated name. While this won't break anything, it will
confuse anyone who has added your dataset before the change. Try to keep
renames to a minimum.

```
qri rename [flags] <dataset_reference>
```

### Examples

```
  rename a dataset named annual_pop to annual_population:
  $ qri rename me/annual_pop me/annual_population
```

### Options

```
  -h, --help   help for rename
```



<a id='qri_render'></a>
## qri render

Execute a template against a dataset

### Synopsis

You can use html templates, formatted in the go/html template style, 
to render visualizations from your dataset. These visualizations can be charts, 
graphs, or just display your dataset in a different format.

Use the `--output` flag to save the rendered html to a file.

Use the `--template` flag to use a custom template. If no template is
provided, Qri will render the dataset with a default template.

```
qri render [flags] <dataset_reference>
```

### Examples

```
  render a dataset called me/schools to the file school.html:
  $ qri render -o=schools.html me/schools

  render a dataset with a custom template:
  $ qri render --template=template.html me/schools
```

### Options

```
  -a, --all               read all dataset entries (overrides limit, offest)
  -h, --help              help for render
  -l, --limit int         max number of records to read (default 50)
  -s, --offset int        number of records to skip
  -o, --output string     path to write output file
  -t, --template string   path to template file
```



<a id='qri_save'></a>
## qri save

Save changes to a dataset

### Synopsis


Save is how you change a dataset, updating one or more of data, metadata, and 
structure. You can also update your data via url. Every time you run save, 
an entry is added to your dataset’s log (which you can see by running `qri log 
<dataset_reference>`). Every time you save, you can provide a message about what 
you changed and why. If you don’t provide a message 
Qri will automatically generate one for you.

When you make an update and save a dataset that you originally added from a different
peer, the dataset gets renamed from `peers_name/dataset_name` to `my_name/dataset_name`.

The `--message` and `--title` flags allow you to add a commit message and title 
to the save.

Use `--no-publish` to not publish this version of the dataset to the Qri registry.

```
qri save [flags] <dataset_reference>
```

### Examples

```
  save updated data to dataset annual_pop:
  $ qri --body /path/to/data.csv me/annual_pop

  save updated dataset (no data) to annual_pop:
  $ qri --file /path/to/dataset.yaml me/annual_pop
```

### Options

```
      --body string       path to file or url of data to add as dataset contents
  -f, --file string       dataset data file (yaml or json)
  -h, --help              help for save
  -m, --message string    commit message for save
  -p, --no-publish        do not publish this dataset to the registry
      --secrets strings   transform secrets as comma separated key,value,key,value,... sequence
  -t, --title string      title of commit message for save
```



<a id='qri_search'></a>
## qri search

Search qri nodes for datasets.

### Synopsis

Search datasets & peers that match your query. Search pings the qri registry. Any dataset 
that has been published to the registry is available for search.

Search will match datasets based on the meta title, keyword, and theme.

```
qri search [flags] <search term>
```

### Examples

```
  # search 
  $ qri search "annual population"
```

### Options

```
  -f, --format string   set output format [json]
  -h, --help            help for search
```



<a id='qri_setup'></a>
## qri setup

initialize qri and IPFS repositories, provision a new qri ID

### Synopsis


Setup is the first command you run to get a fresh install of Qri. If you’ve 
never run qri before, you’ll need to run setup before you can do anything. 

Setup does a few things:
- create a qri repository to keep all of your data
- provisions a new qri ID
- create an IPFS repository if one doesn’t exist

This command is automatically run if you invoke any Qri command without first 
running setup. If setup has already been run, by default Qri won’t let you 
overwrite this info.

Use the `--remove` to remove your Qri repo. This deletes your entire repo, 
including all your datasets, and de-registers your peername from the registry.

```
qri setup [flags]
```

### Examples

```
  run setup with a peername of your choosing:
  $ qri setup --peername=your_great_peername
```

### Options

```
  -a, --anonymous            use an auto-generated peername
      --config-data string   json-encoded configuration data, specify a filepath with '@' prefix
  -h, --help                 help for setup
      --init-ipfs            initialize an IPFS repo if one isn't present (default true)
      --ipfs-config string   json-encoded configuration data, specify a filepath with '@' prefix
      --overwrite            overwrite repo if one exists
      --peername string      choose your desired peername
      --registry string      override default registry URL
      --remove               permanently remove qri, overrides all setup options
```



<a id='qri_use'></a>
## qri use

Select datasets for use with other commands

### Synopsis

Select datasets for use with other commands. 

Run the `use` command to 'save' a reference to a specific dataset. This dataset
will be referenced for future commands, if no dataset reference is explicitly
given for those commands.

We created this command to ease the typing/copy and pasting burden while using
Qri to explore a dataset.

Dataset references saved using the `use` command will work with:

body
diff
export
get
info
log
render
validate

```
qri use [flags]
```

### Examples

```
  # use dataset me/dataset_name, then get meta.title:
  qri use me/dataset_name
  qri get meta.title

  # clear current selection:
  qri use --clear

  # show current selected dataset references:
  qri use --list
```

### Options

```
  -c, --clear   clear the current selection
  -h, --help    help for use
  -l, --list    list selected references
```



<a id='qri_validate'></a>
## qri validate

Show schema validation errors

### Synopsis


Validate checks data for errors using a schema and then printing a list of 
issues. By default validate checks a dataset's body against it’s own schema. 
Validate is a flexible command that works with data and schemas either 
inside or outside of qri by providing one or both of --body and --schema 
arguments. 

Providing --schema and --body is an “external validation" that uses nothing 
stored in qri. When only one of schema or body args are provided, the other 
comes from a dataset reference. For example, to check how a file “data.csv” 
validates against a dataset "foo”, we would run:

  $ qri validate --body data.csv me/foo

In this case, qri will will print any validation as if data.csv was foo’s data.

To see how changes to a schema will validate against a 
dataset in qri, we would run:

  $ qri validate --schema schema.json me/foo

In this case, qri will print validation errors as if stucture.json was the
schema for dataset "me/foo"

Using validate this way is a great way to see how changes to data or schema
will affect a dataset before saving changes to a dataset.

You can get the current schema of a dataset by running the `qri get structure.schema`
command.

Note: --body and --schema flags will override the dataset if both flags are provided.

```
qri validate [flags]
```

### Examples

```
  # show errors in an existing dataset:
  qri validate b5/comics

  # validate a new body against an existing schema
  qri validate --body new_data.csv me/annual_pop

  # validate data against a new schema
  qri validate --body data.csv --schema schema.json
```

### Options

```
  -b, --body string     data file to initialize from
  -h, --help            help for validate
      --schema string   json schema file to use for validation
```



<a id='qri_version'></a>
## qri version

Print the version number

### Synopsis

Qri uses semantic versioning.

For updates & further information check https://github.com/qri-io/qri/releases

```
qri version [flags]
```

### Options

```
  -h, --help   help for version
```


# Drawbacks
[drawbacks]: #drawbacks

Structuring qri around a command line client as our base implementation will mean that CLI features routinely outpace other ways of interacting with qri (like the API and subsequently the frontend). To mitigate this it's important that we view any proposed feature from perspectives other than just the CLI, and work to maintain parity across our outward facing interfaces wherever possible & reasonable.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->

This is less a section about rationale and alternatives of why change to this structure, and more an explanation why we found this refactoring so positive.

The `Complete`, `Validate`, `Run`, structure of our commands is taken from [kubernetes](https://github.com/kubernetes/kubernetes/tree/master/cmd/kubeadm/app/cmd). This allows us to test each part of the command individually. When we refactored from our previous structure, to this current structure, our testing and the clarity of our error messages jumped up.


# Prior art
[prior-art]: #prior-art

[kubernetes](https://github.com/kubernetes/kubernetes/tree/master/cmd/kubeadm/app/cmd)

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
- What related issues do you consider out of scope for this RFC that could b -->
I assume that these commands will go through many orientations as we learn how peers use Qri and other functionalities they might need.
