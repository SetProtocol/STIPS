# STIP-009: Self Service Manager Contracts

*Using template v0.1*

## Abstract

Historically, SetProtocol has manually deployed SetToken specific manager contracts and extensions for IndexCoop issued funds. These contracts encode fee management and rebalance trading logic and provide better security guarantees to SetToken holders than direct management via EOA.

This STIP proposes that we automate manager contract deployments using on-chain factories and support "self-service" manager enabled SetToken creation as a feature available to anyone in the SetProtocol UI (tokensets.com).

This involves:
+ creating a standard set of manager contracts that are:
    + able to delegate responsibilities to certain participants (owners and operators)
    + define the assets that participants are able to trade

+ creating a SetToken and Management Contract factory that deploys and wires all the contract components up.

## Motivation

This feature will allow us to:
+ provide a standardized experience to SetToken owners that want to use the TokenSets UI to manage their Set
+ make previously bespoke SetToken management systems available to any TokenSets user
+ improve the security tokensets-issued assets for Set holders by binding managers to smart-contract defined logics
+ introduce clearly defined roles for SetToken stakeholders and add the ability for SetToken managers to restrict Sets to a white-listed group of assets

## Background Information

At the moment there are two ways to manage Sets:
1. directly via an EOA or multi-sig,
2. via manager contract(s).

EOAs/multi-sigs provide the greatest amount of flexibility because they interact directly with the system. However they do not allow for advanced permissioning or restrictions on how the Set can be rebalanced.

Currently there are no "standard" manager contracts supported by the TokenSets UI for general use. However, several manager contracts we can draw design ideas from have been developed to support select client funds:

