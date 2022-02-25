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

### Single-use vs. Mutli-use Manager Contracts

Single-use manager contracts would be deployed once per Set Token by a manager factory contract, while a multi-use manager contract would be deployed once overall and could subsequently be used by all Set Tokens. Single-use manager contracts maintain separation between Set Tokens at the contract level but require individual deployments for each Set Token. A multi-use manager contract requires only one deployment but has functionality across many Set Tokens, which may open up attack vectors.

### Modular vs. Monolithic Manager Contracts

With modular manager contracts, operators call extension contracts which invoke the manager with arbitrary logic. With monolithic manager contracts, operators call the manager contract directly with logic defined on deployment. The modular manager contracts maintain more flexiblity but require extensions to be deployed and enabled. The monolithic manager contract requires migration for upgradeability but does not need extensions for functionality.

### Individual vs. Global Extensions

With individual extensions, a new extension contract must be deployed and enabled to provide some functionality to the manager. With global extensions, a collection of extension contracts would be deployed by Set Protocol and made available to managers by simply enabling them. Individual extensions offer more flexibility to managers but require individual contract deployments from managers. Global extensions offer less flexibility but do not require managers to make contract deployments.

### Recommended Solution

The recommended solution deploys single-use, modular manager contracts from a manager factory contract along with a collection of multi-use, global extensions providing basic functionality. These manager contracts will hold the `owner`, `methodologist`, and `operator` roles and an asset whitelist which will be used by extensions. The single-use manager contracts maintain the separation of Set Tokens from each other at the contract level. The modularity of the manager contracts allows for functionaity to be flexible and extensible. The collection of multi-use extensions gives managers access to basic functionality with only a state change and no contract deployment. Manager's will still have the option of deploying individual extensions for more complicated functionality.

## Timeline

|  Action               |  End Date  |
|---                    |---         |
| Technical Spec        |   2/25     |
| Implementation        |   3/2      |
| Auditors              |   3/11     |
| Launch                |   3/18     |

## Checkpoint 1

**Reviewer**: LGTM @bweick

## Proposed Architecture Changes

![Proposed Architecture Changes](../assets/stip-009/image1.png "")

**ManagerFactory**: Factory smart contract which provides asset managers (`deployer`) the ability to `create` new Set Tokens with a BaseManagerV3 manager, `migrate` existing Set Tokens to a BaseManagerV3 manager, and `initialize` modules and enable extensions.

**BaseManagerV3**: Manager smart contract which provides asset managers three permissioned roles (`owner`, `methodologist`, `operator`) and asset whitelist functionality. The `owner` grants permissions to `operator`(s) to interact with extensions. The `owner` can restrict the `operator`(s) permissions with an asset whitelist.

**BasicIssuanceExtension**: Global extension which provides users with the ability to `issue` and `redeem` Set Tokens with a smart contract manager.

**StreamingFeeSplitExtension**: Global extension which provides the `owner` and `methodologist` the ability to accrue and split streaming fees at an mutable percentage.

**TradeExtension**: Global extension which provides privileged `operator`(s) the ability to `trade` on a DEX and the `owner` the ability to restrict `operator`(s) permissions with an asset whitelist.

## Requirements

**ManagerFactory**

- Allow `deployer` to create new set tokens with a manager smart contract
- Allow `deployer` to migrate existing set tokens to a manager smart contract
- Allow `deployer` to enable extensions and initialize corresponding modules
- Allow `deployer` to create manager smart contract and initialize and parameterize all modules and extensions in two transactions

**BaseManagerV3**

- Allow `owner` to add and remove global `operator` permissions on extensions
- Allow `owner` to limit `operator`(s) functionality on extensions with an asset whitelist
- Allow `owner` to update asset whitelist
- Allow `owner` to perform Set Token admin functions such as `addModule`, `removeModule`, and `setManager`

**BasicIssuanceExtension**

