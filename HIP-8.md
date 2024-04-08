Eigenlayer

Contracts

A quorum is a grouping and configuration of specific kinds of stake that an AVS considers when interacting with Operators. When Operators register for an AVS, they select one or more quorums within the AVS to register for.

Quorums define a maximum Operator count as well as parameters that determine when a new Operator can replace an existing Operator when this max count is reached. These definitions are contained in a quorum's OperatorSetParam, which the Owner can configure via the RegistryCoordinator. A quorum's OperatorSetParam defines both a max Operator count, as well as stake thresholds that the incoming and existing Operators need to meet to qualify for churn.

DelegationManager

"Whereas the EigenPodManager and StrategyManager perform accounting for individual Stakers according to their native ETH or LST holdings respectively, the DelegationManager sits between these two contracts and tracks these accounting changes according to the Operators each Staker has delegated to.

This means that each time a Staker's balance changes in either the EigenPodManager or StrategyManager, the DelegationManager is called to record this update to the Staker's delegated Operator (if they have one). For example, if a Staker is delegated to an Operator and deposits into a strategy, the StrategyManager will call the DelegationManager to update the Operator's delegated shares for that strategy"

How does this impact the fee rewards for the stakers, earn per message and then claim at once?

and minWithdrawalDelayBlocks = 50400 (1 week) for operators for EL M2

ISM verification cost

Approximate ECDSA verification cost is 219k for 10/18, 273420k for 18/18. But, we give the ISM the ability to set the threshold for their own quorum so this shouldn't be a problem. Note: revisit.

HyperlaneAVS tells the AVS directory of the up-to-date status of each operator in the specific AVS.

```solidity
contract HyperlaneAVS is IServiceManager, OwnableUpgradeable {
    IStakeRegistry internal immutable stakeRegistry;
    IAVSDirectory internal immutable _avsDirectory;

    modifier onlyStakeRegistry() {
        require(msg.sender == address(stakeRegistry);
        _;
    }

    // or mapping internal immutable(uint32 => IStakeRegistry) stakeRegistries;

    function registerOperatorToAVS(
        address operator,
        ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature
    ) public virtual onlyStakeRegistry {
        _avsDirectory.registerOperatorToAVS(operator, operatorSignature);
    }

    function deregisterOperatorFromAVS(address operator) public virtual onlyStakeRegistry {
        _avsDirectory.deregisterOperatorFromAVS(operator);
    }

}
```

-> the stakeRegistry has a single thresholdWeight, totalWeight for the AVS which is used for \_validateThresholdState
-> threshold weight is 66% of the total?
-> updating operator weights?

Workflow for deploying Hyperlane and configuring validating on rollupA, rollupB

- deploy ecdsaRegistry

- deploy and config IStrategy with native ETH and configure them to the StrategyManager through addStrategiesToDepositWhitelist

- enable bitmapped quorum for rollupA, rollupB and set the quorum param for each quorum like maxOperatorCount, minStake, etc and strategy

```
hypAVS = HyperlaneAVS.deploy(owner);
ecdsaRegistry = ECDSARegistry.deploy();

for (domains) {
    ETHstrategy = IStrategy.deploy(manager);
    ETHstrategy.initialize(WETH);
    quorums.push({strategy, 1, [ETHstrategy]});
    ecdsaRegistries.initialize(hypAVS, 6666, quorums);
}

```

Workflow for operators as a hyperlane validator

Registering

- register operator with Eigenlayer via
  ELDelegationManager.registerAsOperator(
  OperatorDetails { feeReceiver = self, address delegationApprover=self, stakerOptOutWindowBlocks = 0},
  metadataURI = "example operator"
  )

- register operator with Hyperlane AVS via

  registryController.register

Operation

- does the work as a normal validator
- if the operator double signer, the slasher will slash the operator

Deregistering

Workflow for ISMs

- selects restaked validator in your ISM.
-

Scope

- Holesky deployment of Hyperlane core
