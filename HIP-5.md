| hip | title           | status | author | created    |
| --- | --------------- | ------ | ------ | ---------- |
| 5   | Aggregation ISM | Draft  | asaj   | 2022-12-21 |

### **Brief Summary / Abstract**

Defines the specification for an interchain security module (ISM) that aggregates other ISMs.

### **Motivation**

The goal of this HIP is to standardize the `AggregationISM` interface, so that users can aggregate multiple ISMs.

For example, some users have expressed the desire to have a security model where n-of-m validators need to sign from some large, semi trusted set, but also one more more signatures from some smaller, more trusted set, are needed. This model can be expressed as the aggregation of two `MultisigIsms`. Similarly, some users have expressed the desire to aggregate security across multiple generalized message bridges (GMBs). If any of those GMBs can be expressed as an ISM, an `AggregationISM` would allow users to aggregate their security models.

### **Tech Spec**

A `AggregationIsm` must return `2` in `moduleType()`.

A `AggregationIsm` must implement the following interface. Relayers should use the `ismsAndThreshold()` view function to understand what metadata the `ISM` needs in order to verify a message.

```
interface IAggregationIsm is IInterchainSecurityModule {
    /**
     * @notice Returns the set of ISMs responsible for verifying _message
     * and the number of ISMs that must verify
     * @dev Can change based on the content of _message
     * @param _message Hyperlane formatted interchain message
     * @return isms The array of ISM addresses
     * @return threshold The number of ISMs needed to verify
     */
    function ismsAndThreshold(bytes calldata _message)
        external
        view
        returns (address[] memory isms, uint8 threshold);
}
```

A `AggregationIsm` must expect the metadata passed to `verify()` to be formatted in the following way:

```
 * [   0:   1] The size of the set of ISMs capable of verifying `_message`.
 * [   1:????] Addresses of the set of ISMs capable of verifying `_message`, left padded to bytes32
 * [????:????] Metadata offsets (i.e. for each ISM, where does the metadata for this ISM start? Zero if not provided)
 * [????:????] ISM metadata, packed encoding
```

Relayers must order the ISM addresses and metadata in order to match the order returned by `ismsAndThreshold(_message)`.

Relayers must provide metadata for exactly `_threshold` ISMs.

`AggregationIsms` should avoid conditioning on frequently-changing state, as this may result in a relayer formatting metadata incorrectly.

### **Rationale**

### **Backwards Compatibility**

### **Security Considerations**
