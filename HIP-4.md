| hip | title       | status | author | created    |
| --- | ----------- | ------ | ------ | ---------- |
| 4   | Routing ISM | Draft  | asaj   | 2022-12-20 |

### **Brief Summary / Abstract**

Defines the specification for an ISM that redirects to other ISMs based on message contents.

### **Motivation**

Applications may want to use different security models based on message contents or application state.

The goal of this HIP is to standardize the `RoutingIsm` interface so that users may start creating custom implementations.

### **Tech Spec**

A `RoutingIsm` must return `1` in `type()`.

A `RoutingIsm` must implement the following interface. Relayers should use this view function to understand what `ISM` will be used to verify a message.

```
interface IRoutingIsm is IInterchainSecurityModule {
    /**
     * @notice Returns the address of the ISM to route to
     * @dev Can change based on the content of _message, chain state, or both
     * @param _message Hyperlane formatted interchain message
     * @return The address of the ISM to route to
     */
    function route(
        bytes calldata _message
    ) external view returns (IInterchainSecurityModule);
}
```

A `RoutingIsm` must route messages and metadata to the `ISM` returned by `route()`.

```
function verify(bytes calldata _metadata, bytes calldata _message)
    external
    returns (bool)
{
    return route(_message).verify(_metadata, _message);
}
```

When relayers detect a `RoutingIsm`, metadata should be formatted in accordance with the `ISM` type being routed to.

`RoutingIsms` should avoid conditioning on frequently-changing state, as this may result in a relayer formatting metadata for the wrong `ISM`.

### **Rationale**

### **Backwards Compatibility**

### **Security Considerations**
