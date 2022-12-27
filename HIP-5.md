| hip | title     | status | author | created    |
| --- | --------- | ------ | ------ | ---------- |
| 5   | Combo ISM | Draft  | asaj   | 2022-12-21 |

### **Brief Summary / Abstract**

Defines the specification for an interchain security module (ISM) that combines other ISMs.

### **Motivation**

The goal of this HIP is to standardize the `ComboIsm` interface, so that users can compose multiple ISMs.

For example, some users have expressed the desire to have a security model where n-of-m validators need to sign from some large, semi trusted set, but also one more more signatures from some smaller, more trusted set, are needed. This model can be expressed as the combination of two `MultisigIsms`.

### **Tech Spec**

A `ComboIsm` must return `2` in `type()`.

A `ComboIsm` must implement the following interface. Relayers should use the `combination()` view function to understand what metadata the `ISM` needs in order to verify a message.

```
interface IComboIsm is IInterchainSecurityModule {
    /**
     * @notice Returns the set of ISMs needed to verify _message
     * @dev Can change based on the content of _message
     * @param _message Hyperlane formatted interchain message
     * @return The set of ISMs needed to verify _message
     */
    function combination(
        bytes calldata _message
    ) external view returns (address[] memory interchainSecurityModules);
}
```

A `ComboIsm` must expect the metadata passed to `verify()` to be formatted in the following way:

```
/**
 * [   0:   1] Expected ISM count
 * [   1:????] Lengths, as uint32s, of each ISM's metadata
 * [????:????] Tightly packed metadata for each ISM
 */
```

Relayers must order the lengths and metadata in order to match the order returned by `combination(_message)`.

`ComboIsms` should avoid conditioning on frequently-changing state, as this may result in a relayer formatting metadata incorrectly.

### **Rationale**

### **Backwards Compatibility**

### **Security Considerations**
