| hip | title               | status | author | created    |
| --- | ------------------- | ------ | ------ | ---------- |
| 1   | Sovereign consensus | Draft  | asaj   | 2022-10-04 |

### **Brief Summary / Abstract**

Defines the specification for sovereign consensus, which puts control of the interchain security model in the hands of message recipients.

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

Message `recipients` should specify their `ISM` via the `interchainSecurityModule()` view function.

The `Inbox` contract, when delivering a message via `Inbox.process()`, calls `recipient.interchainSecurityModule().verify()`, which takes in the message and metadata and decides whether or not the message is valid.
Relayers can inject arbitrary off-chain metadata (e.g. validator signatures, zero-knowledge proofs, etc.) into `verify()`, allowing for a wide degree of flexibility in `ISM` design.

Some possible ISMs:

- Multisig: Given a message, specifies a set of signers and a threshold that need to have signed a merkle root, in order to accept a message proved against that root. Note that the set and threshold can vary based on message content.
- Optimistic: Given a message, specifies a set of watchers that can pause the `ISM`. Otherwise, merkle roots are accepted, and after a fraud window has expired messages can be proved against these roots.
- ZKP: Verifies a zero-knowledge-proof of the origin chain's light client protocol, with a merkle proof of the corresponding Hyperlane Outbox's merkle root.
- Routing: Routes messages to one or more other `ISMs` according to their content.

Individual `ISM` specifications should follow in future HIPs.

```
interface IInterchainMessageRecipient {
    // Optional. Returns the recipient's ISM, which checks the validity
    // of interchain messages.
    function interchainSecurityModule() external view returns (IInterchainSecurityModule);

    function handle(
        uint32 _origin,
        bytes32 _sender,
        bytes calldata _payload
    ) external;
}

interface IInterchainSecurityModule {
    // Called by the Inbox to determine whether the provided message is valid.
    function verify(
        bytes calldata _message,
        bytes calldata _metadata
    ) external returns (bool);
}

interface IInbox {

    // Called by relayers to deliver messages to the recipient.
    // Calls `recipient.interchainSecurityModule().accept()` before
    // calling `recipient.handle()`.
    function process(
        bytes calldata _message,
        bytes calldata _metadata
    ) external;
}
```

### **Rationale**

Sovereign consensus allows for a modular approach to interchain security.
This allows the core protocol to be forward compatible, ensuring that Hyperlane can support new security models as they are developed.
Furthermore, it allows applications to select, configure, or invent security models that offer the most appropriate tradeoffs.

There were many ways to build this feature.
We discuss a few of the design decisions made below.

#### Location of merkle proof verification

The ability for relayers to inject arbitrary metadata into the `ISM` presents a natural opportunity to move the merkle proof out of the `Inbox.process()` interface and into the `ISM`.
In this model, the `Inbox` would be responsible only for replay protection.

This approach has the advantage of being even more modular than the proposed solution, as the `Inbox` is unopinionated about how messages are sent.
This allows applications to create `ISMs` that do not incorporate the incremental merkle tree.
As a simple example, one could build an `ISM` that is a multisig on individual messages, as opposed to a merkle root.

The drawback is that those `ISMs` which wish to incorporate the incremental merkle tree into their security model must do that explicitly.
Furthermore, applications using `ISMs` that do not rely on the merkle tree will still pay the ~15k gas overhead of inserting their messages into the merkle tree on the origin chain.

### **Backwards Compatibility**

For backwards compatibility, each `Inbox` should be configured with a default `ISM`.
If `recipient.interchainSecurityModule()` reverts, the default `ISM` should be used.

### **Security Considerations**

We believe that modularity is the best security philosophy for interchain messaging, as it has the following properties:

- Fault isolation: The impact of `ISM` failure is limited to those applications using the `ISM`.
- Forwards compatibility: Sovereign consensus allows for security models to change as new and better solutions emerge (e.g. zkp-based light clients).
- Customizability: Applications can tailor security models to their needs, rather than relying on a one-size-fits-all solution.
