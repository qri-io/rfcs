- Feature Name: identity_system
- Start Date: 2017-10-01
- RFC PR: [#5](https://github.com/qri-io/rfcs/pull/5)
- Issue: NA

# Summary
[summary]: #summary

Define the baseline _self-soverign identity system_ for identifying & describing
peers on qri.

# Motivation
[motivation]: #motivation

qri is designed to be a system who's trust model is built on trust between
people and organizations. Because qri is envisioned as a _symmetrical 
information system_, having a priviledged server that provides & revokes
identities isn't an option. As such, qri peers need to be able to create their
own identities, associating

qri profiles connect descriptive information about a person or group of people
that control a keypair

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Profiles are the basis of qri's _self-soverign identity system_, as such it's
worth defining a few key terms related to self-soverign identity. Much of this
is drawn from a blog post: [A Gentle Introduction to Self-Soverign Identity](https://bitsonblocks.net/2017/05/17/a-gentle-introduction-to-self-sovereign-identity/)

### Claims, Proofs, Attestations
- **Claim**: an assertion made by the person or business. An Identity is a form 
of claim
    > “My name is Antony and my date of birth is 1 Jan 1901”
- **Proof**: is some form of document that provides evidence for the claim. 
Proofs come in all sorts of formats. eg:
    > photocopies of passports, birth certificates, and utility bills. 
    For companies it’s a bundle of incorporation and ownership structure 
    documents.
- **Attestation**: When a third party validates that according to their records, 
the claims are true. 
    > For example a University may attest to the fact that someone studied there 
    and earned a degree. An attestation from the right authority is more robust 
    than a proof, which may be forged. However, attestations are a burden on the 
    authority as the information can be sensitive. This means that the 
    information needs to be maintained so that only specific people can access 
    it.

### Issuer, Verifier
- **Issuer**: an entity that generates claims
- **Verifier**: an entity that confirms the validity of a claim

### Private & Public Signing Keys (Keypair Cryptography)
- **private key** (signing key) is used to sign documents, and is kept secret by
the issuer.
- **public key** (verification key) used to verify the signature and ensure the 
document has not been tampered with, and it does not need to be kept secret.

### Verifiable Claim
- **Verifiable claims** are a standard way of defining, exchanging, and 
verifying digital credentials. The strength of the claim depends on the degree 
of trust the verifier has in the issuer. For example, if a bank issues a claim 
saying that you have a certain credit card number, a merchant can rely on the 
claim if the merchant has a high degree of trust in the bank.

### Revocation
* "I take back what I just said"

## Qri Profiles
TODO

- Identity on qri is connected to a cryptographic keypair
- Profile connects descriptive info to a keypair
- Profile info is _opt-in_. Requred values are randomly generated
- Profile keypairs are used to sign datasets
- Profile keys are _not_ the same as IPFS keys
- Peernames are _not_ enforced as unique at this level
- Qri keypairs sign dataset body hashes as a verifiable claim


### Identity & Registries
TODO


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO - finish

Profile
```golang
// Profile is a definition of a profile on qri
type Profile struct {
  ID       string `json:"id"`
  PrivKey  string `json:"privkey,omitempty"`
  Peername string `json:"peername"`
  // Created timestamp
  Created time.Time `json:"created"`
  // Updated timestamp
  Updated time.Time `json:"updated"`
  // specifies weather this is a user or an organization
  Type string `json:"type"`
  // user's email address
  Email string `json:"email"`
  // user name field. could be first[space]last, but not strictly enforced
  Name string `json:"name"`
  // user-filled description of self
  Description string `json:"description"`
  // url this user wants the world to click
  HomeURL string `json:"homeurl"`
  // color this user likes to use as their theme color
  Color string `json:"color"`
  // Thumb path for user's thumbnail
  Thumb string `json:"thumb"`
  // Profile photo
  Photo string `json:"photo"`
  // Poster photo for users's profile page
  Poster string `json:"poster"`
  // Twitter is a peer's twitter handle
  Twitter string `json:"twitter"`
  // Online indicates if the user is currently connected to the qri network
  // Should not serialize to config.yaml
  Online bool `json:"online,omitempty"`
  // PeerIDs maps this profile to peer Identifiers in the form /[network]/peerID example:
  // /ipfs/QmSyDX5LYTiwQi861F5NAwdHrrnd1iRGsoEvCyzQMUyZ4W
  // where QmSy... is a peer identifier on the IPFS peer-to-peer network
  // Should not serialize to config.yaml
  PeerIDs []string `json:"peerIDs,omitempty"`
  // NetworkAddrs keeps a list of locations for this profile on the network as multiaddr strings
  // Should not serialize to config.yaml
  NetworkAddrs []string `json:"networkAddrs,omitempty"`
}
```


# Drawbacks
[drawbacks]: #drawbacks

By even introducing these concepts, we run the risk of alienating those who
believe decentralized networks should be privacy first.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative would be to just use keypairs, and not bother with the effort
of building up profiles at all.

# Prior art
[prior-art]: #prior-art

### Papers & Specs
* [Soverin Whitepaper](https://sovrin.org/wp-content/uploads/2018/03/Sovrin-Protocol-and-Token-White-Paper.pdf)
* [W3C Community Group Decentralized Identifier Spec Draft](https://w3c-ccg.github.io/did-spec/)
* [W3C Community Group Identity Credentials](https://opencreds.org/specs/source/identity-credentials/)
* [Keybase Overview](https://keybase.io/docs/server_security)
* [W3C Web ID](https://www.w3.org/2005/Incubator/webid/spec/identity/)
* [W3C Webn Public Key Authentication](https://www.w3.org/TR/2018/CR-webauthn-20180320/)


### Identity Orgs & Projects
There's lots of work happening in the distributed identity space:

* [uPort](https://www.uport.me/)
* [Decentralized Identity Foundation](http://identity.foundation/)
    * Github: https://github.com/decentralized-identity
* [Evernym](https://www.evernym.com/)
    * [Soverin](https://sovrin.org/)
* [keybase](https://keybase.io)
* [Solid](https://solid.mit.edu)
    * Github: https://github.com/solid/solid-spec#identity
* [Centre for Internet & Society](https://cis-india.org/)
    _nonprofit_
    > The Centre for Internet and Society (CIS) is a non-profit organisation that undertakes interdisciplinary research on internet and digital technologies from policy and academic perspectives. The areas of focus include digital accessibility for persons with disabilities, access to knowledge, intellectual property rights, openness (including open data, free and open source software, open standards, open access, open educational resources, and open video), internet governance, telecommunication reform, digital privacy, and cyber-security.
* [Namecoin](https://namecoin.org/)
* [LetsEncrypt](https://letsencrypt.org/)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Solving the problem of unique peernames (see _registries_ for that)
- Integration with other identity systems

