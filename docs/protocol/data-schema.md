# Data schema

`The current contents are a work in progress`

## Origins

The metadata of entities and voting process may refer to data origins. Data can come from various origins, including decentralized filesystems as well as traditional HTTPS endpoints. Messaging can also come from PSS or IPFS PubSub nodes.

The list of potential origins can be as follows:

### Messaging URI

Referd as `<messaging uri>` from now on.

- `pss://<publicKey@address>`
    - Uses Ethereum Swarm/PSS protocol
    - address can be empty
- `pubsub://<topic>`
    - Uses IPFS pubsub protocol
- `shh://<publicKey>`
    - Uses Ethereum Whisper protocol

### Content URI

Referd as `<content uri>` from now on.

- `bzz://<contentHash>`
- `bzz-feed://<feedHash>`
- `ipfs://<contentHash>`
- `http://<url>/<route>`
- `https://<url>/<route>`

## Entity metadata

The JSON payload below is to be stored on Swarm or IPFS, so anyone can fetch the metadata of an entity through a decentralized channel.

```json
{
    "version": "1.0",    // Protocol version
    "address": "0x1234...",
    "name": "The Entity",
    "home": "https://www.the-entity.org/",
    "actions": [{  // Optional array of custom actions
        "name": {
            "default": "Sign up to The Entity",  // used if none of the languages matches
            "fr": "S'inscrise à l'organisation"
        },
        "type": "browser",
        "url": "https://census-manager.domain/sign-up/",
        "parameters": {
            // Tell our custom Process Manager what census service
            // to use when registering users
            "censusOrigin": "<messaging uri>",
            "censusId": "the-entity-main-census"
        },
        // Is a signed timestamp+body required?
        "needs-signature": true,
        // Endpoint to POST with publicKey and signature fields
        // Returning true will show the action and hide it otherwise
        "visible": "https://census-manager.domain/status/visible/"
        // "visible": true
    }],
    "content": {
        "news": {
            "name": {
                "default": "Official news",
                "fr": "Messages officiels"
            },
            "origin": "bzz-feed://<feedHash>" // Points to an origin resolving to Content Data
        }
    }
}
```

**Used in:**
* [Entity creation](/protocol/sequence-diagrams?id=entity-creation)
* [Entity subscription](/protocol/sequence-diagrams?id=entity-subscription)

