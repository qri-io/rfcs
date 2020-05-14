- Feature Name: Externalize Private Keys
- Start Date: 2019-04-30
- RFC PR: [#52](https://github.com/qri-io/rfcs/pull/52)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Separate Qri user private key from configuration, storing in the operating system's keyring, or superseded by an environment variable `QRI_PRIVATE_KEY`.

# Motivation
[motivation]: #motivation

Qri users are getting themselves into un-recoverable states by manually deleting their `.qri` directory, which takes their private key with it, creating major problems when they try to re-add & change datasets. Because Qri is built on public-key infrastructure (PKI), deleting the private key has the effect of locking a user out of their own identity.

More importantly, storing private keys in configuration is a security vulnerability. Coupling sensitive information with configuration renders the entire configuration sensitive.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We inherited this bad habit of storing private keys in configuration as a "deal with it later" approach taken from IPFS, unlike Qri, IPFS identities are not "significant" in the sense that it's not tied to an identity intended for humans. Tying an identity to a private key means we need to deal with this problem much earlier.

This RFC deprecates `profile.privKey` config value, moving the the private key to the user's operating system keyring. When no keyring is available, store the keyring in a separate file called `QRI_PRIVATE_KEY` within the `.qri` directory, and present the user with a warning that they should copy this file to a safe location.

In all cases the active private key can be overridden with an environment variable: `QRI_PRIVATE_KEY`. In all cases the key is expected to be a base64-encoded string.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Doing the work of generalizing secret storage can be handled by an external package: https://github.com/zalando/go-keyring. go-keyring supports Windos, OS X, and Linux. The key is stored in a `(username,password)` combination, with the "username" being the user's ProfileID (hash of public key), and the password being the base64-encoded private key. OS Keyrings support namespacing, qri's namespace will be a reversed-domain-name constant:

```go
const qriKeyringServiceID = "io.qri.keys"
```

Once setup is complete, `config.Profile.ID` stores the ProfileID string this configuration is attached to. Starting a qri process uses this as the "username" in a `Get` call:

```go
privKey, _ := keyring.Get(qriKeyringServiceID, config.Profile.ID)
```

### Keys subsystem

All of this work should happen in a new package within the qri codebase: `github.com/qri-io/qri/keys`. We need to takle key rotation next, followed by storing _many_ keys for doing access control. All of this work should happen in a new subsystem called `keys.Service`. `lib.Instance` gets a new field that points to this service.

### Phasing out `config.Profile.PrivKey`
This work should happen in phases:

1. **copy to keyring, support env var**. In in the next release, add in the new APIs, prefer their use when present.
2. **intiailize with an empty privKey value**. Once we know the keyring approach works across a number of different platforms in a wide release, switch default initialization behaviour to _not_ write the privKey value.
3. **remove the field** Eventually, remove the field entirely with a repo migration.

In all phases the privKey field should be the _lowest_ in the loading hierarchy. Keyring values will override `Profile.PrivKey`, and the env var will override all other values.

### Recovering from a deleted `.qri` folder
Once we've externalized the private keys, we can change the behaviour of `qri setup` to first scan the keyring under the `io.qri.keys` namespace for existing keys. If a key is found, we can ask the user if they'd like to create a new account, or recover an old one.

We can bolster this with calls to the default registry for the discovered public keys to add username info to the list. Profile information may not exist on the default registry, so we'll need to default to showing just the raw ProfileID. We can support this process with docs.

Instead of jumping straight to building the UI for switching, we can supply a `--recover` flag that lets a user supply a ProfileID to recover from:

```
$ qri setup --recover QmProfileID
```

listing available private keys in a keyring may also form the basis of profile switching in the future.

# Drawbacks
[drawbacks]: #drawbacks

### More OS-Specific Work.
This increases our reliance on an OS-level API that sometimes doesn't exist, and runs the risk of deviating behaviour between platforms. 

Fallback to a `QRI_PRIVATE_KEY` file whenever no keyring exists is necessary to keep things working when our keyring implementation fails, effectively reverting us back to the setup we're in today. This time we at least have a separate file and a warning.

### Upstream work to get recovery functionality
The package I've suggested we build on has no high-level method for listing the keys in a namespace. We'e need to be able to do this, and haven't confirmed this will be possible across all platforms.

### No Qri-specific security
This RFC does not propose adding any key-specific security. If we used symmetric encryption on the private key itself, Qri could require a local password of it's own. In this case we're relying on the user's Operating System security architecture

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

A number of alternatives exist for externalizing secrets. We could store keys in the `.ssh` directory if it exists, and looking for guidance on 

# Prior art
[prior-art]: #prior-art

* Keybase - We should really do a dive on the approach keybase uses for key management before moving on to any other keypair work. _Especially_ key rotation.
* Textile wallet
* Hashicorp Vault

* https://github.com/keybase/go-keychain
* https://github.com/zalando/go-keyring

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Loading order for configuration
We haven't defined _when_ key loading happens in this process now that key loading is separate from configuration loading in the relation to seting up other services.

### Profile Switching
This RFC lays the groundwork for switching accounts _within_ qri, but this would have very strange results, as the data stored in `.qri` is mapped to a single user. We also don't have any UI for how this would work. 

### Key Rotation
We still haven't covered how to rotate this master private key in the event of a private key disclosure. That'll come later.

### Multiple Keys 
The "master key" is not the only key we need to store. In the future we'll end up needing to create all sorts of keys for managing access control. I'm wondering if we use something like [BIP0032](https://en.bitcoin.it/wiki/BIP_0032) for this.