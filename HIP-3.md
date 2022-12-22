| hip | title        | status | author | created    |
| --- | ------------ | ------ | ------ | ---------- |
| 3   | Multisig ISM | Draft  | asaj   | 2022-12-20 |

### **Brief Summary / Abstract**

Defines the specification for a multisig-based interchain security module.

### **Motivation**

Multisigs are the simplest form of security model, one that is already supported by Hyperlane. The goal of this HIP is to standardize the `MultisigIsm` interface so that more variants may be implemented.

### **Tech Spec**

A `MultisigIsm` must return `3` in `moduleType()`.

A `MultisigIsm` must implement the following interface. Relayers should use the `validatorsAndThreshold()` view function to understand what metadata the `ISM` needs in order to verify a message.

```
interface IMultisigIsm is IInterchainSecurityModule {
    /**
     * @notice Returns the set of validators responsible for verifying _message
     * and the number of signatures required
     * @dev Can change based on the content of _message
     * @param _message Hyperlane formatted interchain message
     * @return validators The array of validator addresses
     * @return threshold The number of validator signatures needed
     */
    function validatorsAndThreshold(bytes calldata _message)
        external
        view
        returns (address[] memory validators, uint8 threshold);
}
```

A `MultisigIsm` must expect the metadata passed to `verify()` to be formatted in the following way:

```
/**
 * [   0:  32] Origin mailbox merkle root
 * [  32:  36] Origin mailbox merkle root index
 * [  36:  68] Origin mailbox address
 * [  68:1092] Merkle proof of _message.id()
 * [1092:1093] Threshold needed to verify _message
 * [1093:????] Validator signatures, 65 bytes each, with length == Threshold.
 * [????:????] Validator addresses for the entire set, left padded to bytes32
 */
```

Relayers must order the validator set addresses in metadata to match the `MultisigIsm's` ordering stored on-chain. Relayers must order validator signatures in metadata similarly.

To minimize cold SLOADs and save gas, a `MultisigIsm` should store a commitment to each validator set, and compare that commitment to the validator set provided in metadata.

```
/// @notice A commitment to this MultisigIsm's validator set.
/// @dev Note that more complex MultisigIsms may have validator sets and
/// thresholds that change with the message content. In those cases, the
/// corresponding commitments can be compared in verify() instead.
bytes32 public commitment;

function verify(bytes calldata _metadata, bytes calldata _message)
    external
    returns (bool)
{
    uint8 _metadataThreshold = _metadata.threshold();
    address[] memory _metadataValidators = _metadata.validators();

    bytes32 _metadataCommitment = keccak256
    bytes32 _metadataCommitment = keccak256(
        abi.encodePacked(_metadataThreshold, _metadataValidators)
    );
    require(_metadataCommitment == commitment);
}
```

A `MultisigIsm` should verify the merkle proof included in the metadata that `_message.id()` is present in the merkle tree at leaf index `_message.nonce()`, e.g.:

```
import {MerkleLib} from "../libs/Merkle.sol";

function _verifyMerkleProof(
    bytes calldata _metadata,
    bytes calldata _message
) internal pure returns (bool) {
    // calculate the expected root based on the proof
    bytes32 _calculatedRoot = MerkleLib.branchRoot(
        _message.id(),
        _metadata.proof(),
        _message.nonce()
    );
    return _calculatedRoot == _metadata.root();
}
```

Finally, a `MultisigIsm` should verify that at least `metadata.threshold()` signatures were provided on `metadata.root()`, according to [HIP-TBD](https://link-to-signature-hip).

### **Rationale**

#### \_message as an argument to validatorsAndThreshold()

Passing `_message` as argument to `validatorsAndThreshold()` allows for a broad array of customizability.

This allows users to create `MultisigIsms` that are capable of verifying messages from many remote domains (by switching on `message.origin()`).

It also allows users to create `MultisigIsms` that are aware of the semantic meaning of the interchain message, allowing them to e.g. increase or decrease the threshold based on message value.

This could also be accomplished using a `RoutingIsm`, which would redirect to other ISMs based on `_message`, but that could be somewhat less efficient. For example, if you wanted to vary `threshold` based on `_message.body()` using a `RoutingIsm`, you would need to deploy and configure a `MultisigIsm` for each value of `threshold` you wanted to support.

#### Passing the validator set as calldata

Passing the threshold and array of validator addresses in calldata allows `MultisigIsm` implementations to be more gas efficient. Implementations can compare the set provided in calldata (4 gas / byte) against a single stored commitment (1 cold SLOAD) instead of having to look up each validator address in storage (many cold SLOADs).

The tradeoff is that the `verify()` function becomes more complex. Implementations may choose not to use commitments, and instead read the validator set from storage, ignoring the set provided in the metadata. This is simpler but less gas efficient.

### **Backwards Compatibility**

This is different from the current, non-standardized implementation of `MultisigIsm`, and is not backwards compatible. Relayers should (temporarily) support both the old and new implementations.

### **Security Considerations**

Any `MultisigIsm` implementation that complies with this specification will be able to verify Hyperlane interchain messages. This allows for a lot of customizability, but bugs in any implementation may allow fraudulent messages to be passed to recipients that use that implementation.

Great care should be taken when implementing an `ISM`. Hyperlane users are encouraged to use the audited implementations in hyperlane-monorepo when possible.
