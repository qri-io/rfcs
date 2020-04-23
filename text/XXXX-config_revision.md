- Feature Name: configuration hierarchy
- Start Date: <!-- (fill me in with today's date, YYYY-MM-DD) -->
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Define a hierarchy for configuration overrides, revise configuration fields with clear descriptions.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Formalizing support for environment variables

```
TODO (b5) - we need to talk about how env vars are going to play into configuration
Currently we have `QRI_CONFIG_DATA` as the only supported environemnt variable, which is only honored at setup. Maybe it should stay as a single variable, and just be used all the time?
```

Using env var configuration should be _optional_,

### Config Hierarchy

Configuration details should be unified into a single data structure loaded in a predefined hierarchy. From least that hierarchy should be:

* default configuration
* qri repo configuration file
* environment variables
* command-line flags

We can imagine the final computed configuration as a _fold-left_ operation:

```
config = fold_left(defaultConfig(), repoConfig(), getEnvVarConfig(), getCLIConfig())
```

It's worth noting that

### Config in alternate contexts:

**qri CLI commands that work without a repo**
Some commands like `qri setup` have to operate without a `lib.Instance`, and _cannot_ depend on a file, so the loading hierachy simply removes the repoConfig() from the hierarchy, leaving the rest intact:

```
config = fold_left(defaultConfig(), getEnvVarConfig(), getCLIConfig())
```

**using qri as a library**
When using qri as a library, we don't have the `cmd` package, and often supply configuration details directly.

```
config = fold_left(defaultConfig(), getEnvVarConfig(), getCLIConfig())
```

### Determining path to a configuration file
We need a section describing default paths, 


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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

Currently we hard-code far to many qfs configuration details. qfs spins up a multiplexed-filesystem by default, allowing multiple sources of data. If we expand "Store" to "Stores", we can make this fully configurable. The default configuration qri is using would look like this:

```yaml
Filesystems:
- type: ipfs
  options:
    api: true
    pubsub: false
- type: local
- type: http
```

The first win: users could remove `local` or `http` to disable qri's access to the local filesystem or HTTP for the purpose of reading data. We can also expand for later filesystems & options. an `s3` implementation would be nice. this is also how we'd do `dat` integration.

We could use a convention of the default write destination being the first listed filesystem, and present a big warning if users configure anything other than `ipfs` in the first position.

It'd be great to delegate this entire section of store setup to the qfs package, aliasing up the conguration data structure into the config package.


```yaml
API:
  allowedorigins:
  - electron://local.qri.io
  - http://localhost:2505
  - http://localhost:6006
  - http://app.qri.io
  - https://app.qri.io
  enabled: true
  port: 2503
  readonly: false
  serveremotetraffic: false
CLI:
  colorizeoutput: true
Logging:
  levels:
    actions: info
    base: info
    core: debug
    cron: debug
    dsfs: debug
    lib: info
    logbook: info
    qriapi: info
    qricore: info
    qrip2p: info
P2P:
  addrs: null
  enabled: true
  peerid: QmdJQpcmGMkNtKYv5bviZ7z9kPh7uLKSMzqdSbemYs3XpH
  port: 0
  privkey: <base64-encoded private key>
  pubkey: ""
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
  port: 2504
Registry:
  location: https://registry.qri.cloud
Remote:
  acceptsizemax: 0
  accepttimeoutms: 0
  allowremoves: false
  enabled: false
  requireallblocks: false
Remotes: null
Repo:
  type: fs
Revision: 2
Filesystems:
- type: ipfs
  options:
    api: true
    pubsub: false
- type: local
- type: http
```


# Drawbacks
[drawbacks]: #drawbacks



# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives



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
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
