| hip | title | status | author | created |
| --- | --- | --- | --- | --- |
| 2 | Domain IDs to Chain IDs | Draft | nambrot | 2022-10-19 |

### **Brief Summary / Abstract**

This specification suggests the use of a chains Chain ID as the Domain ID and provide strategies for backwards compatibitily for apps and Hyperlane core contracts.

### **Motivation**

Current Domain IDs of AMBs are confusing as they are a new concept that is scarily similar to the existing notion of Chain IDs in single-chain EVM context, but they are virtually completely different values. The original intention behind having them be different since the set of execution environments is larger than the set of existing EVM chain IDs (i.e. non-EVM chains like cosmos chains, etc.). However EVM chains continue to capture the majority of the mindshare and this subtle difference continues to confused developers.

### **Tech Spec**

Primarily, this HIP suggests the use of an (EVM) chains Chain ID as the Domain ID. I.e Ethereum's Domain ID should be 1 instead of 
`0x657468`. This change should happen with the v2 version of the Hyperlane mailboxes. Non EVM-chain Domain IDs remain unspecified for now.


### **Rationale**

The benefit of letting developers who only think about EVM environments use the Chain ID as the Domain ID in our view exceeds the possibly confusion about the eventual addition of non-EVM chains and their Domain IDs


### **Backwards Compatibility**

Applications that migrate to Hyperlane v2 from the current version, can expect a relatively smooth transition since the `dispatch` and `handle` function signatures remain the same. One can create a middleware contract that can be enrolled on an ACM as an outbox and inbox that wraps the Mailbox contract and "translates" the old/new Domain IDs. A rough pseudo contract is shown below:


```solidity
contract CompatibilityRouter {
  mapping(uint32 => uint32) domainIdToChainId;
  mapping(uint32 => uint32) chainIdToDomainId;

  function dispatch(dest, to, body) {
    mailbox.dispatch(domainIdToChainId[dest], remoteCompatabilityRouter[dest], abi.encode(msg.sender, to, body))
  }

  function handle(origin, sender, body) onlyRemoteCompatabilityRouter {
    (actualSender, actualRecipient, actualBody) = abi.decode(body, (bytes32, bytes32, bytes));
    actualRecipient.handle(chainIdToDomainId[origin], actualSender, actualBody);
  }
}
```

### **Security Considerations**

Applications that incorrectly use the wrong Domain ID risk their messages being undeliverable.