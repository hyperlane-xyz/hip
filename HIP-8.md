# Hyperlane AVS (WIP)

### Goal

This HIP proposes a design for the Hyperlane AVS building on top of Eigenlayer protocol to utilitise the economic security for messages outbound from Ethereum and subsequently its rollups. Currently, this proposal comprises of the following functions:

- Allow eigenlayer operators to register/deregister themselves to the AVS as validators for any chain
- Allow stakers to delegate their stake to the operators for the specific strategy.
- Anticipate permissionless slashing of opt-in operators by the ISM
- Enable easy management of staked operators and maintain availability of operators for the ISMs - OOS

**Architecture**

**Interface**

```solidity
contract HyperlaneAVS is IServiceManager, OwnableUpgradeable {
    IStakeRegistry internal immutable stakeRegistry;
    IAVSDirectory internal immutable _avsDirectory;
    mapping(address => EnumerableMap.AddressToBool (IChallenger => bool)) internal optInChallengers;


    modifier onlyStakeRegistry() {
        require(msg.sender == address(stakeRegistry);
        _;
    }

    function registerOperatorToAVS(
        address operator,
        ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature,
        IChallenger[] memory challengers
    ) public virtual onlyStakeRegistry {
        _avsDirectory.registerOperatorToAVS(operator, operatorSignature);
        for (uint256 i = 0; i < challengers.length; i++) {
            optInChallengers[operator][challengers[i]] = true;
        }
    }

    function deregisterOperatorFromAVS(address operator) public virtual onlyStakeRegistry {
        for (uint256 i = 0; i < optInChallengers[operator].length; i++) {
            optInChallengers[operator][optInChallengers[operator].at(i)] = false;
        }
        _avsDirectory.deregisterOperatorFromAVS(operator);
    }

    function optIntoChallenger(IChallenger challenger) public virtual {
        optInChallengers[msg.sender][challenger] = true;
    }

    function optOutOfChallenger(IChallenger challenger) public virtual {
        optInChallengers[msg.sender][challenger] = false;
    }

}
```

-> the stakeRegistry has a single thresholdWeight, totalWeight for the AVS which is used for \_validateThresholdState
-> threshold weight is 66% of the total?
-> updating operator weights?

Workflow for deploying Hyperlane and configuring validating on rollupA, rollupB

- deploy ecdsaRegistry

- deploy and config IStrategy with native ETH and configure them to the StrategyManager through addStrategiesToDepositWhitelist

- enable quorum for rollupA, rollupB and set the quorum param for each quorum like maxOperatorCount, minStake, etc and strategy

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
- if the operator double signs, the slasher will slash the operator

Deregistering

Workflow for ISMs

- selects restaked validator in your ISM.
-

### Workflow for slashing

**Interface**

```solidity
interface IChallenger {
    function slashOperator(address operator, bytes32 signedRoot, bytes32 correctRoot) external;
}

```

- Let's assume an operator signs up for a specific IChallenger[] list which includes (say rollupA remote challenger).
- A watcher will call the INativeChallenger from the destination chain where the double signing was detected. It checks the signed root against the correct root and if it doesn't match the correct root, it will calls the remoteChallenger on the L1 via the native bridge.
- The remoteChallenger calls the stakingManager and then serviceManager which calls the slasher.
- The serviceManager checks if the operator is registered for the challenger and if it is, it calls the slasher to slash the operator.

Q From EL (4/10), some of the items for launching mainnet:

1. let us know your hardware requirements for operators, we'll make intros to the Eigen operators to test your software so you can get started this or next week, you need at least 20 Eigen node operators to launch on mainnet and 10 of them overlapping with other AVSs
2. get whitelisted on the testnet frontend
3. Code frozen
4. get avs registry contracts audited
5. Operator ready: tested with Eigen operators, good operator documentation, sufficient operator support ready
6. Lastly, get whitelisted on our mainnet frontend

Scope for first milestone

- Contracts

  - UML diagrams for contract interactions - 1 day
  - HyperlaneAVS - 2 day
  - General staking interface IStakingManager and loose interactsions for native staking, restaking for Karak, etc - 2 day
  - tests for different configuration of different strategies, slasher configuration, etc (using the EL harness) - 3 day
  - StakeWeightMultisigISM (optional) - NA

- Agent work

  - Full node spec compliance - [link](https://docs.eigenlayer.xyz/category/node-specification) - hardcoded right now - NA
  - relayer changes for StakeWeightMultisigISM - NA

- Deployments
  - Holesky deployment of Hyperlane core - 0.5 day
  - Deploy the AVS contracts and (configure the AW validators) - 2 days
  - e2e test in CI - OOS
- Guide for operators to (de)register with Hyperlane AVS, node classes/requirements, etc + giving EL the - 1.5 day

**Open questions**

- recording stake globally?
- churn for operators?
- minimum stake requirment for operators?

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