- Allow `owner` to enable functionality of BasicIssuanceModule with only a state change and no contract deployment
- Allow `owner` to initialize the BasicIssuanceModule
- Allow users to `issue` and `redeem` the Set Token
- Allow Set Token to accrue `issue` and `redeem` fees

**StreamingFeeSplitExtension**

- Allow `owner` to enable functionality of StreamingFeeModule with only a state change and no contract deployment
- Allow `owner` to initialize the StreamingFeeModule
- Allow `owner` and `methodologist` to split streaming fees
- Allow `owner` to update the streaming fee
- Allow `owner` to update the streaming fee split
- Allow `owner` to update the streaming fee recipient

**TradeExtension**

- Allow `owner` to enable functionality of TradeModule with only a state change and no contract deployment
- Allow `owner` to initialize the TradeModule
- Allow privileged `operator`(s) to perform trades on a DEX
- Allow `owner` to restrict assets the privileged `operator`(s) can trade into with an asset whitelist

## User Flows

### ManagerFactory.create()

![ManagerFactory create](../assets/stip-009/image2.png "")

An `deployer` wants to create a new Set Token with a BaseManagerV3 smart contract manager.

1. The `deployer` calls create() passing in parameters to create a Set Token, parameters for the permissioning on BaseManagerV3, and the desired extensions. Specifically,

    - components: List of addresses of components for initial positions
    - units: List of units for initial positions
    - name: Name of SetToken
    - symbol: Symbol of SetToken
    - owner: The address of the `owner`
    - methodologist: The address of the `methodologist`
    - operators: List of addresses of the `operator`(s)
    - modules: List of addresses of modules to be enabled
    - assets: List of addresses of assets for initial asset whitelist
    - extensions: List of addresses of global extensions to be enabled

2. A Set Token is deployed using SetTokenCreator
3. A BaseManagerV3 is deployed with the ManagerFactory as the temporary `owner` until after initialization
4. The `deployer`, `owner`, and BaseManagerV3 are stored on the Factory in pending state

### ManagerFactory.migrate()

![ManagerFactory migrate](../assets/stip-009/image3.png "")

An `deployer` wants to migrate an existing Set Token to a BaseManagerV3 smart contract manager.

1. The `deployer` calls migrate() passing in the Set Token address, parameters for the permissioning on BaseManagerV3, and the desired extensions. Specifically,

    - owner: The address of the `owner`
    - methodologist: The address of the `methodologist`
    - operators: List of addresses of the `operator`(s)
    - assets: List of addresses of assets for initial asset whitelist
    - extensions: List of addresses of global extensions to be enabled

2. A BaseManagerV3 is deployed with the ManagerFactory as the temporary `owner` until after initialization
3. The `deployer`, `owner`, and BaseManagerV3 are stored on the Factory in pending state

### ManagerFactory.initialize()

![ManagerFactory initialize](../assets/stip-009/image4.png "")

The `deployer` wants to enable all extensions, initialize all corresponding modules, and transfer the manager `owner` role.

1. The `deployer` calls initialize() passing in the parameters for initializing modules and extensions
2. All modules are initialized via the extensions
3. The `owner` role on the BaseManagerV3 is transfered from the Factory to the input `owner`
4. The Factory deletes in `InitializeParams` for the set token, removing it from pending state

### StreamingFeeExtension.accrueFeeAndDistribute()

![StreamingFeeExtension](../assets/stip-009/image5.png "")

An interested party wants to accrue streaming fees and distribute them to the `owner` and `methodologist`.

1. The interested party calls accrueFeeAndDistribute() on the StreamingFeeExtension
2. Fees are accrued to the BaseManagerV3
3. Fees are distributed to the `owner` and `methodologist`


## Checkpoint 2

Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: []

## Specification

### ManagerFactory

#### Structs

##### InitializeParams

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|deployer|Address of the deployer|
|address|owner|Address of the owner|
|address|manager|Address of the BaseManagerV3|
|bool|isPending|Bool if manager in pending state|  

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|factory|Address of Set Token factory|
|mapping(address => InitializeParams)|initialize|Mapping from Set Token to initialization parameters|


