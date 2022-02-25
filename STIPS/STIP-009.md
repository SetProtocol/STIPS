# STIP-009: Self Service Manager Contracts
*Using template v0.1*
## Abstract
In order to offer self-service deployment and management of Sets we need to create a standard set of manager contracts that is able to effectively delegate responsibilities to certain addresses and limit the assets that delegated addresses can add to positions.
## Motivation
This feature will allow us to provide a standardized experience to SetToken owners that want to use the TokenSets UI to manage their Set. Currently, TokenSets supports only EOA managed Sets while the Sets with the largest TVL are generally managed by smart contracts (in order to restrict the manager's ability to rug users), here we seek to move everyone onto manager contracts. The manager contracts will facilitate better delegation abilities via the addition of operators (not to be confused with the operator role in previous manager contracts which will now be known as owners) which can execute trades, wraps, claims, etc. Additionally, the owner role will also be able to enforce asset whitelists on their delegated operators in order to make sure they are not able to trade into an asset they are not supposed to.

Additional functionality will be built to facilitate the deployment of new Sets/manager contracts as well as the migration of old Sets to the new manager system. The goal is to minimize the amount of transactions necessary to deploy the Set/Manager as well initialize the Set's modules.

All of this should add up to a better experience for the owner to create and manage their own Set while simplifying and standardizing flows on TokenSets. 

## Background Information
At the moment there are two ways to manage Sets,(1) directly via an EOA or multi-sig, or (2) via manager contract(s). EOAs/multi-sigs provide the greatest amount of flexibility because you are interacting directly with the system however they do not allow for advanced permissioning or further restrictions on the rebalancing of the Set, something a buyer of a Set may want in order to guarantee they will not be rugged by the manager. Currently there are no "standard" manager contracts built to interact with Sets and supported by the TokenSets UI, however there are a mixture of previous smart contracts with various different features:

[ICManager](https://github.com/IndexCoop/index-coop-smart-contracts/blob/master/contracts/manager/ICManager.sol) - This was the monolithic, first iteration of a manager contract that contains `operator` and `methodologist` roles. It is not extensible via other contracts but did contain a function that allowed the `operator` to pass arbitrary bytedata to a target contract address (to call modules). This contract wasn't desirable due to it being unable to be conveniently upgraded as well as its poor UX for managers who would look to add new functionality (having to submit arbitrarty bytestrings instead of having clear interfaces).

[BaseManager](https://github.com/IndexCoop/index-coop-smart-contracts/blob/master/contracts/manager/BaseManager.sol) - This next generation manager contract system was a more modular approach where extensions could be added to a BaseManager contract to add new functionality. This contract maintained the idea of an `operator` and `methodologist`, giving the `operator` the ability to add new functionality. Extensions became the only addresses permissioned to call an `interactManager` function on the manager which forwarded arbitrary bytedata to a target address (generally a module). This gave `operator`s much of the same powers as in the `ICManager` but the ability to add clean interfaces to interact with as well as be able to encode strategies to govern SetTokens within one of their extensions to allow for more decentralized execution of rebalances.

[BaseManagerV2](https://github.com/IndexCoop/index-coop-smart-contracts/blob/master/contracts/manager/BaseManagerV2.sol) - This manager contract is very similar to `BaseManager` it just gave `methodologists` greater ability to counteract `operators` ablility to add and remove privileged functionalities from the manager in case of an adversarial relationship between the `operator` and `methodologist`.

## Open Questions
Pose any open questions you may still have about potential solutions here. We want to be sure that they have been resolved before moving ahead with talk about the implementation. This section should be living and breathing through out this process.
- [ ] When accruing streaming fees where do we send the fees to? The manager?
    - *Answer*
- [ ] How do we validate that trades for *x* asset are actually for *x* asset when the trade is encoded in a bunch of bytedata?
    - *Answer*
- [ ] What data needs to be held on every extension? What data needs to be passed in for initialization on each extension?
    - *Initialization and Data Needs:*
        - *SetToken address*
        - *Manager contract address*
- [ ] Do we need an initialization flow similar to SetTokens/modules?
    - *Answer*
- [ ] During migration when does the EOA manager update the manager address to the deployed manager?
    - *Current thoughts are that extension initialization and module initialization happen in same step. However the manager would need to be the manager contract at that point for it to not fail. Given that the initialization step is being called by the factory we would need to update the manager on the SetToken before initialization*

## Feasibility Analysis
**Single-use vs. Mutli-use Manager Contracts**

Single-use manager contracts would be deployed once per Set Token by a manager factory contract, while a multi-use manager contract would be deployed once overall and could subsequently be used by all Set Tokens. Single-use manager contracts maintain separation between Set Tokens at the contract level but require individual deployments for each Set Token. A multi-use manager contract requires only one deployment but has functionality across many Set Tokens, which may open up attack vectors.

**Modular vs. Monolithic Manager Contracts**

With modular manager contracts, operators call extension contracts which invoke the manager with arbitrary logic. With monolithic manager contracts, operators call the manager contract directly with logic defined on deployment. The modular manager contracts maintain more flexiblity but require extensions to be deployed and enabled. The monolithic manager contract requires migration for upgradeability but does not need extensions for functionality.

**Individual vs. Global Extensions**

With individual extensions, a new extension contract must be deployed and enabled to provide some functionality to the manager. With global extensions, a collection of extension contracts would be deployed by Set Protocol and made available to managers by simply enabling them. Individual extensions offer more flexibility to managers but require individual contract deployments from managers. Global extensions offer less flexibility but do not require managers to make contract deployments.

**Recommended Solution**

The recommended solution deploys single-use, modular manager contracts from a manager factory contract along with a collection of multi-use, global extensions providing basic functionality. These manager contracts will hold the `owner`, `methodologist`, and `operator` roles and an asset whitelist which will be used by extensions. The single-use manager contracts maintain the separation of Set Tokens from each other at the contract level. The modularity of the manager contracts allows for functionaity to be flexible and extensible. The collection of multi-use extensions gives managers access to basic functionality with only a state change and no contract deployment. Manager's will still have the option of deploying individual extensions for more complicated functionality.
## Timeline
|  Action               |  End Date  |
|---                    |---         |
| Technical Spec        |   2/25     |
| Implementation        |   3/2      |
| Auditors              |   3/11     |
| Launch                |   3/18     |
## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**: LGTM @bweick

## Proposed Architecture Changes
![](../assets/stip-009/image1.png "")

**ManagerFactory**: Factory smart contract which enables the creation of new Set Tokens with a BaseManagerV3 manager, the migration of existing Set Tokens to a BaseManagerV3 manager, and the initialization of modules and extensions.

**BaseManagerV3**: Manager smart contract which supports three roles (`owner`, `methodologist`, `operator`) and an asset whitelist which can be set by the `owner` to restrict the `operator`'s interactions with extensions. 

**BasicIssuanceExtension**: Global extension which provides manager smart contracts with an interface to the BasicIssuanceModule (`issue`, `redeem`).

**StreamingFeeExtension**: Global extension which provides manager smart contracts with an interface to the StreamingFeeModule (`accrueFeesAndDistribute`, `updateStreamingFee`, `updateFeeRecipient`, `updateFeeSplit`).

**TradeExtension**: Global extension which provides manager smart contracts with an interface to the TradeModule (`trade`).

## Requirements

**ManagerFactory**

- Allow owners to create new set tokens with a manager smart contract
- Allow owners to migrate existing set tokens to a manager smart contract
- Allow owners to create manager smart contract and initialize and parameterize all modules and extensions in two transactions

**BaseManagerV3**

- Allow owners to permission specific functionality (extensions) to different operators
- Allow owners to update permissions on specific functionality (extensions) for different operators
- Allow owners to limit operator functionality using extensions with an asset whitelist
- Allow owners to update asset whitelist

**BasicIssuanceExtension**

- Allow owners to enable functionality of BasicIssuanceModule with only a state change and no contract deployment

**StreamingFeeExtension**

- Allow owners to enable functionality of StreamingFeeModule with only a state change and no contract deployment

**TradeExtension**

- Allow owners to enable functionality of TradeModule with only a state change and no contract deployment

## User Flows

### ManagerFactory.create()

An owner wants to create a new Set Token with a BaseManagerV3 smart contract manager.

1. The `owner` calls create() passing in parameters to create a Set Token, parameters for the permissioning on BaseManagerV3, and the desired extensions. Specifically,

    - components: List of addresses of components for initial positions
    - units: List of units for initial positions
    - name: Name of SetToken
    - symbol: Symbol of SetToken
    - owner: The address of the `owner`
    - methodologist: The address of the `methodologist`
    - operators: List of addresses of the `operator`s
    - assets: List of addresses of assets for initial asset whitelist
    - extensions: List of addresses of global extensions to be enabled

2. A Set Token is deployed using SetTokenCreator
3. A BaseManagerV3 is deployed
4. Initialization parameters for the Set Token are stored on the Factory
5. The Set Token and Manager are put in pending state

### ManagerFactory.migrate()

An owner wants to migrate an existing Set Token to a BaseManagerV3 smart contract manager.

1. The `owner` calls migrate() passing in the Set Token address, parameters for the permissioning on BaseManagerV3, and the desired extensions. Specifically,

    - owner: The address of the `owner`
    - methodologist: The address of the `methodologist`
    - operators: List of addresses of the `operator`s
    - assets: List of addresses of assets for initial asset whitelist
    - extensions: List of addresses of global extensions to be enabled

2. A BaseManagerV3 is deployed
3. Initialization parameters for the Set Token are stored on the Factory
4. The Set Token and Manager are put in pending state

### ManagerFactory.initialize()

An owner wants to enable all extensions and initialize all corresponding modules.

1. The `owner` calls initialize() passing in the parameters for initializing modules and extensions
2. All modules are initialized
3. All extensions are enabled
4. The Set Token and Manager are put in initialized state

## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: []
## Specification
### [Contract Name]
#### Inheritance
- List inherited contracts
#### Structs
| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|manager|Address of the manager|
|uint256|iterations|Number of times manager has called contract|  
#### Constants
| Type 	| Name 	| Description 	| Value 	|
|------	|------	|-------------	|-------	|
|uint256|ONE    | The number one| 1       	|
#### Public Variables
| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|uint256|hodlers|Number of holders of this token|
#### Functions
| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|startRebalance|Manager|Set rebalance parameters|
|rebalance|Trader|Rebalance SetToken|
|ripcord|EOA|Recenter leverage ratio|
#### Modifiers
> onlyManager(SetToken _setToken)
#### Functions
> issue(SetToken _setToken, uint256 quantity) external
- Pseudo code
## Checkpoint 3
Before we move onto the implementation phase we want to make sure that we are aligned on the spec. All contracts should be specced out, their state and external function signatures should be defined. For more complex contracts, internal function definition is preferred in order to align on proper abstractions. Reviewer should take care to make sure that all stake holders (product, app engineering) have their needs met in this stage.

**Reviewer**:

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()  
[Link to Deploy outputs PR]()
