# STIP-012: Intrinsically Productive Assets

*Using template v0.1*

## Abstract

Currently, SetToken managers using self-service manager contracts do not have the capabilities to improve their passive returns by depositing their SetToken's components into popular interest-yielding protocols like Aave, Compound, and Yearn.

## Motivation

This STIP proposes we provide infrastructure for the [DelegatedManager system](https://docs.tokensets.com/developers/contracts/strategies/delegatedmanager-system) to deposit assets into yield bearing vaults, claim protocol rewards earned from those deposits, and withdraw assets at any time. This enables the self-service manager system to access the functionality provided by the `WrapModuleV2`, `ClaimModule`, and `AirdropModule`.

## Background Information

[WrapModuleV2](https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/modules/v1/WrapModuleV2.sol) - Module that enables the wrapping of ERC20 and Ether positions via third party protocols. The WrapModuleV2 works in conjunction with WrapV2Adapters. There are WrapV2Adapters for blank, blank, blank.

[ClaimModule](https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/modules/v1/ClaimModule.sol) - Module that enables managers to claim tokens from external protocols given to the Set as part of participating in incentivized activities such as depositing assets. The ClaimModule works in conjunction with ClaimAdapters. There are ClaimAdapters for blank, blank, blank

[AirdropModule](https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/modules/v1/AirdropModule.sol) - Module that enables managers to absorb tokens sent to the SetToken into the token's positions. With each SetToken, managers are able to specify

- the airdrops they want to include
- an airdrop fee recipient
- airdrop fee
- whether all users are allowed to trigger an airdrop


## Open Questions

- [ ] Is more functionality necessary to provide asset managers needs for intrinsically productive assets?
    - There is potential need for the ability to trade and wrap in a single transaction, especially when dealing with ETH. This has come up with the [CurveStEthExchangeAdapter](https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/integration/exchange/CurveStEthExchangeAdapter.sol).

## Feasibility Analysis

### WrapExtension

This extension submits calls to the `WrapModuleV2` from the `DelegatedManager`. This contract enables `DelegatedManager`'s to wrap and unwrap ERC20 and Ether positions.

### Individual ClaimExtension and AirdropExtension

These extensions submit calls to the `ClaimModule` and `AirdropModule` from the DelegatedManager. These extensions maintain the level of functionality of the modules, but require two transactions to claim external protocol rewards and absorb them in to the `SetToken`.

### Combined ClaimExtension and AirdropExtension

This single extension submits calls to the `ClaimModule` and `AirdropModule` from the DelegatedManager. This extension would claim external protocol rewards and absorb them in to the `SetToken` in a single transaction. Endpoints for the standalone `AirdropModule` functionality are provided.

### Recommended Solution

We recommend implementing the `WrapExtension` and the `ClaimExtension` which calls both the `ClaimModule` and `AirdropModule`. This solutions allows `DelegatedManager`'s with a single transaction experience for wrapping assets, claiming external protocol rewards into a `SetToken`, and unwrapping assets.

## Timeline

|  Action               |  End Date  |
|---                    |---         |
| Technical Spec        |   4/25     |
| Implementation        |   4/29     |
| Auditors              |   TBD      |
| Launch                |   TBD      |

## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

## Proposed Architecture Changes
A diagram would be helpful here to see where new feature slot into the system. Additionally a brief description of any new contracts is helpful.
## Requirements

### WrapExtension

- Allow `owner` to enable functionality of WrapModuleV2 with only a state change and no contract deployment
- Allow privileged `operator`(s) to wrap and unwrap ERC20 and ether positions via third party protocols

### ClaimExtension
- Allow `owner` to enable functionality of ClaimModule and AirdropModule with only a state change and no contract deployment
- Use ClaimModule to claim external protocol rewards and AirdropModule to absorb them into SetToken positions in a single transaction

## User Flows
<!-- - Highlight *each* external flow enabled by this feature. It's helpful to use diagrams (add them to the `assets` folder). Examples can be very helpful, make sure to highlight *who* is initiating this flow, *when* and *why*. A reviewer should be able to pick out what requirements are being covered by this flow. -->

### ClaimExtension.claimAndAbsorb()



## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: []
## Specification

### WrapExtension

#### Inheritance

- BaseGlobalExtension

#### Events

##### WrapExtensionInitialized

| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|_setToken|Address of SetToken with WrapExtension initialized|
|address|_delegatedManager|Address of DelegatedManager with WrapExtension initialized|

#### Global Variables

| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|IWrapModuleV2|wrapModule|WrapModuleV2 for SetToken|

#### Functions
| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|initializeModule|owner|Initializes the WrapModuleV2 on the SetToken associated with the DelegatedManager|
|initializeExtension|owner|Initializes the WrapExtension on the DelegatedManager|
|initializeModuleAndExtension|owner|Initializes the WrapModuleV2 on the SetToken and the WrapExtension on the DelegatedManager|
|removeExtension|manager|Remove an existing SetToken and DelegatedManager tracked by the WrapExtension|
|wrap|operator|Instructs the SetToken to wrap an underlying asset into a wrappedToken via a specified adapter.|
|wrapWithEther|operator|Instructs the SetToken to wrap Ether into a wrappedToken via a specified adapter.|
|unwrap|operator|Instructs the SetToken to unwrap a wrapped asset into its underlying via a specified adapter.|
|unwrapWithEther|operator|Instructs the SetToken to unwrap a wrapped asset collateralized by Ether into Wrapped Ether.|

---

#### Functions

> wrap

ONLY OPERATOR: Instructs the SetToken to wrap an underlying asset into a wrappedToken via a specified adapter.

```solidity
function wrap(
    ISetToken _setToken,
    address _underlyingToken,
    address _wrappedToken,
    uint256 _underlyingUnits,
    string calldata _integrationName,
    bytes memory _wrapData
)
    external
    onlyOperator(_setToken)
```
+ Formulate call to wrap asset from manager
    ```solidity
    bytes memory callData = abi.encodeWithSelector(
        IWrapModuleV2.wrap.selector,
        _setToken,
        _underlyingToken,
        _wrappedToken,
        _underlyingUnits,
        _integrationName,
        _wrapData
    );
    ```
+ call *_invokeManager(_manager(_setToken), address(wrapModule), callData);*

---

> wrapWithEther

ONLY OPERATOR: Instructs the SetToken to wrap Ether into a wrappedToken via a specified adapter.

```solidity
function wrapWithEther(
    ISetToken _setToken,
    address _wrappedToken,
    uint256 _underlyingUnits,
    string calldata _integrationName,
    bytes memory _wrapData
)
    external
    onlyOperator(_setToken)
```
+ Formulate call to wrap Ether from manager
    ```solidity
    bytes memory callData = abi.encodeWithSelector(
        IWrapModuleV2.wrapWithEther.selector,
        _setToken,
        _wrappedToken,
        _underlyingUnits,
        _integrationName,
        _wrapData
    );
    ```
+ call *_invokeManager(_manager(_setToken), address(wrapModule), callData);*

---

> unwrap

ONLY OPERATOR: Instructs the SetToken to unwrap a wrapped asset into its underlying via a specified adapter.

```solidity
function unwrap(
    ISetToken _setToken,
    address _underlyingToken,
    address _wrappedToken,
    uint256 _wrappedUnits,
    string calldata _integrationName,
    bytes memory _unwrapData
)
    external
    onlyOperator(_setToken)
```
+ Formulate call to unwrap asset from manager
    ```solidity
    bytes memory callData = abi.encodeWithSelector(
        IWrapModuleV2.unwrap.selector,
        _setToken,
        _underlyingToken,
        _wrappedToken,
        _wrappedUnits,
        _integrationName,
        _unwrapData
    );
    ```
+ call *_invokeManager(_manager(_setToken), address(wrapModule), callData);*

---

> unwrapWithEther

ONLY OPERATOR: Instructs the SetToken to unwrap a wrapped asset collateralized by Ether into Wrapped Ether.

```solidity
function unwrapWithEther(
    ISetToken _setToken,
    address _wrappedToken,
    uint256 _wrappedUnits,
    string calldata _integrationName,
    bytes memory _unwrapData
)
    external
    onlyOperator(_setToken)
```
+ Formulate call to unwrap Ether from manager
    ```solidity
    bytes memory callData = abi.encodeWithSelector(
        IWrapModuleV2.unwrapWithEther.selector,
        _setToken,
        _wrappedToken,
        _wrappedUnits,
        _integrationName,
        _unwrapData
    );
    ```
+ call *_invokeManager(_manager(_setToken), address(wrapModule), callData);*

---

### ClaimExtension

#### Inheritance

- BaseGlobalExtension


## Checkpoint 3
Before we move onto the implementation phase we want to make sure that we are aligned on the spec. All contracts should be specced out, their state and external function signatures should be defined. For more complex contracts, internal function definition is preferred in order to align on proper abstractions. Reviewer should take care to make sure that all stake holders (product, app engineering) have their needs met in this stage.

**Reviewer**:

## Implementation
[Implementation PR](https://github.com/SetProtocol/set-v2-strategies/pull/35)
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()
[Link to Deploy outputs PR]()
