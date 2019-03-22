- Feature Name: merkle derived messages
- Start Date: 2019-03-18
- RFC PR: <!-- (leave this empty) -->
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Define the pattern & interfaces for "merkle derived messages": distributed execution of deterministic functions against a trusted merkle-DAG. Define prerequisites and assumptions for creating, passing, and verifying messages derived from content-addressed data.

# Motivation
[motivation]: #motivation

In Qri we have built a foundation of data that is content-addressed. We've encountered a recurrung desire to make _requests_ who's subject is content-addressed data, and have the result be trustable. In many of these scenarios, someone else has a (potentially large) DAG that we don't have, and we'd like the network to provide us with details about that DAG, without transferring all the data related to that DAG.

Concrete examples include:

* Qri DAG Info's & DAG Manifests
* dataset histories
* some dataset queries (if they existed)

All of these examples are _deterministic functions_ applied to _immutable content_. If well-formed, applying a determinsitc function to immutable content produces a derived value that can also be made immutable through content-addressing. 

These derived values will never expire, but in the context of a version control system the demand for these derived values declines as the hash content they are based on moves deeper into a version history.

Hashing this content forms a foundation for message passing.

The reason for writing this _now_ is we need to implement both DAGInfo & dataset histories as message protocols over p2p. By agreeing on the common foundations for these protocols, we can maintain two instances of a common pattern, cutting down on work.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Jo wants to get the DAGInfo of the content at `QmFoo`. Jo is connected to Lucy, Omar, Violet and Nadal.

1. Jo creates a _description_ of her request, she'd like a `daginfo` of the hash `QmFoo`.
2. Jo signs that description with her public key, and combines the description & signature into a _request message_.
3. Jo sends the _request message_ to some connected peers: Lucy, Omar, and Violet.
  * Lucy doesn't have the hash `QmFoo`, so Violet does nothing
  * Omar receives the request message, has `QmFoo`, calculates a derived value, places the result in a content addressed store (IPFS), and sends the hash of the response along with an expiry of how long Omar will hang onto the hash
  * Violet receives the request message, has `QmFoo`, but has already calculated this value, so Violet sends the hash of the response without performing a calculation
4. Jo receives Omar & Violet's responses, and they've both responded with the same hash
5. Jo fetches the hash at the given response from anyone who has it, opens it, checks it's integrity, and that the root hash matches the one she'd initially requested.

While the protocol is relatively straightforward, it has some interesting properties that need to be called out:

### We have some reason to trust the subject hash.
Why we trust the subject hash is outside of the scope of this document, but a common example would be it was resolved through Qri's naming system. It could also have been a hash we were sent by a friend we trust. It could be an IPNS record. The point is, we trust the subject hash, but don't have the DAG associated with it.

