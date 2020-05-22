# Voting Process

- [Voting Process](#voting-process)
  - [Smart Contract](#smart-contract)
    - [Internal structs](#internal-structs)
    - [ProcessId](#processid)
    - [Current implementation](#current-implementation)
  - [Data schema](#data-schema)
    - [Process metadata (JSON)](#process-metadata-json)
    - [Vote Envelope](#vote-envelope)
      - [Containing Snark votes](#containing-snark-votes)
      - [Containing Poll votes](#containing-poll-votes)
    - [Vote Package](#vote-package)
      - [Snark Vote](#snark-vote)
      - [Poll Vote](#poll-vote)
      - [Petition Sign](#petition-sign)
    - [Results (JSON)](#results-json)
  - [Future work](#future-work)
    - [Define a versioning system](#definie-a-versioning-system)
    - [Define sanity checks](#define-sanity-checks)
    - [Oracles protocol](#oracles-protocol)
    - [Support multi-layer vote encryption](#support-multi-layer-vote-encryption)

Voting processes are declared on the Blockchain and store the critical information for integrity. However, the metadata of a process lives on a JSON file with the information on which voters can make a choice. It also allows validators to fetch the technical parameters of a vote.

The starting point is the **[Voting Process](#smart-contract)** contract, but it is tightly coupled with the **[JSON Process Metadata](#data-schema)** living on P2P filesystems.

## Smart Contract

Used as a registry of voting processes, associated to the entity with the same Ethereum address as the transaction sender.

The address of the Voting process contract instance is resolved from `voting-process.vocdoni.eth` on the ENS registry.

### Internal structs

```solidity

// GLOBAL STRUCTS

struct Process {
    uint8 envelopeType;         // One of valid envelope types, see: https://vocdoni.io/docs/#/architecture/components/process
    uint8 mode;                 // 0 [scheduled-single-envelope], 1 [ondemand-single-envelope]
    address entityAddress;      // The Ethereum address of the Entity
    uint256 startBlock;         // Tendermint block number on which the voting process starts
    uint256 numberOfBlocks;     // Amount of Tendermint blocks during which the voting process is active
    string metadata;            // Content Hashed URI of the JSON meta data (See Data Origins)
    string censusMerkleRoot;    // Hex string with the Merkle Root hash of the census
    string censusMerkleTree;    // Content Hashed URI of the exported Merkle Tree (not including the public keys)
    uint8 status;               // 0 [open], 1 [ended], 2 [canceled], 3 [paused]
    string results;             // String containing the actual results
}

// GLOBAL DATA

address contractOwner;
string[] validators;            // Public key array
address[] oracles;              // Public key array
string genesis;                 // Content Hashed URI
uint256 chainId;
mapping(uint8 => bool) envelopeTypes; // valid envelope types
mapping(uint8 => bool) modes;         // valid process modes

// PER-PROCESS DATA

Process[] public processes; // Array of Process struct
mapping(bytes32 => uint256) processesIndex; // Mapping of processIds with processess idx
mapping(address => uint256) public entityProcessCount; // index of the last process for a given address

```

### ProcessId

Processes are uniquely identified by their `processId`

To guarantee its uniqueness is generated out of:
- `entityId`
- `idx`
- `genesis`
- `chainId`

```solidity
function getNextProcessId(address entityAddress) public view returns (bytes32){
    // From 0 to N-1, the next index is N
    uint nextIdx = entityProcessCount[entityAddress];
    
    return keccak256(abi.encodePacked(entityAddress, nextIdx, genesis, chainId));
}

function getProcessId(address entityAddress, uint processCountIndex) public view returns (bytes32) {
    return keccak256(abi.encodePacked(entityAddress, processCountIndex, genesis, chainId));
}

```

where `entityProcessCount` is an incremental nonce per `entityAddress`.

### Current implementation

[VotingProcess.sol](https://gitlab.com/vocdoni/dvote-solidity/raw/master/contracts/VotingProcess.sol)

## Data schema

### Process metadata (JSON)

The creation of this document is critical. Multiple checks should be in place to ensure that the data is coherent (well formatted, all relevant locales present, etc).

The JSON payload below is stored on IPFS.

```json
{
    "version": "1.0", // Protocol version
    "type": "snark-vote", // One of: snark-vote, poll-vote, encrypted-poll, petition-sign
    "startBlock": 10000, // Vochain block number to start the voting process
    "numberOfBlocks": 400,
    "census": {
        "merkleRoot": "0x1234...",
        "merkleTree": "ipfs://1234,https://server/file.dat!sha3-hash" // Content Hashed URI of the exported Merkle Tree
    },
    "details": {
        "entityId": "0x123",
        "title": {
            "en": "Universal Basic Income",
            "ca": "Renda Bàsica Universal"
        },
        "description": {
            "en": "## Markdown text goes here\n### Abstract",
            "ca": "## El markdown va aquí\n### Resum"
        },
        "headerImage": "<content uri>",
        "questions": [
            {
                "type": "single-choice", // Defines how the UI should allow to choose among the votingOptions.
                "question": {
                    "en": "Should universal basic income become a human right?",
                    "ca": "Estàs d'acord amb que la renda bàsica universal sigui un dret humà?"
                },
                "description": {
                    "en": "## Markdown text goes here\n### Abstract",
                    "ca": "## El markdown va aquí\n### Resum"
                },
                "voteOptions": [
                    {
                        "title": {
                            "en": "Yes",
                            "ca": "Sí"
                        },
                        "value": "1"
                    },
                    {
                        "title": {
                            "en": "No",
                            "ca": "No"
                        },
                        "value": "2"
                    }
                ]
            }
        ]
    }
}
```

### Vote Envelope

The Vote Envelope wraps different types of vote packages and features certain fields, depending on the underlying Vote Package Type

#### Containing Snark Votes

```json
{
    "processId": "0x1234567890...",
    "proof": "0x1234...",  // ZK Proof
    "nonce": "1234567890",  // Unique number per vote attempt, so that replay attacks can't reuse this payload
    "nullifier": "0x1234...",   // Hash of the private key + processId
    "encryptionKeyIndexes": [0, 1, 2, 3, 4],  // (optional) On encrypted votes, contains the (sorted) indexes of the keys used to encrypt
    "votePackage": "base64-vote-package"  // base64(jsonString) or base64( encrypt(jsonString) )
}
```

#### Containing Poll votes

The Vote Envelope of a Poll vote features the process ID, the Census Merkle Proof of the user, a nonce to prevent replay attacks and a Base64 representation of the Vote Package. The signature should be generated from a JSON object containing the keys in ascending alphabetical order.

```json
{
    "processId": "0x1234567890...",
    "proof": "0x1234...",  // Merkle Proof
    "nonce": "1234567890",  // Unique number per vote attempt, so that replay attacks can't reuse this payload
    "encryptionKeyIndexes": [0, 1, 2, 3, 4],  // (optional) On encrypted polls, contains the (sorted) indexes of the keys used to encrypt
    "votePackage": "base64-vote-package",  // base64(jsonString) or base64( encrypt(jsonString) )
    "signature": "0x12345678...",  // sign( JSON.stringify( { processId, proof, nonce, encryptionKeyIndexes?, votePackage } ), privateKey )
}
```

The `nullifier` to identify the vote in the blockchain is computed as follows: 

`nullifier = keccak256(bytes(hex(addr(signature))) + bytes(hex(processId)))`

### Vote Package

Is the actual data contained within a vote Envelope.

#### Snark Vote

Used for anonymous votes using ZK Snarks to restrict voters to only those on the Merke Tree.

```json
{
    "type": "snark-vote",
    "nonce": "01234567890abcdef", // 8+ byte random string to prevent guessing the encrypted payload before the reveal key is released
    "votes": [  // Direclty mapped to the `questions` field of the metadata
        1, 3, 2
    ]
}
```

- On encrypted votes, public keys must be used in ascending order.

#### Poll Vote

Non-anonymous votes, where the Merkle Proof and the signature are enough.

```json
{
    "type": "poll-vote",
    "nonce": "01234567890abcdef", // 8+ byte random string to prevent guessing the encrypted payload before the reveal key is released
    "votes": [  // Direclty mapped to the `questions` field of the metadata
        1, 3, 2
    ]
}
```

- The `nonce` is mandatory if the poll is encrypted. Can be omitted otherwise.
- On encrypted polls, public keys must be used in ascending order.

#### Petition Sign

(Coming soon)

<!--

(Petition signing)

```json
{
    "type": "petition-sign", // One of: snark-vote, poll-vote, encrypted-poll, petition-sign
    "nullifier": "0x1234...",   // Hash of the private key
    "votes": [  // Direclty mapped to the `questions` field of the metadata
        1
    ]
}
```

-->

### Results (JSON)

- Its hash is published in the voting-process smart-contract
- Sorting must match the `ProcessMetadata` questions.

```json

{
    "questions":[
        {
            "invalid": 0, 
            "results": [
                {
                    "count": 12342,
                    "value": "0" // Mataches the value in voteOption in ProcessMetadata
                },
                {
                    "count": 4321,
                    "value": "1"
                }
            ]
        },
        {
            "invalid": 0,
            "results": [
                {
                    "count": 12342,
                    "value": "0" 
                },
                {
                    "count": 4321,
                    "value": "1"
                }
            ]
        }
    ]
}

```

## Future work

### Define a versioning system
### Define sanity checks
There are serveral events where the process may be invalid.

- Process-metadata can't be fetched from IPFS
- Process-metadata field can't be parsed or does not exist
- Block-times have an invalid range. Define how far in the future they want the

Oracles must report them, and action should be taken if there is consensus.

### Oracles protocol

Most of the sanity check logic can't no longer be in the ethereum blockchain, therefore we need to define process/consensus mechanism for the oracles to report to the ethereum smart-contracts.

Oracles therefore must have an ethereum account, and it should be registered together with its public key

Oracles can complain when there is a problem.
```
complain(uint reason){}
```

If a pre-defined percentage of oracles agrees on a problem of a specific reason, the process can be invalidated.

```
invalidate(uint reason){}
```

Before the user is shown a process, it should've been validated that the process is all good. Therefore is probably a good idea that the validators approve the process at the beginning.

```
complain(0);
```

The same logic should be used for reporting the results

```
reportResults(string resultsHash){}
```

```
validateResults(){}
````

### Support multi-layer vote encryption

Define how multiple entities can publish the public and private key for vote encryption. So no single entity has privilege information.
