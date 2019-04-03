# Entity

An entity can have many roles. For the most part, it is the organizer and the ultimate responsible of a voting process. However, an Entity could make no use of the voting system and instead be used as a trusted actor by

- Publishing news/information via the [Feed](#feed)
- Providing pointers to trusted/related [Entities](#entities-list)
- Providing [Gateway boot nodes](#gateway-boot-nodes) or Relays

## Index

- [Entity Resolver](#entity-resolver)
  - [Storage of Text records](#storage-of-text-records)
  <!-- - [Storage of lists of text](#storage-of-lists-of-text) -->
  - [Naming convention for Resolver keys](#naming-convention-for-resolver-keys)
- [Entity](#entity)
  - [Index](#index)
  - [//The contract, but it is tightly coupled with the **JSON Entity Metadata** living on P2P filesystems.](#the-contract-but-it-is-tightly-coupled-with-the-json-entity-metadata-living-on-p2p-filesystems)
  - [Entity Resolver](#entity-resolver)
    - [Storage of Text records](#storage-of-text-records)
    - [Storage of lists of text records](#storage-of-lists-of-text-records)
    - [Record guidelines](#record-guidelines)
    - [Resolver keys](#resolver-keys)
  - [| `vnd.vocdoni.relays.active` | '&lt;relay&gt;' | Relays public keys |](#vndvocdonirelaysactive--ltrelaygt--relays-public-keys)
  - [Data schema](#data-schema)
    - [Meta](#meta)
    - [Gateway boot node](#gateway-boot-node)
      - [Relay](#relay)
      - [GatewayUpdate](#gatewayupdate)
    - [EntitiyReference](#entitiyreference)
    - [Feed](#feed)
    - [Entity Actions](#entity-actions)
  - [ENS](#ens)
    - [Overview](#overview)
    - [No Mainnet](#no-mainnet)
    - [ENS domain authentication](#ens-domain-authentication)
    - [Comparison](#comparison)

---

For the most part, the entity metadata lives in the blockchain. Alternatively, it is indexed in the blockchain and retrieved using P2P communication.

- Data on the Blockchain provides durability and enables integrity checking
- Data on content-addressed filesystems allows to transfer larger data objects at the expense of a higher risk of data aviability problems.

We refer to this data aggregate as the Entity Metadata.

//The  contract, but it is tightly coupled with the **[JSON Entity Metadata](#data-schema)** living on P2P filesystems.
---

## Entity Resolver

An **[Entity Resolver](#entity-resolver)** is the smart-contract where the Entity metadat is stored/indexd. It follows the same architecture as a ENS resolver contract, but currently does not make use of ENS domains.

The `entityId` is the unique identifier of each entity, being a hash of its creator's address:

```solidity
bytes32 entityId = keccak256 (entityAddress);
```

An Entity Resolver implements several interfaces

- Storage of Text records
- Storage of list text records
- Interface resolver

### Storage of Text records

We make use of [EIP 634: Storage of Text records in ENS](https://eips.ethereum.org/EIPS/eip-634). It is a convenient way to store arbitrary data as a string following a key-value store model.

- [Implementation](https://github.com/vocdoni/dvote-solidity/blob/master/contracts/profiles/TextResolver.sol)
  
### Storage of lists of text records

This is necessary in order to minimize the amount of data to write when the metadata can be split.

The user is responsible for managing the indexes, the array does not move its elements.

The behaviour wants to mimic `The storage of text records`

- [Implementation](https://github.com/vocdoni/dvote-solidity/blob/master/contracts/profiles/TextListResolver.sol)

### Record guidelines

Content URIs are formated using the [Content URI specification](/architecture/protocol/data-origins?id=content-uri).

Any other record stored under Vocdoni's key convention is formatted as stringified JSON object

- **Objects**  `JSON.parse('{"keyName":"valueGoesHere"}')` => `{ keyName: "valueGoesHere" }`
- **Arrays**  `JSON.parse('["0x1234","0x2345","0x3456"]')` => `[ "0x1234", "0x2345", "0x3456" ]`
- **Strings**  `JSON.parse('"String goes here"')` => `"String goes here"`
- **Numbers**  `JSON.parse('8')` => `8` 
- **Booleans**  `JSON.parse('true')` => `true`

This applies to [Storage of Text records](#storage-of-text-records) as well as  [Storage of lists of text records](#storage-of-lists-of-text-records).

>**Important:** It is the Entity's responsibility to ensure that the stored data properly parses into a valid JSON object, once retrieved from the blockchain.

### Resolver keys

[EIP 634](https://eips.ethereum.org/EIPS/eip-634) convention:
> Keys must be made up of lowercase letters, numbers and the hyphen (-). Vendor specific services must be prefixed with vnd.

This is the suggested convention:

`<prefix>.<type>.<purpose>.<attribute>`

- `prefix`: **mandatory**. To not kidnap ENS keys. In our case this is always `vnd.vocdoni`
- `purpose`:  **mandatory**. what is this key used for.
- `attribute`: **optional**. What is special about key compared to other keys with the same `purpose` (active/ended, trusted, locale)

Below is a table with the proposed standard for key/value denomination.

**Text record keys**

| Key                                 | Example                                                       | Description                                                                                                                                         |
|-------------------------------------|---------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `vnd.vocdoni.entity-name`           | 'Free Republic of Liberland'                                  | Entity's name                                                                                                                                       |
| `vnd.vocdoni.meta`                  | 'bzz://12345,ipfs://12345'                                    | [Content URI](/architecture/protocol/data-origins?id=content-uri) to fetch the JSON metadata. <br/>See [Entity Metadata](#entity-metadata-1) below. |
| `vnd.vocdoni.voting-contract`       | '0xccc'                                                       | Address of the Processes Smart Contract instance used by the entity                                                                                 |
| `vnd.vocdoni.gateway-update`        | '&lt;gatewayUpdate&gt;'                                       | Parameters for Gateways to report availability to boot nodes. [See GatewayUpdate](#gatewayupdate)                                                   |
| `vnd.vocdoni.process-ids.active`    | '["0x987","0x876"]'                                           | List of `processId`'s displayed as available by the client                                                                                          |
| `vnd.vocdoni.process-ids.ended`     | '["0x887","0x886"]'                                           | List of `processId`'s displayed as unavailable by the client                                                                                        |
| `vnd.vocdoni.news-feed.en`          | 'bzz-feed://23457,ipfs://23457,https://liberland.org/feed'    | [Content URI](/architecture/protocol/data-origins?id=content-uri) of the feed in an specific language.                                              |
| `vnd.vocdoni.news-feed.fr`          | 'bzz-feed://23456,ipfs://23456,https://liberland.org/feed/fr' | [Content URI](/architecture/protocol/data-origins?id=content-uri) of the feed in an specific language.                                              |
| `vnd.vocdoni.entity-description.en` | 'Is a sovereign state...'                                     | Entity description in the default language                                                                                                          |
| `vnd.vocdoni.entity-description.fr` | 'Dans un état souverain...'                                   | Entity description in the specific locale )                                                                                                         |
| `vnd.vocdoni.avatar`                | 'https://liberland.org/logo.png'                              | [Content URI](/architecture/protocol/data-origins?id=content-uri) of an image file to display next to the entity name                               |

**List of Text record keys**

| Key                                              | Record example            | Description                                                                                                                                                                         |
|--------------------------------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `vnd.vocdoni.gateway-boot-nodes`                 | '&lt;gatewayBootNode&gt;' | Data of the boot nodes to ask for active gateways. [See below](#gateway-boot-nodes) for more details                                                                                |
| `vnd.vocdoni.boot-entities`                      | '&lt;entityReference&gt;' | A starting point of entities list to allow users to browser from a curated list. Only used in "boot" entities like the case of Vocdoni itself.  See [Entities List](#entities-list) |
| `vnd.vocdoni.fallback-bootnodes-entities`        | '&lt;entityReference&gt;' | A [list of entities](#entities-list) to borrow the bootnodes from in case of failure.                                                                                               |
| `vnd.vocdoni.trusted-entities`                   | '&lt;entityReference&gt;' | A list of entities that the own entity trusts.  See [Entities List](#entities-list)                                                                                                 |
| `vnd.vocdoni.census-service-authorized-entities` | '&lt;entityReference&gt;' | Census-service uses it to authorize entities to write on it. Only entities with a census-service use it                                                                             |
| `vnd.vocdoni.census-ids.active`                  | '0xccc'                   | CensusIds that the census-service keeps alive                                                                                                                                       |
| `vnd.vocdoni.relays.active`                      | '&lt;relay&gt;'           | Relays public keys                                                                                                                                                                  |
---

## Data schema

### Meta

**Description**

It replicates the entire data of the Entity metadata stored in the resolved in a single JSON object.

Its purpose is to minimize the retrival of metadata requests.

This creates two sources of truth. The metadata in the contract itself and the one in this object.

>In the event of a mismatch, the metadata on the blockchain is used.

**Schema**

Meta
```json
{
  //text records
  "version": "1.0",    // Protocol version
  "entity-name": "Free Republic of Liberland",
  "entity-description": {
    "ca": "In a sovereign state...",
    "fr": "Dans un état souverain"
  },
  "voting-contract": "0xccc",
  "gateway-update":{
    "timeout": 60000,                   // milliseconds after which a Gateway is marked as down
    "topic": "vocdoni-gateway-update",  // Topic used for the messaging protocol
    "difficulty": 1000                  // Difficulty of the proof of work, to prevent spammers
  },
  "process-ids":{
    "active":["0x987","0x876"],
    "ended":["0x887","0x886"]
  },
  "news-feed":{
    "en": "bzz-feed://34567,ipfs://34567,https://liberland.org/feed/",
    "fr": "bzz-feed://23456,ipfs://23456,https://liberland.org/feed/fr"
  },
  "avatar": "https://liberland.org/logo.png,bzz://12345,ipfs://12345",
  ...
  // list of text records
  "gateway-boot-nodes": [  // Bootnodes providing a list of active Gateways
    {
      "update": "pss://publicKey@0x0",
      "fetch": "https://hostname:port/route"
    },
  ],
  // List of currently active Relays. This list is just for informational purposes.
  // The effective list of relays for a voting process is within the Process Metadata
  // and the Voting Process smart contract
  "relays": [
      {
          "address": "0x1234...",     // PSS adress to help routing messages
          "publicKey": "0x23456...",  // Key to encrypt data sent to it
          "uri": "<messaging-uri>"    // Where to send messages. See Data origins > Messaging URI
      }, 
      ...
  ],
  "actions": [ <actions> ], // See Entity Actions below
  
  ...
}
```

**Sequence diagrams:**

- [Set Entity metadata](/architecture/sequence-diagrams?id=set-entity-metadata)
- [Entity subscription](/architecture/sequence-diagrams?id=entity-subscription)

**Related:**

- [Entity Smart Contract](https://github.com/vocdoni/dvote-solidity/blob/master/contracts/VotingEntity.sol)
- [Entity JS methods](https://github.com/vocdoni/dvote-js/blob/master/src/dvote/entity.ts)

### Gateway boot node

**Description**

Client apps will normally be unable to join P2P networks by themselves, so Vocdoni makes use of Gateways to enable decentralized transactions over http/https.

A gateway-boot-node is a server trusted by the Entity whoes goal is to provide a list of active gateway IP addresses via https

Considerations:

- Initial bootnodes are hardcoded into the client App to prevent the chicken and the egg problem of an App unable to find a an active gateway in the blockchain because it does not have a gateway in the first place. 
- A gateway-boot-node is a best effort starting point
- To minimize censorship attacks organizations should should provide their own set of Gateways.

**Usages**

`vocdoni.gateway-boot-nodes` provides a list of currently active boot-nodes.

**Schema**

GatewayBootNode
```json
  {
    "update": "pss://publicKey@0x0",        // Messaging URI to use for notifying updates to the bootnode
    "fetch": "https://hostname:port/route"  // URL to use for fetching the list of Gateways
  }
```

#### Relay

```json
{
  "address": "0x1234...",     // PSS adress to help routing messages
  "publicKey": "0x23456...",  // Key to encrypt data sent to it
  "uri": "<messaging-uri>"    // Where to send messages. See Data origins > Messaging URI
}
```

#### GatewayUpdate

Bootnode servers provide the list of available gateways at the time of requesting. In order to keep an accurate state, Gateways need to notify events to the Bootnodes.

To this end, the `vocdoni.gateway-update` text field provides the details that Gateways need to use:

```json
{
  "timeout": 60000,                   // milliseconds after which a Gateway is marked as down
  "topic": "vocdoni-gateway-update",  // Topic used for the messaging protocol
  "difficulty": 1000                  // Difficulty of the proof of work, to prevent spammers
}
```

This value is global and effects all the Gateways of the Entity.

### EntitiyReference

It represents a specific entity
Lists of `EntityReference`s have several purposes.

- `vocdoni.entities.boot` : Entry point for the user to subscribe to new entities.
- `vocdoni.entities.trusted`: Reputation/whitelisting mechanism for entities to express a relationship with other entities

Each element on the list contains an `Entity Reference` entry
- `resolverInstance`: The contract instance address, where the Entity's metadata lives
- `entityAddress`: The address of an entity


```json
{
  "resolverAddress": "0xaaa", //contract address of the smart-contract
  "entityId": "0xeee"         //entityId. Hash of the the creator address
}
```

### Feed

The `Feed` serves the purpose of having a uni-directional censorship-resistant communication channel between the `Entity` and the `user`.

Content feeds are expected to conform to the specs of a [JSON Feed](https://jsonfeed.org/) and be accessible through a [Content URI](/architecture/protocol/data-origins?id=content-uri). 

See:
- [JSON feed specification](https://jsonfeed.org/version/1)

### Entity Actions

Entity Actions are custom operations that clients will be offered to perform. Their definition is stored within the [JSON metadata](#json-metadata).

Below is a reference of suported use cases:

```json
{
    ...
    "actions": [{

        // Opening an interactive web browser
        "type": "browser",

        // Localized Call To Action to appear on the app
        "name": {
            "default": "Sign up to The Entity",  // used if none of the languages matches
            "fr": "S'inscrire à l'organisation"
        },

        // The URL to open with query string parameters:
        // - signature = sign(hash(timestamp), privateKey)
        // - publicKey
        // - timestamp (UNIX timestamp)
        "url": "https://census-register.cloud/sign-up/",

        // Endpoint to POST to with publicKey and signature+timestamp JSON fields
        // Returning true will show the action and hide it otherwise
        "visible": "https://census-registry.cloud/lambda/census-register-visible/"
        // "visible": true    (always visible, alternatively)
    },
    {
        // App-driven image upload example
        "type": "image",

        // Localized Call To Action to appear on the app
        "name": {
            "default": "Verify my identity",  // used if none of the languages matches
            "fr": "Vérifier mon identité"
        },

        // Requested image types to provide
        "source": [

            // An entry example expecting a picture from the front camera, overlaying a face silhouette
            // on the screen and identified by the "face-portrait" field name in the JSON payload
            {
                "type": "front-camera",
                "name": "face-portrait",
                "orientation": "portrait",
                "overlay": "face",
                "caption": {
                    "default": "...",
                    "fr": "..."
                }
            },

            // Example expecting two more pictures using the back camera and overlaying
            // either sides of an ID card respectively
            {
                "type": "back-camera",
                "name": "id-front",
                "orientation": "landscape",
                "overlay": "id-card-front",
                "caption": {
                    "default": "...",
                    "fr": "..."
                }
            },
            {
                "type": "back-camera",
                "name": "id-back",
                "orientation": "landscape",
                "overlay": "id-card-back",
                "caption": {
                    "default": "...",
                    "fr": "..."
                }
            },

            // Example requesting one more image from the phone's library
            {
                "type": "gallery",
                "name": "custom-1",
                "caption": {
                    "default": "...",
                    "fr": "..."
                }
            }
        ],

        // Endpoint accepting POST requests with JSON payload:
        // {
        //   name1: "base64-image-payload",
        //   name2: "base64-image-payload",
        //   ...
        // }
        // 
        // The URL will receive the following query string parameters:
        // - signature = sign(hash(jsonBody), privateKey)
        // - publicKey
        "url": "https://census-registry.cloud/lambda/upload-kyc-pictures/",

        // Endpoint to POST to with publicKey and signature+timestamp fields
        // Returning true will show the action and hide it otherwise
        "visible": "https://census-registry.cloud/lambda/image-upload-visible/"
    }]
}
```

## ENS

> ENS’s job is to map human-readable names like ‘alice.eth’ to machine-readable identifiers such as Ethereum addresses, content hashes, and metadata. ENS also supports ‘reverse resolution’, making it possible to associate metadata such as canonical names or interface descriptions with Ethereum addresses.

ENS exist to suit our exact purpose, we endorse it, but we may not make complete use of it.

### Overview

There are two smart contracts that make it work:

> The `ENS registry` is deliberately straightforward and exists only to map from a name to the resolver responsible for it.

> `Resolvers` are responsible for the actual process of translating names into addresses. Any contract that implements the relevant standards may act as a resolver in ENS.

The usual interaction will be:

1. Ask to the registry what resolver is responsible for a domain
2. Ask for a property of this resolver. Usually, this is an address, but it can be anything. And this is what we can make use of

![Basic ENS interaction](https://lh5.googleusercontent.com/_OPPzaxTxKggx9HuxloeWtK8ggEfIIBKRCEA6BKMwZdzAfUpIY6cz7NK5CFmiuw7TwknbhFNVRCJsswHLqkxUEJ5KdRzpeNbyg8_H9d2RZdG28kgipT64JyPZUP--bAizozaDcxCq34)

### No Mainnet

Vocdoni may be used in different networks other than the Ethereum Mainnet.

This presents a potential attack vector where a malicious entity supplanting a Mainnet domain in an alternative network. This is is very relevant to us because we will have users that are not illiterate about it, and communicating the difference between networks is very hard.

Therefore, we understand that we should only make use of ENS domains in the Ethereum Mainnet, independently from the network where the rest of Vocdoni smart contracts are deployed.

At the same time, we understand that some implementations of Vocdoni may not have a use for ENS, therefore we should not rely on it for any fundamental architectural design.

We do make use of the ENS resolvers and we follow the same architecture since there is a lot of value in potentially using ENS domains.

### ENS domain authentication

If ENS domains are used in the future, the `entityId` would be the [ENS node](https://docs.ens.domains/terminology), otherwise, the `entityId` is the hash of the address that created the entity.

`ENS public resolver` permission access to edit and add records is through the ENS registry. The `msg.sender` [needs to match](https://github.com/ensdomains/resolvers/blob/180919414b7f1dec80100b4aeff081d5afa8f3ce/contracts/PublicResolver.sol#L23) the owner of the ENS domain

### Comparison

We mostly make use of ENS resolvers to store metadata.

Some implementations may decide to make use of ENS domains

|                                   | Minimal             | ENS support |
|-----------------------------------|---------------------|-------------|
| On Mainnet                        |                     | ✓           |
| Usage of ENS domains              |                     | ✓           |
| **Resolver**                      |                     |             |
| EntityId  (node)                  | Record creator hash | ENS node    |
| Authentication                    | Record creator      | ENS owner   |
| Storage of text records           | ✓                   | ✓           |
| Storage of array of text records  | ✓                   | ✓           |
| Other Vocdoni specific interfaces | ✓                   | ✓           |