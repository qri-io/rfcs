- Feature Name: configuration overhaul
- Start Date: [#51](https://github.com/qri-io/rfcs/pulls/51)
- RFC PR: 2020-04-23
- Issue: 

# Summary
[summary]: #summary

Revise configuration fields with clear descriptions, removing unused fields, and defining new fields to cover existing configuration. Define a hierarchy for configuration overrides.

# Motivation
[motivation]: #motivation

Our configuration story needs work. _Many_ fields are not in use, and there's ambiguity about how others should work. Cleaning up our configuration story should open up new use cases & cut down on confusion.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC proposes the following concrete changes:
1. Overhaul configuration data structures, removing unnecessary fields & adding new ones
1. Support configuring `qfs.Filesystem`
1. Formalize the hierarchy of different configuration sources
1. Switch to multiaddresses for all network configuration.
1. Move ipfs repo location into configuration, deprecate support for `IPFS_PATH`.

### Updated Config File

This RFC proposes a new configuration layout with the following fields, values should show defaults. Many fields are removed. New and altered fields have a comment.

```yaml
API:
  allowedorigins:
  - electron://local.qri.io
  - http://localhost:2505
  enabled: true
  address: "/ip4/0.0.0.0/tcp/2503" # replaces the "port" field with a multiaddr
  readonly: false
  serveremotetraffic: false
CLI:
  colorizeOutput: true
  pager: ""
  warnBeforeFetchingSize: 250MB    # any fetch larger than this will prompt the user before continuing
Filesystems:                       # see note on filesystems
- type: ipfs
  options:
    path: ./ipfs
    api: true
    pubsub: false
- type: local
- type: http
Logging:
  levels:
    lib: info
P2P:
  addrs: null                     # an array of multiaddresses to listen on, only if *not* using IPFS
  enabled: true
  peerid: QmdJQpcmGMkNtKYv5bviZ7z9kPh7uLKSMzqdSbemYs3XpH
  privkey: <base64-encoded private key>
  qribootstrapaddrs:
  - /ip4/35.193.162.149/tcp/4001/ipfs/QmTZxETL4YCCzB1yFx4GT1te68henVHD1XPQMkHZ1N22mm
  - /ip4/35.239.80.82/tcp/4001/ipfs/QmdpGkbqDYRPCcwLYnEm8oYGz2G9aUZn9WwPjqvqw3XUAc
  - /ip4/35.225.152.38/tcp/4001/ipfs/QmTRqTLbKndFC2rp6VzpyApxHCLrFV35setF1DQZaRWPVf
  - /ip4/35.202.155.225/tcp/4001/ipfs/QmegNYmwHUQFc3v3eemsYUVf3WiSg4RcMrh3hovA5LncJ2
  - /ip4/35.238.10.180/tcp/4001/ipfs/QmessbA6uGLJ7HTwbUJ2niE49WbdPfzi27tdYXdAaGRB4G
  - /ip4/35.238.105.35/tcp/4001/ipfs/Qmc353gHY5Wx5iHKHPYj3QDqHP4hVA1MpoSsT6hwSyVx3r
  - /ip4/35.239.138.186/tcp/4001/ipfs/QmT9YHJF2YkysLqWhhiVTL5526VFtavic3bVueF9rCsjVi
  - /ip4/35.226.44.58/tcp/4001/ipfs/QmQS2ryqZrjJtPKDy9VTkdPwdUSpTi1TdpGUaqAVwfxcNh
Profile:
  color: ""
  created: "2020-02-12T04:29:18Z"
  description: ""
  email: brendan+test@qri.io
  homeurl: ""
  id: QmSyDX5LYTiwQi861F5NAwdHrrnd1iRGsoEvCyzQMUyZ4W
  name: ""
  peername: b5
  photo: https://qri-user-images.storage.googleapis.com/1570029824064.png
  poster: /ipfs/QmSN2yHp4qFy1xLCRVQNJKAjgbwHYPAho9fT7C3vAM426c
  privkey: <base64-encoded private key>
  thumb: ""
  twitter: ""
  type: peer
  updated: "2018-04-19T18:10:46.627471799-04:00"
RPC:
  enabled: true
  address: "/ip4/0.0.0.0/tcp/2504" # replaces "port" with a multiaddr
Registry:
  location: https://registry.qri.cloud
Remote:
  acceptsizemax: 0
  accepttimeoutms: 0
  allowremoves: false
  enabled: false
  requireallblocks: false
Remotes:
  remote_name: https://remote_name_domain.com
Repo:
  type: fs
Revision: 2
```

### The Filesystems field

Qri is built atop a filesystem abstraction called [qfs](https://github.com/qri-io/qfs) that defines a common interface with numerous implementations. At runtime, qri composes a number of these filesystems together to provide access to HTTP assets, local files, and IPFS data. In qri v0.9.8, the `qfs.Filsystem` instance cannot be configured.

This RFC proposes a format for configuring the qfs.Filesystem. Making qfs configurable opens the doors to new backing filesystems like Amazon S3, opening the door to operating Qri in new contexts entirely through configuration.

We accomplish this by removing the `Store` field and replacing it with `Filesystems`. The default configuration used in qri v0.9.8 would look like this:

```yaml
Filesystems:
- type: ipfs
  options:
    path: ~/.ipfs
    api: true
    pubsub: false
- type: local
- type: http
```

Filesystems is an array of objects, with each object representing a filesystem. each object MUST have a `type` parameter that matches the name of a known system. Any system-specific configuration details can be set in `options`, an object that will be passed onto the Filesystem itself. Finally an optional `source` key tells Qri where to load fileystem plugins from:

```yaml
Filesystems:
- type: s3
  source: https://github.com/qri-io/s3-filesystem/releases/v0.9.8.tar
  options:
    access_token: TOKEN
- type: http
```

The source value is itself a filesystem path. In this case we're loading over https, which would necessitate the `http` filesystem for this plugin to be able to load. Any plugin architecture is a ways off, and should be specified in a later RFC.

### Use multiaddrs, port 0 for random open port
We should switch all `port` fields to an `address` field instead, and accept a [multiaddr](https://github.com/multiformats/go-multiaddr) string to enhance configurability.

In addition, we should support the convention of using `0` as a port number to indicate "any available port", allowing the operating system to choose.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Config Hierarchy

Configuration details should be unified into a single data structure loaded in a predefined hierarchy. From least that hierarchy should be:

* default configuration
* qri repo configuration file
* global command-line flags

We can imagine the final computed configuration as a _fold-left_ operation:

```
config = fold_left(defaultConfig(), ConfigFile(), CliFlags())
```

Any qri process loads at most 1 configuration file. The source of that file defaults to `$HOME/.qri/config.yaml`, and can be overridden with the `QRI_PATH` environment variable.

In the event that a command line flag and a configuration setting are affecting the same behaviour, the
command line flag wins, overriding any configuration setting. With that said, **flags should change parameters on lib method calls, not affect configuration**.


### Config in alternate contexts:

**qri CLI commands that work without a repo**
Some commands like `qri setup` have to operate without a `lib.Instance`, and _cannot_ depend on a file, so the loading hierarchy simply removes the repoConfig() from the hierarchy, leaving the rest intact:

```
config = fold_left(defaultConfig(), getEnvVarConfig(), getCLIConfig())
```

**using qri as a library**
When using qri as a library, we don't have the `cmd` package, and often supply configuration details directly.

```
config = fold_left(defaultConfig(), getUserOfLibraryConfig())
```

### Remove stale configuration fields

The following fields should be **removed**:

```yaml
API:
  proxyforcehttps: false
  remoteacceptsizemax: 0
  remoteaccepttimeoutms: 0
  remotemode: false
  tls: false
  urlroot: ""
P2P:
  autoNAT: false
  profilereplication: full
  bootstrapaddrs: []
  httpgatewayaddr: ""
  pubkey: ""
Repo:
  middleware: []
Webapp:
  analyticstoken: ""
  enabled: true
  entrypointhash: /ipfs/QmXofmXcQKnYTMjsfhgTGqzzLLPyS9enaHsfaRR5AxTGBK
  entrypointupdateaddress: /ipns/webapp.qri.io
  port: 2505
Render:
  defaultTemplateHash: /ipfs/QmeqeRTf2Cvkqdx4xUdWi1nJB2TgCyxmemsL3H4f1eTBaw
  templateUpdateAddress: /ipns/defaulttmpl.qri.io
Stats: null
Update:
  address: ""
  daemonize: false
  type: fs
```

### Expanding Store to Filesystems

Currently we hard-code qri filesystem (`qfs`) configuration details. qfs spins up a multiplexed-filesystem by default, composing multiple file-like persistence layers using prefix demuxing. If we expand "Store" to "Filesystem", we can make this fully configurable.

The first win: users could remove `local` or `http` to disable qri's access to the local filesystem or HTTP for the purpose of reading data. We can also expand for later filesystems & options. an `s3` implementation would be nice. this is also how we'd do `dat` integration.

We could use a convention of the default write destination being the first listed filesystem, and present a big warning if users configure anything other than `ipfs` in the first position.

It'd be great to delegate this entire section of store setup to the qfs package, aliasing up the configuration data structure into the config package.

### Moving past cafs

Config has a "Store" field that configured a _content addressed file system_ (CAFS), cafs is the precursor to `qfs.Filesystem`, and is in the process of being merged into this API. The `Filesystems` field is used to consutrct a `qfs.Muxfs` multiplexing filesystem at runtime. Muxfs will provide methods for accessing individual filesystems, where ones like `ipfs` still satisfy the `cafs` interface. It'll be possible to progressively migrate code that requires a `cafs`  toward a `qfs.Filesystem` while we work on configuration changes.

### move IPFS repo location into configuration
the above filesystem example shows a `ipfs` filesystem type with a `config.path` field. This value should now be the canonical source of where to load.

the global `ipfs-path` flag should also be removed:
```
      --ipfs-path string   override IPFS path location (default "/Users/b5/.ipfs")
```

### Relative Paths in configuration files are relative to the file
If a configuration value specifies a filepath, and the given path is relative, that path should be relative to the location of the config file. We should add a note of this to `$ qri config --help`.

### Environment Variables don't configure, they point to configuration files

This is the total list of environment variables qri is sensitive to:

| variable name            | description                           |
| ------------------------ | ------------------------------------- |
| `PAGER`                  | what pager to use for output          |
| `QRI_PATH`               | path to qri configuration data        |
| `QRI_SETUP_CONFIG_DATA`  | JSON data to use when creating a repo |
| `IPFS_SETUP_CONFIG_DATA` | JSON data when initializing IPFS      |
| `QRI_PRIVATE_KEY`        | defined in the private key RFC        |
| `QRI_BACKTRACE`          | show stack trace output on crash      |


_note: I (b5) haven't had a chance to vet our dependencies for env vars they are sensitive to, so this list is only complete for the purpose of this RFC._

This RFC proposes environment variables are _not_ a direct source of configuration. Instead environment variables either point to a path to load configuration from, or contain a payload to create an initial configuration file. 

`QRI_SETUP_CONFIG_DATA` as the only supported is only honored by `qri setup`. We need `SETUP` env variables for operating within containerized contexts, but that's about it.

Directing as much configuration as possible to a configuration file cuts down on potential sources of confusion. As an example, `git` will listen to `$PAGER`, `$GIT_PAGER`, and `git config core.pager`, in that order, and I (b5) had to look up what the order is.

Instead of supporting a potential `QRI_PAGER` environment variable, we should _only_ have `qri.cli.pager` to override the default value.

All that said, using env var configuration should be _optional_, meaning a user employing qri as a library can provide an option to the `lib.NewInstance` constructor function that ignores all environment variables. Put another way: environment variables never affect configuration, which is a lib-level concept, but as a caller of lib, the command-line client `qri` is able to use them in order to create the configuration that it wants.

# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Per-field env vars
We could support setting configuration on a per-field basis. I'm not sure that's totally necessary at this point.

### Attach config to context.Context
We could build a sophisticated system of attaching configuration to request contexts, to support scoping configuration on a per-request basis. We might need to in the future, but it's overkill for the moment.

### Config changes emit events
We could also use the event bus to propagate changes to configuration. Again, overkill for now, but we could do so later.

# Prior art
[prior-art]: #prior-art

### Configuration:
awesome-go has an entire section of configuration libraries https://awesome-go.com/#configuration. Some of the ones I found intersting:
* hot-reloading config: https://github.com/lalamove/konfig
* env var config with lots of bells& whistles: https://github.com/kelseyhightower/envconfig

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Some configuration things are out of scope for this RFC, but should be discussed later:
* Distinguish runtime _options_ from static _configuration_.
* hot-swapping configuration.