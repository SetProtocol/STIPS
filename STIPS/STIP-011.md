# STIP-011: Batch Trading
*Using template v0.1*
## Abstract

Rebalancing Set Token indicies is a complicated process which involves managers and operators executing multiple transactions. Making this process simpler improves the experience of current asset managers and makes index management more approachable for new asset managers.

## Motivation

This STIP proposes we provide infrastrcuture to batch trade calls together into one transaction. This will make Set Token index rebalancing less cumbersome for asset managers by reducing the number of transactions necessary to rebalance an index.

## Background Information

[TradeModule](https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/modules/v1/TradeModule.sol) with [TradeExtension](https://github.com/SetProtocol/set-v2-strategies/blob/master/contracts/extensions/TradeExtension.sol) - This module and accompanying extension allow managers to make individual swaps. A complete index rebalance could be done by executing multiple transactions through the `TradeExtension`.

[GeneralIndexModule](https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/modules/v1/GeneralIndexModule.sol) - This module is tailored to the complete rebalance of Set Token indicies. The manager begins the rebalance by specifying a target allocation in `startRebalance()`. Once this allocation is passed in, allowed traders can submit rebalance transactions by calling `trade()` and specifying the component they wish to rebalance.

[Multicall2](https://github.com/makerdao/multicall/blob/master/src/Multicall2.sol) - This contract aggregates results from multiple contract constant function calls. This reduces the number of separate JSON RPC requests and guarantees that all values returned are from the same block. `Multicall2` allows calls within the batch to fail.

## Open Questions
- [ ] How often will trades within a batch fail? What factors effect this failure rate?
    - *Answer*
- [ ] Is there a trade ordering algorithm which can be provided to managers to sequence trades into batches for a desired rebalance?
    - *Answer*
## Feasibility Analysis

### BatchTradeExtension

This extension submits multiple calls to the `TradeModule` in a single transaction. Events are emitted if any of the individual calls fail. This implementation is simple but has some gas inefficiencies. Permissions are checked on each call to the `TradeModule`, but permissions only need to be checked once. Also if components are present in multiple trades in the batch, their position will be updated multiple times when only one update per component is necessary.

### BatchTradeModule

This module is similar to the `TradeModule`, but executes multiple trades in a single transaction. Logic duplication during permission checking and position updating can be avoided with this module. This implementation is more complicated because it requires a module with new logic, an accompanying extension, and integrations added to the `IntegrationRegistry`.

### Recommended Solution

We recommend implementing the `BatchTradeExtension`. This solution requires less smart contract implementation, leverages all current and future `TradeModule` integrations, and only performs from an estimated ~10% worse on gas usage compared to the `BatchTradeModule`.

## Timeline

|  Action               |  End Date  |
|---                    |---         |
| Technical Spec        |   4/18     |
| Implementation        |   4/20     |
| Auditors              |   4/22     |
| Launch                |   4/29     |

## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

## Proposed Architecture Changes
A diagram would be helpful here to see where new feature slot into the system. Additionally a brief description of any new contracts is helpful.
## Requirements
These should be a distillation of the previous two sections taking into account the decided upon high-level implementation. Each flow should have high level requirements taking into account the needs of participants in the flow (users, managers, market makers, app devs, etc)
## User Flows
- Highlight *each* external flow enabled by this feature. It's helpful to use diagrams (add them to the `assets` folder). Examples can be very helpful, make sure to highlight *who* is initiating this flow, *when* and *why*. A reviewer should be able to pick out what requirements are being covered by this flow.
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
