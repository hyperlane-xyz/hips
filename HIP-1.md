| hip | title               | status | author | created    |
| --- | ------------------- | ------ | ------ | ---------- |
| 1   | Sovereign consensus | Draft  | asaj   | 2022-10-04 |

### **Brief Summary / Abstract**

Defines the specification for sovereign consensus and interchain security modules, which allows message recipients to configure their own security models.

### **Motivation**

Interchain security models offer different tradeoffs with respect to security, latency, and cost. For example:

1. Multisig models prioritize latency and cost over security
2. Bonding models (e.g. proof-of-stake) prioritize latency and security over cost
3. Optimistic models prioritize security and cost over latency
4. Natively verified models prioritize security over cost (and sometimes latency)

Interchain applications necessarily adopt the tradeoffs of their interchain security models. Rather than offering a one-size-fits-all product, sovereign consensus allows applications to choose (or construct) a security model that offers the best choice of tradeoffs for that application.

Furthermore, the modularity of sovereign consensus allows Hyperlane to be forward compatible with new security models as they emerge.

### **Tech Spec**

Sovereign consensus is powered by `Interchain Security Modules` or `ISMs`, which specify the rules by which a `recipient` will accept an interchain message.

`ISMs` must implement the folling interface:

```
interface IInterchainSecurityModule {
    /**
     * @notice Returns an enum that represents the type of security model
     * encoded by this ISM.
     * @dev Relayers infer how to fetch and format metadata.
     */
    function type() external view returns (uint8);

    /**
     * @notice Defines a security model that decides whether or not to accept
     * an interchain message based on the provided metadata.
     * @param _metadata Off-chain metadata provided by a relayer, specific to
     * the security model encoded by the module (e.g. validator signatures)
     * @param _message Hyperlane encoded interchain message
     * @return True if the message should be accepted
     */
    function accept(
        bytes calldata _metadata,
        bytes calldata _message
    ) external returns (bool);
}
```

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