**Related:**
* [Entity Smart Contract](https://github.com/vocdoni/dvote-smart-contracts/blob/master/contracts/VotingEntity.sol)
* [Entity JS methods](https://github.com/vocdoni/dvote-client/blob/master/src/dvote/entity.ts)
* [Content Data](/protocol/data-schema?id=content-data)

## Process metadata

The JSON payload below is to be stored on Swarm or IPFS, so anyone can fetch the metadata of a voting process through a decentralized channel.

```json
{
    "version": "1.0",    // Protocol version
    "name": "Basic income rule",
    "address": "0x1234...", // on the blockchain
    "question": "Should basic income be a human right?",
    "voteOptions": [
        { "name": "Yes", "value": 1 },
        { "name": "No", "value": 2 },
        { "name": "I don't know", "value": 3 }
    ],
    "type": "zk-snarks",  // Allowed ["zk-snarks", "lrs"]
    "startBlock": 10000,
    "endBlock":  11000,
    "meta": {
        "description": "## Markdown text goes here\n### Abstract",
        "images": [ "<content uri>", ... ],
        "organizer": {
            "address": "0x1234...",  // Address of the Entity entry on the blockchain
            "metadata": "<content uri>" // Organizer's metadata on Swarm
        }
    },
    "census": {
        "id": "the-entity-main-census",  // Census ID to use
        "origin": "<messaging uri>", // Census service to request data from
        "merkleRoot": "0x1234...",
        "modulusSize": 5000  // Only when type="lrs"
    },
    "publicKey": "0x1234..." // To encrypt vote packages
}
```

**Used in:**
- [Voting process creation](/protocol/sequence-diagrams?id=voting-process-creation)
- [Voting process retrieval](/protocol/sequence-diagrams?id=voting-process-retrieval)
- [Casting a vote with ZK Snarks](/protocol/sequence-diagrams?id=casting-a-vote-with-zk-snarks)
- [Casting a vote with Linkable Ring Signatures](/protocol/sequence-diagrams?id=casting-a-vote-with-linkable-ring-signatures)
- [Vote Scrutiny](/protocol/sequence-diagrams?id=vote-scrutiny)

**Related:**
* [Process Smart Contract](https://github.com/vocdoni/dvote-smart-contracts/blob/master/contracts/VotingProcess.sol)
* [Process JS methods](https://github.com/vocdoni/dvote-client/blob/master/src/dvote/process.ts)

**Notes:**
- The `type` field indicates the scrutiny method that will be used for the process. Any vote package generated with the wrong type will be discarded. 
- The list of authorized relays is available on the Process smart contract

## Vote Package

### Vote Package - ZK Snarks

```json
{
    "version": "1.0",    // Protocol version
    "type": "zk-snarks",
    "processAddress": "0x1234...",
    "encryptedVote": "0x1234...",
    "nullifyer": "0x1234...",
    "proof": "01234...",
    "censusMerkleRoot": "0x1234..."
}
```

**Used in:**
- [Casting a vote with ZK Snarks](/protocol/sequence-diagrams?id=casting-a-vote-with-zk-snarks)
- [Vote Scrutiny](/protocol/sequence-diagrams?id=vote-scrutiny)

### Vote Package - Ring Signature

```json
{
    "version": "1.0",    // Protocol version
    "type": "lrs",
    "processAddress": "0x1234...",
    "encryptedVote": "0x1234...",
    "signature": "0x1234...", // The ring signature over the processAdress
    "publicKeyModulus": 4321,
    "censusMerkleRoot": "0x1234..."
}
```

**Used in:**
- [Casting a vote with Linkable Ring Signatures](/protocol/sequence-diagrams?id=casting-a-vote-with-linkable-ring-signatures)
- [Vote Scrutiny](/protocol/sequence-diagrams?id=vote-scrutiny)

## Vote Batch

```json
{
    "version": "1.0",    // Protocol version
    "type": "lrs", // or zk-snarks
    "relay": {
        "publicKey": "0x1234..."
    },
    "votes": [ // Vote Package, see above
        { ... },
        { ... },
        { ... }
    ]
}
```

**Used in:**
- [Registering a Vote Batch](/protocol/sequence-diagrams?id=registering-a-vote-batch)
- [Checking a submitted vote](/protocol/sequence-diagrams?id=checking-a-submitted-vote)
- [Vote Scrutiny](/protocol/sequence-diagrams?id=vote-scrutiny)

## Vote Summary

```json
{
    "version": "1.0",    // Protocol version
    "process": {
        "name": "Basic income rule",
        "address": "0x1234...", // on the blockchain
        "question": "Should basic income be a human right?",
        "voteOptions": [
            { "name": "Yes", "value": 1 },
            { "name": "No", "value": 2 },
            { "name": "I don't know", "value": 3 }
        ],
    },
    "organizer": {
        "address": "0x1234...",  // Address of the Entity entry on the blockchain
        "metadata": "<content uri>" // Organizer's metadata on Swarm
    },
    "results": {
        "1": 12345678,
        "2": 23456789,
        "3": 34567890
    }
}
```

**Used in:**
- [Vote Scrutiny](/protocol/sequence-diagrams?id=vote-scrutiny)

## Vote List

```json
{
    "version": "1.0",    // Protocol version
    "process": {
        "name": "Basic income rule",
        "address": "0x1234...", // on the blockchain
        "question": "Should basic income be a human right?",
        "voteOptions": [
            { "name": "Yes", "value": 1 },
            { "name": "No", "value": 2 },
            { "name": "I don't know", "value": 3 }
        ],
    },
    "organizer": {
        "address": "0x1234...",  // Address of the Entity entry on the blockchain
        "metadata": "<content uri>" // Organizer's metadata on Swarm
    },
    "votes": {
        "valid": [{
            "nullifyer": "0x1234...",
            "vote": 1,
            "batchId": "0x1234..."
        }],
        "invalid": [{
            "reason": "invalid-relay",
            "package": { ... }  // original vote package
        }, {
            "reason": "duplicate-vote",
            "batchId": "0x1234...",
            "package": { ... }  // original vote package
        }, {
            "reason": "invalid-proof",
            "batchId": "0x1234...",
            "package": { ... }  // original vote package
        }, {
            "reason": "invalid-signature",
            "batchId": "0x1234...",
            "package": { ... }  // original vote package
        }, {
            "reason": "invalid-vote-value",
            "batchId": "0x1234...",
            "package": { ... }  // original vote package
        }]
    }
}
```

**Used in:**
- [Vote Scrutiny](/protocol/sequence-diagrams?id=vote-scrutiny)

**Notes:**
- The current payload may lead to a size of several Gigabytes of data, which may not be suitable for mobile devices

## Content Data

Used to contain news and data posts.

```json
[{
    "title": "New voting process available",
    "description": "Basic income vote available on April 1st",
    "date": "2019-01-01T10:10:00.000Z",
    "content": "<h2>Basic income vote</h2><p>HTML content goes here</p>"
}]
```

## Census Service

### Census Service request payload

Requests sent to the census service may invoke different operations.

Depending on the `method`, certain parameters are expected or optional:

#### Census addClaim
```json
{
    "method": "addClaim",
    "censusId": "string",
    "claimData": "string",
    "signature": "string"
}
```

**Used in:**
* [Adding users to a census](/protocol/sequence-diagrams?id=adding-users-to-a-census)

#### Census getRoot
```json
{
    "method": "getRoot",
    "censusId": "string" 
}
```

**Used in:**
- [Voting process creation](/protocol/sequence-diagrams?id=voting-process-creation)

#### Census genProof
```json
{
    "method": "genProof",
    "censusId": "string",
    "claimData": "string",
    "rootHash": "optional-string"  // from a specific version
}
```

**Used in:**
- [Check census inclusion](/protocol/sequence-diagrams?id=check-census-inclusion)
- [Casting a vote with ZK Snarks](/protocol/sequence-diagrams?id=casting-a-vote-with-zk-snarks)

#### Census getChunk
```json
{
    "method": "getChunk",
    "censusId": "string",
    "rootHash": "optional-string",  // from a specific version
    "publicKeyModulus": 4321
}
```

**Used in:**
- [Casting a vote with Linkable Ring Signatures](/protocol/sequence-diagrams?id=casting-a-vote-with-linkable-ring-signatures)

#### Census checkProof
```json
{
    "method": "checkProof",
    "censusId": "string",
    "claimData": "string",
    "rootHash": "optional-string",  // from a specific version
    "proofData": "string"
}
```

#### Census getIdx
```json
{
    "method": "getIdx",
    "censusId": "string",
    "claimData": "string",
    "rootHash": "optional-string"
}
```

#### Census dump
```json
{
    "method": "dump",
    "censusId": "string",
    "rootHash": "optional-string",
    "signature": "string"
}
```

**Used in:**
- [Voting process creation](/protocol/sequence-diagrams?id=voting-process-creation)

Requests may be sent over HTTP/HTTPS, as well as PSS or IPFS pub/sub.

**Related:**

* [Census service API specs](https://github.com/vocdoni/go-dvote/tree/master/cmd/censushttp#api)

### Census Service response payload

```json
{
    "error": false,
    "response": "string"
}
```

## Gateway request payload

`Work in progress`

## Relay request payload

`Work in progress`

**Notes:**
- See [Vote Package](/protocol/data-schema?id=vote-package) above