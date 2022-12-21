| hip | title                | status | author | created    |
| --- | -------------------- | ------ | ------ | ---------- |
| 2   | Validator signatures | Draft  | asaj   | 2022-12-20 |

### **Brief Summary / Abstract**

Defines the specification for Hyperlane validator signatures.

### **Motivation**

Multisigs are the simplest form of security model, one that is already supported by Hyperlane. The goal of this HIP is to standardize the `MultisigIsm` interface so that more variants may be implemented.

### **Tech Spec**

A `MultisigIsm` must expect the metadata passed to `accept()` to be formatted in the following way:

```
/**
 * [   0:  32] Merkle root
 * [  32:  36] Root index
 * [  36:  68] Origin mailbox address
 * [  68:1092] Merkle proof
 * [1092:1093] Threshold
 * [1093:????] Validator signatures, 65 bytes each, with length == Threshold
 * [????:????] Addresses of the entire validator set, left padded to bytes32
 */
```

A `MultisigIsm` should verify the the merkle proof against `root`,

A `MultisigIsm` must implement the following interface:

```
interface IMultisigIsm is IInterchainSecurityModule {
    /**
     * @notice Returns the set of validators responsible for securing _message
     * @dev Can change based on the content of _message
     * @param _message Hyperlane formatted interchain message
     * @return The set of validators responsible for securing _message
     */
    function validators(bytes calldata _message) external view returns (address[] memory);

    /**
     * @notice Returns the number of validator signatures needed to accept
     * _message
     * @dev Can change based on the content of _message
     * @param _message Hyperlane formatted interchain message
     * @return The number of signatures needed to accept _message
     */
    function threshold(bytes calldata _message) external view returns (uint8);
}
```

When attempting to deliver a message to a recipient that uses a `MultisigIsm`, relayers should query the `validators()` and `threshold()` view functions to

Message `recipients` should specify their `ISM` via the `interchainSecurityModule()` view function.

```
interface ISpecifiesInterchainSecurityModule {
    /**
     * @notice Returns the ISM to use for this recipient of interchain messages.
     * @dev Optional, if not implemented the Mailbox's default ISM will be used.
     */
    function interchainSecurityModule() external view returns (IInterchainSecurityModule);
}
```

The `Mailbox` contract, when delivering a message via `Mailbox.process()`, must check to see if the message recipient implements the `interchainSecurityModule()` view function. If it does, and returns a non-zero address, the `Mailbox` must call `accept()` on that address. Otherwise, it must call `accept()` on the `Mailbox's` default `ISM`.

### **Rationale**

Sovereign consensus allows for a modular approach to interchain security.
This allows the core protocol to be forward compatible, ensuring that Hyperlane can support new security models as they are developed.
Furthermore, it allows applications to select, configure, or invent security models that offer the most appropriate tradeoffs.

We discuss a few of the design decisions made below.

#### Finite ISM types

This proposal relys on their being a finite set of ISM types that the relayer knows how to fetch and format metadata for.

Alternatively, one could imagine a CCIP-read (or similar) based protocol for expressing arbitrary metadata, allowing for the definition of arbitrary ISMs.

We chose this approach because we imagine a small number of ISM types being sufficiently expressive to encode most (if not all) security models.

### **Backwards Compatibility**

For backwards compatibility with V1, each `Mailbox` should be configured with a default `ISM`.
If `recipient.interchainSecurityModule()` reverts or returns the zero address, the default `ISM` should be used.

### **Security Considerations**

We believe that modularity is the best security philosophy for interchain messaging, as it has the following properties:

- Fault isolation: The impact of `ISM` failure is limited to those applications using the `ISM`.
- Forwards compatibility: Sovereign consensus allows for security models to change as new and better solutions emerge (e.g. zkp-based light clients).
- Customizability: Applications can tailor security models to their needs, rather than relying on a one-size-fits-all solution.

### **Future work**

Individual `ISM` specifications should follow in future HIPs.

Some possible ISMs to define:

- Multisig: Given a message, specifies a set of signers and a threshold that need to have signed a merkle root, in order to accept a message proved against that root. Note that the set and threshold can vary based on message content.
- Optimistic: Given a message, specifies a set of watchers that can pause the `ISM`. Otherwise, merkle roots are accepted, and after a fraud window has expired messages can be proved against these roots.
- ZKP: Verifies a zero-knowledge-proof of the origin chain's light client protocol, with a merkle proof of the corresponding Hyperlane Outbox's merkle root.
- Routing: Routes messages to one or more other `ISMs` according to their content.
- Combo: Requires acceptance from multiple `ISMs`