Trusting a hash in practice comes from two places: knowledge of the hash (ie: you've seen this content before) or trust in the person advertising (ie: this peerID is associated with someone I believe is trustworthy).

### The derived result will include a reference to the subject hash.
DAGInfo & DAGManifest will both include the subject hash as the first value in the list of hashes, dataset histories will do the same, also as the first entry. If we get a derived value that doesn't have the subject hash in the _subject hash position_, the result isn't trustworthy.

### Anyone with the subject DAG can derive messages.
If you have the content in question (meaning you have the DAG a hash represents stored locally), you're able to provide any derived value. This property is the same as the hashing process itself. A merkle derived message is built on a widely distributed deterministic algorithm. And just like the hashing process, we can use this feature to verify messages if we have the subject DAG.

In a peer-2-peer context, this means that asking two peers to derive the same value from the same subject DAG MUST produce the same result.

This also means we can compose _challenge requests:_ a request to derive a message to which we already know the answer. A challenge request looks exactly like a normal request. Anyone fullfilling a request for a derived value doesn't know if a request is a challenge, so long as the subject DAG is widely disseminated.

### Placing derived values in a content-addressed store creates a cache and layer of indirection.
Merkle derived messages respond to a request with the _hash of the result_, not the result itself. Because the results are deterministic, values are cached as part of the response process, cutting down on repeated work, and can even be precalculated. For Qri, derived values will often be calculated long before a peer asks form them because these same derived values are used locally. This caching property reduces the threat of bad actors forcing excessive computation, especially when combined with basic request throttling.

Respond-with-hash also means that the peer has no garuntee that _they_ will be the one asked for the result DAG, which reduces incentives to respond with bad data in the first place. It also means the group that responds to the request may not be the same group that provides the result. Some or all responding peers can be thrown directly into a bitswap session, or specifically excluded from the session for added obfuscation of the origin of the request.

### Signed responses can be re-broadcast.
If some minimum threshold of peers responded with the same value, a peer can opt to store one of the responses at random. If a new peer later asks for the same derived value, the receiver of this above-threshold number of responses can _rebroadcast_ the stored response, effectively saying "this peer says the hash-of-result is x". By retaining the response message (and signature), peers retain proper attribution of the origin of a response.

I don't propose we include rebroadcasting in the initial implementation, but save it as a later optimization if needed. A nice starting point is to have those who answer merkle-derived questions colocate with the data they are able to derive, providing higher assurances of base data availability.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We leverage this caching for an added layer of indirection. bad actors may 

If the derived value involved overly complex calculation or request load is too high, peers can simply opt to not respond. Peers who cannot resolve

The only difference between a request & a response is the request is sent out incomplete. Peers who receive a message without a

`request:`
```json
{
  "value": {
    "subject": "/ipfs/QmFoo",
    "derived": ""
  },
  "scheme": "mdm-daginfo",
  "validity": "<binary signature data>"
}
```

`response:`
```json
{
  "value": {
    "subject": "/ipfs/QmFoo",
    "derived": "/ipfs/QmResult"
  },
  "scheme": "mdm-daginfo",
  "validity" : "<binary signature data>"
}
```

### Attacks that leverage scarce content
Actors who want to be mean can create or leverage content who's DAG exists only in places the bad actor controls. The most common example of this is a "brand new hash" that the network hasn't yet replicated or verified. This case is a violation of the first principle of having an external reason to trust the base hash. One way to establish trust is to replicate the content in question & decide from there. This leads to a general rule:

merkle-derived messages should only be created as the result of actions initiated by a user that convey trust.

That sounds vague, but in practice this can be the equivalent of navigating to a dataset page in the qri app, or running `qri add` or `qri get` on a remote dataset, which explicitly conveys intent to transfer the data in question. Users who end up with data they don't expect will be able to reverse this process with corresponding commands.

### Message types MUST be versioned
For merkle-derived messages to behave properly, the _version_ of the algorithm used to deriving values must be the same. This property is shared with encryption content-addressing is based on. SHA256 works a as a point of reference because it is a known, concrete algorithm that isn't subject to change, and can be implemented in a number of programming languages. Invariably algorithms will need to change the inner workings of their implementation, denoting the version is imperitive to prevent undefined results. 

When exchanging messages peers must engage in version _negotiation_ to determine a common implementation both parties can use for the actual message derivation. Version negotiation is an uncovered section of this RFC that should be worked out in subsequent writing. In the short term implementations should be written to abort if the specified version does not match a hard-coded constant.

### Challenge requests
Requests can be framed as a _challenge_, where a peer locally constructs a request message to which they already know the answer, and disseminate 

Not all challenge requests are created equal. Highly motivated bad actors can check the pinned content of a peer, and respond with "proper" hashes

The utility of challenge requests is more academic than practical at this point. Merkle derived messages rely


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

* [minimal version selection](https://research.swtch.com/vgo-mvs)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Version Negotiation
As mentioned, version negotiation between message types needs to fleshed out.

### IPRS
The Inter Planatary Record System (IPRS) is/was a project for _generalized signed record exchange_: https://github.com/libp2p/specs/blob/master/IPRS.md. Merkle derived messages could be adapted to implement IPRS as currently specified, but it's unclear what the advantage of doing so would be.