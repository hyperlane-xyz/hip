| hip | title                | status | author | created    |
| --- | -------------------- | ------ | ------ | ---------- |
| 2   | Validator signatures | Draft  | asaj   | 2022-12-20 |

### **Brief Summary / Abstract**

Defines the specification for Hyperlane validator signatures.

Hyperlane validators sign messages that act as attestations of the form:

> "On chain _d_, the mailbox contract at address _m_ had a messages tree with merkle root _r_ and message count _i_"

### **Motivation**

Hyperlane validators provide the foundation for two core interchain security modules types: Multisig, and Optimistic.

This HIP defines a standard for validator signatures so that they can be shared across different security module instances.

### **Tech Spec**

A Hyperlane validator should sign the following, in compliance with [EIP-191](https://eips.ethereum.org/EIPS/eip-191).

```solidity
/**
 * @notice Returns the digest that Hyperlane validators should sign
 * @param _domain The origin domain of the Mailbox being validated
 * @param _mailbox The address of the Mailbox being validated, as bytes32
 * @param _root The merkle root that the validator is attesting to
 * @param _index The message count that the validator is attesting to
 * @return The digest to EIP-191 sign
 */
function getDigestToSign(
    uint32 _domain,
    bytes32 _mailbox,
    bytes32 _root,
    uint32 _index
)
    external
    pure
    returns (bytes32)
{

    bytes32 _domainHash = keccak256(
        abi.encodePacked(_domain, _mailbox, "HYPERLANE")
    );
    return keccak256(abi.encodePacked(_domainHash, _root, _index));
}
```

A `ValidatorSignatureVerifier` contract implementing the following interface should be deployed to each chain supporting Hyperlane. This contract must implement the following interface and emit the `ValidatorSignature` event when verifying a signature.

Verifying signatures via `ValidatorSignatureVerifier` allows watchers to more easily monitor for fraudulent validator signatures.

```solidity
interface IValidatorSignatureVerifier {
    /**
     * @notice Emitted when a validator signature is verified
     * @dev Used by watchtowers to detect fraudulent validators
     * @param domain The origin domain of the Mailbox being validated
     * @param mailbox The address of the Mailbox being validated, as bytes32
     * @param root The merkle root that the validator is attesting to
     * @param index The message count that the validator is attesting to
     * @param signature The 65-byte ECDSA validator signature
     */
    event ValidatorSignature(
        uint32 domain,
        bytes32 mailbox,
        bytes32 root,
        uint32 index
        bytes signature
    );

    /**
     * @notice Emitted when one or more validator signatures are verified
     * @dev Used by watchtowers to detect fraudulent validators
     * @param domain The origin domain of the Mailbox being validated
     * @param mailbox The address of the Mailbox being validated, as bytes32
     * @param root The merkle root that the validator is attesting to
     * @param index The message count that the validator is attesting to
     * @param signature 65-byte ECDSA validator signatures
     */
    event ValidatorSignatures(
        uint32 domain,
        bytes32 mailbox,
        bytes32 root,
        uint32 index
        bytes[] signatures
    );

    /**
     * @param _domain The origin domain of the Mailbox being validated
     * @param _mailbox The address of the Mailbox being validated, as bytes32
     * @param _root The merkle root that the validator is attesting to
     * @param _index The message count that the validator is attesting to
     * @param _signature The 65-byte ECDSA validator signature
     * @return The address of the validator that signed
     */
    function recoverValidatorFromSignature(
        uint32 _domain,
        bytes32 _mailbox,
        bytes32 _root,
        uint32 _index,
        bytes calldata _signature
    ) external returns (address);

    /**
     * @param _domain The origin domain of the Mailbox being validated
     * @param _mailbox The address of the Mailbox being validated, as bytes32
     * @param _root The merkle root that the validator is attesting to
     * @param _index The message count that the validator is attesting to
     * @param _signatures 65-byte ECDSA validator signatures
     * @return Addresses of the validators that signed
     */
    function recoverValidatorsFromSignatures(
        uint32 _domain,
        bytes32 _mailbox,
        bytes32 _root,
        uint32 _index,
        bytes[] calldata _signatures
    ) external returns (address[] memory);

}
```

### **Rationale**

#### Inclusion of origin mailbox address

Including the origin mailbox address in the validator signature gives it clearer semantic meaning that can be verified:

> > "On chain _d_, the mailbox contract at address _m_ had a messages tree with merkle root _r_ and message count _i_"

The downside is that care will need to be taken in order to ensure that malicious validators cannot manipulate the mailbox address in their signature in order to avoid the consequences of fraud.

Take the following psuedo-code for a slashing contract:

```solidity
function slash(
    uint32 _domain,
    bytes32 _mailbox,
    bytes32 _root,
    uint32 _index,
    bytes calldata _signature
) external returns (address) {
    address _validator = signatureVerifier.recoverValidatorFromSignature(
        _domain,
        _mailbox,
        _root,
        _index,
        _signature
    );
    require(IMailbox(_mailbox).rootAt(_index) != _root);
    _slash(_validator);
}
```

Validators could avoid the consequences of signing a fraudulent checkpoint by deploying a contract that implements the `IMailbox` interface but allows them to push arbitrary messages to it. They could then sign a checkpoint for this contract and attempt to use that to deliver fraudulent messages on some destination chain.

There are two potential mitigations:

1. Encode remote mailbox addresses in ISMs
   This would ensure that ISMs only accept messages from mailboxes that the ISM deployer knows to be a correct implementation of `Mailbox`. This ensures that `slash()` will always succeed if the validator attempts to attest to a root containing fraudulent messages.

2. Encode supported mailboxes in the slashing contract
   This would mean that signing a checkpoint for an unsupported mailbox would be considered a slashable offense. This could be done either by hardcoding addresses in storage, or hardcoding supported mailbox implementation bytecode, presuming the contents of mailbox storage do not affect the trust assumptions of outbound messages (they do not as of 2022-12-21).

#### Encouraging verification through ValidatorSignatureVerifier

While less gas efficient, this ensures that there is a single contract that watchers can monitor for fraudulent validator signatures.

Fraudulent validator signatures may be used to slash the fraudulent validator, or as a trigger to disconnect an optimistic ISM.

Encouraging a single implementation also reduces the risk that validator-signature-based ISM implementers introduce bugs in signature verification.

### **Backwards Compatibility**

This format is not backwards compatible with V1, which is actually a good thing as it means validator signatures cannot be re-used.

The default MultisigIsm in the V2 pre-release does not use a `ValidatorSignatureVerifier`. This can be corrected in future ISM deployements.

### **Security Considerations**

A bug in `ValidatorSignatureVerifier` would risk compromising all validator-signature-based ISMs.
