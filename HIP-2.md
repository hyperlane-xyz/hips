| hip | title                  | status | author | created    |
| --- | ---------------------- | ------ | ------ | ---------- |
| 2   | Merge Outbox and Inbox | Draft  | asaj   | 2022-10-19 |

### **Brief Summary / Abstract**

Proposes merging the Outbox and Inbox contracts into a single Mailbox contract.
This reduces the API surface area from O(n^2) to O(n) contract addresses, and simplifies the process for deploying Hyperlane to new chains.

### **Motivation**

Merging the Outbox and Inbox into the same contract simplifies the protocol.
This simplification can be felt by two parties.

#### Interchain developers

Currently, interchain app developers integrating Hyperlane need to be aware of O(n^2) addresses, where n is the number of chains that their app spans.

On each chain, the app needs to be aware of:

1. The local chain's Outbox contract, so the app knows where to send messages
2. The remote chains' Inbox contracts, so that the app knows what contracts are allowed to deliver messages.

To date, the recommended solution for tracking these addresses has been to use a ConnectionManager contract, which exposes view calls that allow the app to query for the Outbox address and whether or not an address is an Inbox.
While this allows applications to only track a single address, we've observed that the additional complexity of understanding the ConnectionManager and its API is a barrier to adoption.
Furthermore, use of a shared ConnectionManager comes with additional trust assumptions (i.e. that the owner will not register fraudulant Inboxes).
Developers that wish to avoid those trust assumptions must deploy and configure their own ConnectionManagers, and thus must deal with O(n^2) contract addresses.

With a single Mailbox contract, app developers can simply initialize their contracts with the local chain's Mailbox address, and send and receive messages from that single contract.

#### Hyperlane deployers

At present, to deploy Hyperlane to a new chain requires the following contract deployments:

1. An Outbox on the new chain
2. For each remote chain, an Inbox on the new chain
3. For each remote chain, an Inbox on the remote chain

Requiring multiple contract deployments on multiple chains makes new deployments of Hyperlane impractical without complex automated tooling.

With a single Mailbox contract, this can be reduced to:

1. A Mailbox on the new chain

Note that in either case, in order for the new deployment to be practically usable, we also need:

1. ISMs deployed to remote chains that can validate outbound messages
2. Metadata generators for the new chain (e.g. validators, provers for zkp-light-client ISMs, etc.)
3. ISMs deployed to the new chain that can validate inbound messages
4. One or more relayers to support relaying messages to/from the new chain

### **Tech Spec**

Implementation requires a pretty straightforward merge of the existing Outbox and Inbox contracts, with a few minor suggested changes:

1. Caching of checkpoints should be moved to a separate, slashing-specific contract.
2. The concept of "failed" state should be removed, as with sovereign consensus there is no enshrined security model that can fail

```
interface Mailbox {
    // For replay protection
    function delivered(bytes32 _messageId) external view returns (bool);

    // Sends a message
    function dispatch(
        uint32 _destinationDomain,
        bytes32 _recipientAddress,
        bytes calldata _messageBody
    ) external;

    // Delivers a message
    function process(
        bytes32 _root,
        bytes32[32] calldata _proof,
        bytes calldata _message,
        bytes calldata _metadata
    ) external;
}
```

### **Rationale**

There are conceptual advantages to the current multi-inbox design, in that the overal Hyperlane network to be decomposed into collections of contracts that support outbound messaging from a single chain, without overlap in these collections.
These are, however, outweighed by the practical drawbacks of a system that includes so many different contracts.

### **Security Considerations**

In order to prevent v1 messages from being re-played in v2, a version number should be added to >= v2 Hyperlane deployments.
This version number should included in validator signatures, so that attestations can not be re-used between versions.
Similarly, messages should be prefixed with the version number, so that the merkle root of outbound messages is tied to that version.
The version number in the message should be checked against the Mailbox contract's version.