[ICManager](https://github.com/IndexCoop/index-coop-smart-contracts/blob/master/contracts/manager/ICManager.sol) - This was the monolithic, first iteration of a manager contract that contains `operator` and `methodologist` roles. It is not extensible via other contracts but did contain a function that allowed the `operator` to pass arbitrary bytedata to a target contract address (to call modules). This contract wasn't desirable due to it being unable to be conveniently upgraded as well as its poor UX for managers who were required to submit arbitrary bytestrings instead of having clear interfaces.

[BaseManager](https://github.com/IndexCoop/index-coop-smart-contracts/blob/master/contracts/manager/BaseManager.sol) - This next generation manager contract system implements a more modular approach where extensions can be added to a BaseManager contract to add new functionality. This contract maintained the idea of an `operator` and `methodologist`, giving the `operator` the ability to add new functionality. Extensions became the only addresses permissioned to call an `interactManager` function on the manager which forwarded arbitrary bytedata to a target address (generally a module). This gave `operator`s much of the same powers as in the `ICManager` but the ability to add clean interfaces to interact with as well as be able to encode strategies to govern SetTokens within one of their extensions to allow for more decentralized execution of rebalances.

[BaseManagerV2](https://github.com/IndexCoop/index-coop-smart-contracts/blob/master/contracts/manager/BaseManagerV2.sol) - This manager contract is very similar to `BaseManager` it just gave `methodologists` greater ability to counteract `operators` ablility to add and remove privileged functionalities from the manager in case of an adversarial relationship between the `operator` and `methodologist`.

## Open Questions

Pose any open questions you may still have about potential solutions here. We want to be sure that they have been resolved before moving ahead with talk about the implementation. This section should be living and breathing through out this process.

- [ ] When accruing streaming fees where do we send the fees to? The manager?
    - The manager. We don't want to pool fees in generalized extensions.
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

### Single-user vs. Mutli-user Manager Contracts

Single-user manager contracts would be deployed once per Set Token by a manager factory contract, while a multi-user manager contract would be deployed once overall and could subsequently be used by all Set Tokens. Single-user manager contracts maintain separation between Set Tokens at the contract level but require individual deployments for each Set Token. A multi-user manager contract requires only one deployment but has functionality across many Set Tokens, which may open up attack vectors.

### Modular vs. Monolithic Manager Contracts

With modular manager contracts, operators call extension contracts which invoke the manager with arbitrary logic. With monolithic manager contracts, operators call the manager contract directly with logic defined on deployment. The modular manager contracts maintain more flexiblity but require extensions to be deployed and enabled. The monolithic manager contract requires migration for upgradeability but does not need extensions for functionality.

### Individual vs. Global Extensions

With individual extensions, a new extension contract must be deployed and enabled to provide some functionality to the manager. With global extensions, a collection of extension contracts would be deployed by Set Protocol and made available to managers by simply enabling them. Individual extensions offer more flexibility to managers but require individual contract deployments from managers. Global extensions offer less flexibility but do not require managers to make contract deployments.

### Recommended Solution

We recommend deploying single-user, modular manager contracts from a manager factory contract along with a collection of multi-user, "global" extensions providing basic functionality. Manager contracts will define `owner`, `methodologist`, and `operator` roles and an asset whitelist which will be used by extensions.

**Design features**

+ single-user manager contracts mirror the separation of Set Tokens from each other at the contract level.
+ modular manager contracts allow for functionaity to be flexible and extensible.
+ a collection of multi-use extensions gives managers access to basic functionality without requiring a dedicated contract deployment.
+ managers will retain the option of deploying individual extensions for more complicated functionality.

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

**ManagerFactory**: Factory smart contract which provides asset managers (`deployer`) the ability to `create` new Set Tokens with a DelegatedManager manager, `migrate` existing Set Tokens to a DelegatedManager manager, and `initialize` modules and enable extensions.

**DelegatedManager**: Manager smart contract which provides asset managers three permissioned roles (`owner`, `methodologist`, `operator`) and asset whitelist functionality. The `owner` grants permissions to `operator`(s) to interact with extensions. The `owner` can restrict the `operator`(s) permissions with an asset whitelist.

**BasicIssuanceExtension**: Global extension which provides users with the ability to `issue` and `redeem` Set Tokens with a smart contract manager.

**StreamingFeeSplitExtension**: Global extension which provides the `owner` and `methodologist` the ability to accrue and split streaming fees at an mutable percentage.

**TradeExtension**: Global extension which provides privileged `operator`(s) the ability to `trade` on a DEX and the `owner` the ability to restrict `operator`(s) permissions with an asset whitelist.

## Requirements

### ManagerFactory

- Allow `deployer` to create new set tokens with a manager smart contract
- Allow `deployer` to migrate existing set tokens to a manager smart contract
- Allow `deployer` to enable extensions and initialize corresponding modules
- Allow `deployer` to create manager smart contract and initialize and parameterize all modules and extensions in two transactions

### DelegatedManager

- Allow `owner` to add and remove global `operator` permissions on extensions
- Allow `owner` to limit `operator`(s) functionality on extensions with an asset whitelist
- Allow `owner` to update asset whitelist
- Allow `owner` to perform Set Token admin functions such as `addModule`, `removeModule`, and `setManager`
- Allow extensions to interact with modules

### BasicIssuanceExtension

- Allow `owner` to enable functionality of BasicIssuanceModule with only a state change and no contract deployment
- Allow `owner` to initialize the BasicIssuanceModule
- Allow users to `issue` and `redeem` the Set Token
- Allow Set Token to accrue `issue` and `redeem` fees

### StreamingFeeSplitExtension

- Allow `owner` to enable functionality of StreamingFeeModule with only a state change and no contract deployment
- Allow `owner` to initialize the StreamingFeeModule
- Allow `owner` and `methodologist` to split streaming fees
- Allow `owner` to update the streaming fee
- Allow `owner` to update the streaming fee split
- Allow `owner` to update the streaming fee recipient

### TradeExtension

- Allow `owner` to enable functionality of TradeModule with only a state change and no contract deployment
- Allow `owner` to initialize the TradeModule
- Allow privileged `operator`(s) to perform trades on a DEX
- Allow `owner` to restrict assets the privileged `operator`(s) can trade into with an asset whitelist

## User Flows

### ManagerFactory.createSetAndManager()

![ManagerFactory create](../assets/stip-009/image2.png "")

An `deployer` wants to create a new Set Token with a DelegatedManager smart contract manager.

1. The `deployer` calls createSetAndManager() passing in parameters to create a Set Token, parameters for the permissioning on DelegatedManager, and the desired extensions. Specifically,

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

2. Creation Parameters are validated:
    - If assets are defined, asset list must match components

3. A Set Token is deployed using SetTokenCreator

4. A DelegatedManager is deployed with the ManagerFactory as the temporary `owner` until after initialization

5. The `deployer`, `owner`, and DelegatedManager are stored on the Factory in pending state

### ManagerFactory.createManager()

![ManagerFactory migrate](../assets/stip-009/image3.png "")

An `deployer` wants to migrate an existing Set Token to a DelegatedManager smart contract manager.

1. The `deployer` calls createManager() passing in the Set Token address, parameters for the permissioning on DelegatedManager, and the desired extensions. Specifically,

    - owner: The address of the `owner`
    - methodologist: The address of the `methodologist`
    - operators: List of addresses of the `operator`(s)
    - assets: List of addresses of assets for initial asset whitelist
    - extensions: List of addresses of global extensions to be enabled

2. createManager parameters are validated:
    - If assets are defined, asset list must match SetToken's existing components

3. A DelegatedManager is deployed with the ManagerFactory as the temporary `owner` until after initialization

4. The `deployer`, `owner`, and DelegatedManager are stored on the Factory in pending state

### ManagerFactory.initialize()

![ManagerFactory initialize](../assets/stip-009/image4.png "")

The `deployer` wants to enable all extensions, initialize all corresponding modules, and transfer the manager `owner` role.

1. The `deployer` calls initialize() passing in the parameters for initializing modules and extensions

2. Initialization parameters are validated:
    - `initializationState` must be `pending`
    - `initialize` caller must be the `deployer`
    - `initializeTargets` (extension addresses) must be the same length as `initializeBytecode` (initialization instructions)

3. All extensions are initialized for the SetToken (using bytecode blobs)
    - modules are initialized for the SetToken via the extensions

4. The `owner` role on the DelegatedManager is transfered from the Factory to the `owner` designated during the creation step.

5. The Factory deletes in `InitializeParams` for the set token, removing it from pending state

(5.a) In a separate step, the SetToken's current manager address must be reset to point at the newly deployed DelegatedManager contract.
    - If SetToken manager is EOA, call setToken.setManager(_newAddress)
    - If SetToken manager is contract, call CurrentManagerContract.setManager(_newAddress)

### StreamingFeeExtension.accrueFeeAndDistribute()

![StreamingFeeExtension](../assets/stip-009/image5.png "")

An interested party wants to accrue streaming fees and distribute them to the `owner` and `methodologist`.

1. The interested party calls accrueFeeAndDistribute() on the StreamingFeeExtension
2. Fees are accrued to the DelegatedManager
3. Fees are distributed to the `owner` and `methodologist`


## Checkpoint 2

**Reviewer**:

Reviewer: []

## Specification

### ManagerFactory

#### Events

##### DelegatedManagerDeployed

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|setToken|Address of SetToken to be managed|
|address|deployer|Address of the deployer|
|address|owner|Address of the owner|
|address|manager|Address of the DelegatedManager|


##### DelegatedManagerInitialized

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|setToken|Address of SetToken to be managed|
|address|deployer|Address of the deployer|
|address|owner|Address of the owner|
|address|manager|Address of the DelegatedManager|


#### Structs

##### InitializeParams

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|deployer|Address of the deployer|
|address|owner|Address of the owner|
|address|manager|Address of the DelegatedManager|
|bool|isPending|Bool if manager in pending state|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|factory|Address of Set Token factory|
|mapping(address => InitializeParams)|initialize|Mapping from Set Token to initialization parameters|


#### Functions

| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|createSetAndManager |deployer|Create new Set Token with a DelegatedManager manager|
|createManager | SetToken owner |Migrate existing Set Token to a DelegatedManager manager|
|initialize |deployer or SetToken owner |Initialize modules and extensions|

> createSetAndManager

```solidity
function createSetAndManager(
    address[] memory _components,
    uint256 _units,
    string memory _name,
    string memory _symbol,
    address _owner,
    address _methodologist,
    address[] memory _modules,
    address[] memory _operators,
    address[] memory _assets,
    address[] memory _extensions
)
    external
{
    if (_assets.length != 0) {
        _validateComponentsMatchWhitelistedAssets(_components, assets);
    }

    address setTokenAddress = _deploySet(
        _components,
        _modules,
        _units,
        _name,
        _symbol
    );

    address managerAddress = _deployManager(
        setTokenAddress,
        _methodologist,
        _operators,
        _assets,
        _extensions
    );

    initialize[setTokenAddress] = InitializeParams({
        deployer: msg.sender,
        owner: _owner,
        manager: managerAddress,
        isPending: true
    });
}
```

> createManager
```solidity
function createManager(
    address memory _setTokenAddress,
    address _owner,
    address _methodologist,
    address[] memory _modules,
    address[] memory _operators,
    address[] memory _assets,
    address[] memory _extensions
)
    external
{
    _validateComponentsMatchWhitelistedAssets(setToken.getComponents(), assets);

    address managerAddress = _deployManager(
        setTokenAddress,
        _methodologist,
        _operators,
        _assets,
        _extensions
    );

    initialize[setTokenAddress] = InitializeParams({
        deployer: msg.sender,
        owner: _owner,
        manager: managerAddress,
        isPending: true
    });
}
```

> initialize

```solidity
function initialize(
    address memory _setTokenAddress,
    address[] memory _initializeTargets,
    bytes[] memory _initializeBytecode,
)
    external
{
    require(msg.sender == initialize[_setTokenAddress].deployer);
    require(initialize[_setTokenAddress].isPending);
    require(_initializeTargets.validatePairsWithArray(_initializeBytecode));

    for (uint256 i = 0; i < _extensions.length; i++) {
        _initializeTargets[i].initialize(_initializeBytecode[i]]);
    }

    initialize[_setTokenAddress].manager.setManager(initialize[_setTokenAddress].owner)

    delete initialize[_setTokenAddress];
}
```

### DelegatedManager

#### Enums

##### ExtensionState

| Value  | Name  | Description   |
|------ |------ |-------------  |
| 0 | NONE | State when extension has not been added |
| 1 | PENDING | State when extension has been added but not yet initialized |
| 2 | INITIALIZED | State when extension has been initialized |

#### Events

##### MethodologistChanged

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|oldMethodologist| previous methodologist|
|address|oldMethodologist| new methodologist|

#### ExtensionAdded

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|extension| added extension address |

#### ExtensionRemoved

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|extension| removed extension address |


#### OperatorAdded

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|operator| added operator address |


#### OperatorRemoved

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|operator| removed operator address |

#### AllowedAssetAdded

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|asset| added allowed asset |

#### AllowedAssetRemoved

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|asset| added allowed asset |


#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|ISetToken|setToken|Instance of SetToken|
|address|factory|Address of factory contract used to deploy contract|
|address|methodologist|Address of methodologist which serves as providing methodology for the index|
|boolean|anyAssetAllowed|when true, assetAllowList restrictions are ignored |
|mapping(address => ExtensionState)|extensionAllowList|Mapping to check if extension is enabled|
|mapping(address => bool)|operatorAllowList|Mapping indicating if address is an approved operator|
|mapping(address => bool)|assetAllowlist|Mapping indicating if asset is approved to be traded for, wrapped into, claimed, etc.|

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
|initializeExtension|extension|Initializes an added extension from PENDING to INITIALIZED state|
|addExtensions|owner|Add a new extension that the DelegatedManager can call|
|removeExtensions|owner|Remove an existing extension tracked by the DelegatedManager|
|addOperators|owner|Add new operator(s) address|
|addAllowedAssets|owner|Add new asset(s) that can be traded to, wrapped to, or claimed|
|removeAllowedAssets|owner|Remove asset(s) so that it/they can't be traded to, wrapped to, or claimed|
|setAnyAssetAllowed|owner|set anyAssetAllowed variable to true or false|
|setMethodologist|owner|Update the methodologist address|
|setManager|owner|Update the manager of the Set Token|
|addModule|owner|Add module to Set Token|
|removeModule|owner|Remove module from Set Token|

#### Modifiers

| Name  | Description   |
|------ |-------------  |
|onlyOwner| Requires that DelegatedManager `owner` is caller |
|onlyExtension | Requires that msg.sender is an initialized extension in the `extensionAllowList` array |

### Functions

> constructor

```solidity
constructor(
    ISetToken _setToken,
    address _factory,
    address _methodologist,
    address[] memory _extensions,
    address[] memory _operators,
    address[] memory _allowedAssets
)
```

TODO: more details here....
+ Set *setToken*, *factory* and *methodologist* public variables
+ Add allowed *_extensions* (these will be in set to *PENDING* state)
+ Add approved *_operators*
+ If *_allowedAssets* array is empty, set `anyAssetAllowed` to otherwise add allowedAssets to....

> interactManager

ONLY EXTENSION: Interact with a module registered on the SetToken

```
function interactManager(address _module, bytes calldata _data) external onlyExtension
```

+ Require that *_module* is not the SetToken (to prevent operator from bypassing the extension interface)
+ Call `_module.functionCallWithValue(_data, 0)`

> initializeExtension

Initializes an added extension from PENDING to INITIALIZED state. An address can only
enter a PENDING state if it is an enabled extension added by the manager. Only callable
by the extension itself, hence msg.sender is the subject of update.

```solidity
function initializeExtension()
```

+ Require that extension exists and it's state is *PENDING*
+ Set *extensionAllowlist[msg.sender]* to *INITIALIZED*
+ Add *msg.sender* to *extensions* array

> addExtension

ONLY OWNER: Add a new extension that the DelegatedManager can call

```solidity
function addExtensions(address[] memory _extensions)
```

+ for each extension in _extensions
  + require that extension state in *extensionAllowList* is *NONE* (has not already been added)
  + set *extensionAllowlist[extension]* to PENDING
  + emit *ExtensionAdded* event


> removeExtensions

ONLY OWNER: Remove an existing extension tracked by the DelegatedManager.

```solidity
function removeExtensions(address[] memory _extensions)
```

+ for each extension in _extensions
  + require that extension state in *extensionAllowList* is *INITIALZED*
  + delete extension from *extensions* array
  + set extension state to *NONE* in *extensionAllowList*
  + call extensions's own *removeExtension* method for manager's setToken
  + emit *ExtensionRemoved* event

>

### BaseGlobalExtension

#### Modifiers

> onlyAssetAllowList

```solidity
modifier onlyAssetAllowList(address memory _receiveAsset) {
    require(manager.assetAllowList[_receiveAsset], "Must be allowed asset");
    _;
}
```

> onlyOwner

```solidity
modifier onlyOwner(ISetToken _setToken) {
    require(msg.sender == _manager(_setToken).owner(), "Must be owner");
    _;
}
```

> onlyMethodologist

```solidity
modifier onlyMethodologist(ISetToken _setToken) {
    require(msg.sender == _manager(_setToken).methodologist(), "Must be methodologist");
    _;
}
```

> onlyOperator

```solidity
modifier onlyOperator(ISetToken _setToken) {
    require(_manager(_setToken).operatorAllowlist(msg.sender), "Must be approved operator");
    _;
}
```

> onlyManager

```solidity
modifier onlyManager(ISetToken _setToken) {
    require(address(_manager(_setToken)) == msg.sender, "Manager must be sender");
    _;
}
```

### TradeExtension

#### Inheritance

- BaseGlobalExtension

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|ITradeModule|tradeModule|Trade Module for Set Token|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|mapping(address => IDelegatedManager)|setManagers|Mapping from Set Token to DelegatedManager|

#### Functions

| Name  | Caller  | Description     |
|------	|------	|-------------	|
|initializeExtension|owner|Initialize the TradeExtension on the DelegatedManager|
|initializeModuleAndExtension|owner|Initialize the TradeModule on the SetToken and the TradeExtension on the DelegatedManager|
|trade|operator|Trade between whitelisted assets on a DEX|

> initializeExtension

ONLY OWNER: Initialize the TradeExtension on the DelegatedManager

```solidity
function initializeExtension(IDelegatedManager _manager) external {
    require(msg.sender == _manager.owner(), "Must be owner")
    require(_manager.setToken().isInitializedModule(tradeModule), "Module must be initialized");
    require(_manager.isPendingExtension(address(this)), "Extension must be pending");

    _manager.initializeExtension();

    setManagers[_manager.setToken()] = _manager;
}
```

> initializeModuleAndExtension

ONLY OWNER: Initialize the TradeModule on the SetToken and the TradeExtension on the DelegatedManager

```solidity
function initializeModuleAndExtension(IDelegatedManager _manager) external {
    require(msg.sender == _manager.owner(), "Must be owner")
    require(_manager.setToken().isInitializedModule(tradeModule), "Module must be pending");
    require(_manager.isPendingExtension(address(this)), "Extension must be pending");

    tradeModule.initialize(_manager.setToken());

    _manager.initializeExtension();

    setManagers[_manager.setToken()] = _manager;
}
```

### BasicIssuanceExtension

#### Inheritance

- BaseGlobalExtension

##### FeeState

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|feeRecipient|Address to accrue fees to|
|uint256|maxIssueFee|Max issuance fee manager commits to using (1% = 1e16, 100% = 1e18)|
|uint256|issueFee|Percent of Set accruing to manager on issuance (1% = 1e16, 100% = 1e18)|
|uint256|maxRedeemFee|Max redemption fee manager commits to using (1% = 1e16, 100% = 1e18)|
|uint256|redeemFee|Percent of Set accruing to manager on redemption (1% = 1e16, 100% = 1e18)|

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|IIssuanceModule|issuanceModule|Issuance Module for Set Token|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|mapping(address => IDelegatedManager)|setManagers|Mapping from Set Token to DelegatedManager|

#### Functions

| Name  | Caller  | Description     |
|------	|------	|-------------	|
|initializeExtension|owner|Initialize the BasicIssuanceExtension on the DelegatedManager|
|initializeModuleAndExtension|owner|Initialize the BasicIssuanceModule on the SetToken and the BasicIssuanceExtension on the DelegatedManager|
|updateIssueFee|owner|Update issue fee on IssuanceModule|
|updateRedeemFee|owner|Update redeem fee on IssuanceModule|

> initializeExtension

ONLY OWNER: Initialize the BasicIssuanceExtension on the DelegatedManager

```solidity
function initializeExtension(IDelegatedManager _manager) external {
    require(msg.sender == _manager.owner(), "Must be owner")
    require(_manager.setToken().isInitializedModule(issuanceModule), "Module must be initialized");
    require(_manager.isPendingExtension(address(this)), "Extension must be pending");

    _manager.initializeExtension();

    setManagers[_manager.setToken()] = _manager;
}
```

> initializeModuleAndExtension

ONLY OWNER: Initialize the BasicIssuanceModule on the SetToken and the BasicIssuanceExtension on the DelegatedManager

```solidity
function initializeModuleAndExtension(IDelegatedManager _manager, FeeState _feeSettings) external {
    require(msg.sender == _manager.owner(), "Must be owner")
    require(_manager.setToken().isInitializedModule(tradeModule), "Module must be pending");
    require(_manager.isPendingExtension(address(this)), "Extension must be pending");

    issuanceModule.initialize(_manager.setToken(), _feeSettings);

    _manager.initializeExtension();

    setManagers[_manager.setToken()] = _manager;
}
```

### StreamingFeeSplitExtension

#### Inheritance

- BaseGlobalExtension

#### Structs

##### FeeState

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|feeRecipient|Address to accrue fees to|
|uint256|maxStreamingFeePercentage|Max streaming fee manager commits to using (1% = 1e16, 100% = 1e18)|
|uint256|streamingFeePercentage|Percent of Set accruing to manager annually (1% = 1e16, 100% = 1e18)|
|uint256|lastStreamingFeeTimestamp|Timestamp last streaming fee was accrued|

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|IStreamingFeeModule|streamingFeeModule|Streaming Fee Module for Set Token|

#### Public Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|mapping(address => IDelegatedManager)|setManagers|Mapping from Set Token to DelegatedManager|

#### Functions

| Name  | Caller  | Description     |
|------	|------	|-------------	|
|accrueFeesAndDistribute|public|Accrue fees and distribute to owner and methodologist|
|initializeExtension|owner|Initialize the StreamingFeeSplitExtension on the DelegatedManager|
|initializeModuleAndExtension|owner|Initialize the StreamingFeeModule on the SetToken and the StreamingFeeSplitExtension on the DelegatedManager|
|updateStreamingFee|owner|Migrate existing Set Token to a DelegatedManager manager|
|updateFeeRecipient|owner|Update fee recipient|

> initializeExtension

ONLY OWNER: Initialize the StreamingFeeSplitExtension on the DelegatedManager

```solidity
function initializeExtension(IDelegatedManager _manager, FeeState _feeSettings) external {
    require(msg.sender == _manager.owner(), "Must be owner")
    require(_manager.setToken().isInitializedModule(streamingFeeModule), "Module must be initialized");
    require(_manager.isPendingExtension(address(this)), "Extension must be pending");

    _manager.initializeExtension();

    setManagers[_manager.setToken()] = _manager;
}
```

> initializeModuleAndExtension

ONLY OWNER: Initialize the StreamingFeeModule on the SetToken and the StreamingFeeSplitExtension on the DelegatedManager

```solidity
function initializeModuleAndExtension(IDelegatedManager _manager, FeeState _feeSettings) external {
    require(msg.sender == _manager.owner(), "Must be owner")
    require(_manager.setToken().isInitializedModule(tradeModule), "Module must be pending");
    require(_manager.isPendingExtension(address(this)), "Extension must be pending");

    streamingFeeModule.initialize(_manager.setToken(), _feeSettings);

    _manager.initializeExtension();

    setManagers[_manager.setToken()] = _manager;
}
```

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