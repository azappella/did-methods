
# Sidetree Asset Protocol

Using blockchains for anchoring and tracking unique digital assets (IDs, tokens, etc.) is a useful primitive, but the current strategies for doing so suffer from severely limited transactional performance constraints. Sidetree Assets is a Layer 2 protocol for anchoring and tracking assets across blockchains. Sidetree Assets support a variety of DPKI operations. Sidetree Assets, and nodes that implement the protocol, are not another blockchain, and do not require a consensus layer, they run atop any existing blockchain that supports the ability to embed a small amount of data in a transaction.

## System Overview

To generate the global state of Sidetree Assets in accordance with the protocol, a compute node observes implementation-included blockchains to watching for transactions embedded with Sidetree Asset payloads, then applies protocol rules to process them. A valid Sidetree Asset transaction is marked in a recognizable way, and includes the root hash of a Sidetree Asset structure (a Merkle Tree of hashes that correspond to Sidetree Asset object operations).

Both the Sidetree's full Merkle Tree and source data for its leaves are stored in an open, decentralized, content addressable, highly-replicated storage system that anyone can run to provide redundancy of Sidetree Asset source data.

The protocol relies on the ordered nature of blockchain transactions, signed pointers to previous operations, and embedded Merkle proofs to derive a deterministic lineage of Asset operations as they progress through their lifecycle - this is called a Sidetree Asset Trail.

## Sidetree Asset Operations

Sidetree Assets are a form of [Decentralized Identifier](https://w3c-ccg.github.io/did-spec/) (DID) that manifest through blockchain-anchoring DID Document objects (and subsequent deltas) that represent the existence and state of assets. Each update to an asset includes its full operational lineage (creation, update, recovery, transfer, etc.) across a blockchain's linear history. Ownership of an Asset is linked to possession of keys specified within the Asset object itself.

The following is a simplified pseudo code example of a Sidetree Asset's JSON text, to help explain the content and functional elements:

```javascript
[
  {
    sig: OWNER_SIG_DELTA_2, // Latest update of DID
    proof: PROOF_OF_DELTA_1,
    delta: { RFC_6902_JSON_PATCH },
    recovery: RECOVERY_VALUE
  },
  {
    sig: OWNER_SIG_DELTA_1, // Previous update of DID
    proof: PROOF_OF_DELTA_0,
    delta: { RFC_6902_JSON_PATCH },
    recovery: RECOVERY_VALUE
  },
  {
    sig: OWNER_SIG_DELTA_0, // Initial creation of DID
    delta: { RFC_6902_JSON_PATCH },
    recovery: RECOVERY_VALUE
  }
]
```

System diagram showing op sig links that form a Sidetree Asset Trail:

> NOTE: This graphic is out-of-date, and does not match the description of the explainer. TODO: Update graphic.

![Sidetree Asset Trail diagram](https://i.imgur.com/RbwM1nj.png)

### Creation of a Sidetree Asset

Creation of an Asset is accomplished via the following set of procedures:

1. Generate a JSON-encoded array that includes one member: an object with a `delta` property that describes the initial state of a [DID Document](https://w3c-ccg.github.io/did-spec/#self-managed-did-document), in the delta format specified by [RFC 6902](http://tools.ietf.org/html/rfc6902).
2. Add a `sig` property to the object that is the signature of the initial delta, generated with the owning keys specified in the delta.
3. Hash the Sidetree Asset object and embed it in a Merkle Tree with other Sidetree Asset operations.
4. Create a transaction on the blockchain with the Merkle root embedded within it and mark the transaction to indicate it contains a Sidetree.
5. Store the Merkle tree source and all Asset objects in the decentralized storage system.
6. OPTIONAL: You may choose to include a `recovery` property in the DID Document, the value of which is a hash of a recovery secret.

```javascript
[
  {
    sig: OWNER_SIG_DELTA_0, // Initial creation of DID
    delta: { RFC_6902_JSON_PATCH },
    recovery: RECOVERY_PROOF
  }
]
```
> NOTE: Because every operation beyond initial creation contains pointers and delta state proofs that link previous operations together, only the latest revision of the object must be retained in the decentralized storage layer.

> NOTE: Proof of Asset ownership plus a subset of uses should be possible (because regardless of block inclusion, the owner is verifiable via included key references), but the Asset remains in a pending state until it is confirmed in a block.

### Asset ID

Each Sidetree Asset has an emergent ID, which is composed of the the following combination of values: `MERKLE_ROOT` + `Unicode EN DASH (U+2013)` + `SIDETREE_ASSET_HASH`
  - Example: `did:xcid:1xms56ng0646faf3f43fa33f4faw4ds3-3bv3232k23937m7ds3133f4faw4f43f`


### Updating a Sidetree Asset

To update an Asset, use the following procedures:

1. Create a new Asset operation object to append to the Asset's JSON array
2. Add a `delta` property to the operation object with the value being an object that includes any changes to the DID Document, encoded in the format specified by [RFC 6902](http://tools.ietf.org/html/rfc6902)
3. Add a `sig` property to the operation object that is the signature of the initial delta, generated with the owning keys specified in the delta.
4. Add a `proof` property to the operation object with the value being the Merkle Proof that validates the last operation and its inclusion in the blockchain.

## Sidetree Asset Processing Rules

Assets are processed by protocol-enforcing compute nodes that observes the chain and apply the protocol rules to transactions marked as bearing Sidetree Assets. When a transaction is found, the node follows this procedural rule set

#### Walking the Chain

1. Secure a copy of the target blockchain and listen for incoming transactions
2. Starting at the genesis entry of the blockchain, begin processing the included transactions in order, from earliest to latest.

#### Inspecting a Transaction

3. For each transaction, inspect the property known to bear the marking of a Sidetree Asset. If the transaction is marked as a Sidetree Asset, continue, if unmarked, move to the next transaction.
4. Locate the Sidetree Asset's Merkle Root within the transaction.

#### Processing the Sidetree

5. Fetch the Sidetree's Merkle Tree from the decentralized storage system
6. Fetch each of the Merkle Tree's leaf objects from the decentralized storage system. If a leaf object is not found in the decentralized storage system, skip and proceed with the next leaf.

#### Evaluating a Leaf

7. Process each leaf via the following steps:

  - If `recovery` is present ... TBD

  - If `proof` is not included:
      1. The object shall be treated as a new Asset registration
      2. Ensure that the delta is signed with the owner's specified key material. If valid, proceed, if invalid, discard the leaf and proceed to the next.
      3. Within the protocol-enforcing node's cache, create an entry for the new Asset, grouped by Merkle Root, and indexed by the Sidetree Asset object's hash.
      4. Pin the asset for retention in the decentralized storage medium.

  - If `proof` is present:
      1. Retrieve the last valid Asset object state from cache.
      2. Ensure that the delta is signed with the owner's known key material. If valid, proceed, if invalid, discard the leaf and proceed to the next.
      3. Use the proof in the new operation to validate that the last operation was included in the Sidetree Merkle Root it indicates.
      4. Verify that the new Asset version is not a deviation from the chronological lineage of operations described in the currently cached version of the Asset object. If there is no deviation in operation history, and the latest operation is found to be valid, proceed. If the new Asset object differs in its operation history and any of the operations are invalid, discard it. If it differs in operation history, but is valid, and its operations are chronologically earlier than the current Asset object's, proceed.
      4. Replace the existing Asset's cached object with the new operation.
      5. Pin the asset for retention in the decentralized storage medium.
      6. Unpin the previous version of the Asset from the decentralized storage medium.