#### Functions

| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|create|deployer|Create new Set Token with a BaseManagerV3 manager|
|migrate|deployer|Migrate existing Set Token to a BaseManagerV3 manager|
|initialize|deployer|Initialize modules and extensions|

### BaseManagerV3

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|ISetToken|setToken|Instance of SetToken|
|address|factory|Address of factory contract used to deploy contract|
|mapping(address => bool)|extensionAllowList|Mapping to check if extension is enabled|
|mapping(address => bool)|operatorAllowList|Mapping indicating if address is an approved operator|
|mapping(address => bool)|assetAllowlist|Mapping indicating if asset is approved to be traded for, wrapped into, claimed, etc.|
|address|methodologist|Address of methodologist which serves as providing methodology for the index|

#### Private Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address[]|extensions|Array of enabled extensions|
|address[]|operators|List of approved operators|
|address[]|allowedAssets|Array of enabled extensions|

#### Functions

| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|interactManager|extension|Interact with a module registered on the Set Token|
|addExtensions|owner|Add a new extension that the DelegatedManager can call|
|removeExtensions|owner|Remove an existing extension tracked by the DelegatedManager|
|addOperators|owner|Add new operator(s) address|
|addAllowedAssets|owner|Add new asset(s) that can be traded to, wrapped to, or claimed|
|removeAllowedAssets|owner|Remove asset(s) so that it/they can't be traded to, wrapped to, or claimed|
|setMethodologist|owner|Update the methodologist address|
|setManager|owner|Update the manager of the Set Token|
|addModule|owner|Add module to Set Token|
|removeModule|owner|Remove module from Set Token|

#### Modifiers
> onlyOwner
> onlyMethodologist
> onlyExtension

### BasicIssuanceExtension

#### Structs

##### IssuanceParams

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|manager|Address of the BaseManagerV3|

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|IIssuanceModule|issuanceModule|Issuance Module for Set Token|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|mapping(address=>IssuanceParams)|setIssuanceParams|Mapping from Set Token to issuance parameters|

#### Functions

| Name  | Caller  | Description     |
|------	|------	|-------------	|
|updateIssueFee|owner|Update issue fee on IssuanceModule|
|updateRedeemFee|owner|Update redeem fee on IssuanceModule|

#### Modifiers
> onlyOperator

### StreamingFeeSplitExtension

#### Structs

##### FeeParams

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|uint256|operatorFeeSplit|Percent of fees in precise units (10^16 = 1%) sent to operator, rest to methodologist|
|address|operatorFeeRecipient|Address that receives the operator's fees|
|address|manager|Address of the BaseManagerV3|

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|IStreamingFeeModule|streamingFeeModule|Streaming Fee Module for Set Token|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|mapping(address=>FeeParams)|setFeeParams|Mapping from Set Token to fee parameters|

#### Functions

| Name  | Caller  | Description     |
|------	|------	|-------------	|
|accrueFeesAndDistribute|public|Accrue fees and distribute to owner and methodologist|
|updateStreamingFee|owner|Migrate existing Set Token to a BaseManagerV3 manager|
|updateFeeRecipient|owner|Update fee recipient|
|updateFeeSplit|owner|Update fee split between operator and methodologist
|updateOperatorFeeRecipient|owner|Update the address that receives the operator's fees|

#### Modifiers
> onlyOperator

### TradeExtension

#### Structs

##### TradeParams

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|manager|Address of the BaseManagerV3|

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|ITradeModule|tradeModule|Trade Module for Set Token|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|mapping(address=>TradeParams)|setTradeParams|Mapping from Set Token to trade parameters|

#### Functions

| Name  | Caller  | Description     |
|------	|------	|-------------	|
|trade|operator|Trade between whitelisted assets on a DEX|

#### Modifiers
> onlyOperator

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