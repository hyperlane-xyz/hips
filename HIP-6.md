| hip | title                   | status | author | created    |
| --- | ----------------------- | ------ | ------ | ---------- |
| 6   | Validator announcements | Draft  | asaj   | 2022-12-21 |

### **Brief Summary / Abstract**

Defines an optional specification for Hyperlane validators to announce themselves and the location of their signed checkpoints.

### **Motivation**

Hyperlane relayers need to aggregate validator signed checkpoints in order to deliver interchain messages.

At present, relayers are manually configured with a list of validators and the location of their signed checkpoints.

Relying on manual configuration creates friction to the introduction of additional validators via sovereign consensus.

This HIP defines a standard for validators to announce themselves to relayers via a smart contract on the underlying blockchain.

Relayers should query this contract to discover the location of validator signed checkpoints.

### **Tech Spec**

#### Announcment attestations

The validator announcement protocol relies on validators signing attestations with the following semantic meaning:

> "Checkpoints for the mailbox contract with address _m_ on chain _d_, signed by the validator with address _v_, can be found at location _l_

The Hyperlane validator binary should sign the following data upon startup.

```solidity
/**
 * @notice Returns the digest that Hyperlane validators should sign to support
 * the announcement protocol
 * @param _domain The origin domain of the Mailbox being validated
 * @param _mailbox The address of the Mailbox being validated, as bytes32
 * @param _storageMetadata Information encoding the location of signed
 * checkpoints, specific to the storage modality (e.g. S3 bucket URL)
 * @return The digest to EIP-191 sign
 */
function getDigestToSign(
    uint32 _domain,
    bytes32 _mailbox,
    string calldata _storageMetadata
)
    external
    pure
    returns (bytes32)
{
    bytes32 _domainHash = keccak256(
        abi.encodePacked(_domain, _mailbox, "HYPERLANE")
    );
    return keccak256(abi.encodePacked(_domainHash, _storageMetadata));
}
```

#### Storage metadata

To be forward compatible with alternative storage modalities, the announcement protocol accepts arbitrary strings as metadata.

However, in order to this metadata to be understood by relayers, a common standard should be adopted for each storage modality supported by relayers.

The modalities currently supported are local storage and AWS S3. Their storage metadata formats should be, respectively:

> "file://{path_to_file}"
> "s3://{bucket_name}/{bucket_region}"

Future storage modalities should be announced using the "{modality_type}://" format.

#### ValidatorRegistry

A contract that implements the `IValidatorRegistry` interface should be deployed to each chain.

This contract stores validator registrations for that chain and mailbox contract.

Validators for that domain and mailbox contract should register themselves so that they are discoverable by relayers.

```solidity
interface IValidatorRegistry {
    /// @notice Returns the local domain for validator registrations
    function localDomain() external view returns (uint32);

    /// @notice Returns the mailbox contract for validator registrations
    function mailbox() external view returns (address);

    /// @notice Returns a list of validators that have registered
    function validators() external view returns (address[] memory);

    /**
     * @notice Returns a list of all registrations for all provided validators
     * @param _validators The list of validators to get registrations for
     * @return A list of validator addresses and registered storage metadata
     */
    function getValidatorRegistrations(
        address[] calldata _validators,
    ) external view returns (
        address[] memory validators,
        string[][] memory storageMetadata
    );

   /**
    * @notice Registers a validator
    * @param _storageMetadata Information encoding the location of signed
    * checkpoints
    * @param _signature The signed validator announcement attestation
    * previously specified in this HIP
    * @returns True upon success
    */
    function registerValidator(
        string calldata _storageMetadata
        bytes calldata _signature
    ) external returns (bool);
}
```

### **Rationale**

#### Registration of multiple storage locations

This proposal allows validators to register multiple storage locations.

Validators may choose to post their signed checkpoints in multiple locations (e.g. S3, GCP, IPFS) in order to maximize liveness.

#### Deregistration

For simplicity, this proposal intentionally omits the ability to deregister a storage modality.

Relayers can optionally blacklist registered validators if they discover them to no longer be signing checkpoints.

### **Backwards Compatibility**

This HIP is backwards compatible, validators can register themselves at any time.

Eventually, relayers may choose to stop supporting unregistered validators.

At this point, validators would need to register in order to make meaningful contributions to securing Hyperlane.

### **Security Considerations**

None, an incorrect implementation of this proposal would only affect liveness.
