# STIP-008: Slippage Issuance
*Using template v0.1*
## Abstract
We need to be able to support issuance of assets that require a trade to replicate the position without opening issuers/redeemers to sanwdwich attacks. Effectively, we need to be able to set slippage limits on issuances.
## Motivation
Currently our issuance processes only allow for exact replication of positions based on the assets held in the Set or based on the NAV of assets held by the Set. However, there may be some issuance processes that require executing a trade as part of entering a position (ie Perpetuals). We need to devise some way to ensure that issuers and redeemers can't get sandwich attacked as part of the issuing process.

## Background Information
### Current Issuance Flows
Currently we have two general types of issuance:
1. **Issuance through Replication** - Issuing by exactly replicating all of the assets currently held by the Set (with the correct proportions)
2. **NAV Issuance** - Issuing by value one "share" in the Set and allowing a user to pay for their share with one asset

The solution we seek right now is to enable **Issuance through Replication**. When a trade is required for replication the issuer or redeemer is going to end up paying that slippage. We need to make sure that they don't pay more than they would expect, or are willing to pay, to enter or leave the Set.

The most comprehensive module we have is the `DebtIssuanceModule`. The issue and redeem interfaces for the `DebtIssuanceModule` are are such:
```
    function issue(
        ISetToken _setToken,
        uint256 _quantity,
        address _to
    )
        external
    
    function redeem(
        ISetToken _setToken,
        uint256 _quantity,
        address _to
    )
        external
```
The amount of `_setToken` issued and redeemed is fixed and the amount of assets transferred in is then calculated by the smart contract and transferred in, for this version of issuance we would want the same. This means that we need to provide some way for issuers and redeemers to specify the amount of tokens they are willing to pay since that would vary based on the spot price of the AMM at the time the transaction is mined and the slippage incurred during the issuance transaction. 

## Open Questions
Pose any open questions you may still have about potential solutions here. We want to be sure that they have been resolved before moving ahead with talk about the implementation. This section should be living and breathing through out this process.
- [ ] Question
    - *Answer*
## Feasibility Analysis
### The Solution
Allow the issuer and redeemer to specify a max amount of tokens they are willing to pay on issuance and a redeemer a min amount of tokens they are willing to receive on redemption. Doing this ensures that any AMM pool transactions cannot be sandwich attacked nor can a lack of liquidity inadvertently affect the issuer or redeemer. For greater gas savings we could allow the user to specify components that they want to be checked since not all components will require an interaction with an AMM pool. This allows the module to also be easily usable with components that do not require a trade since the issuer or redeemer would simply not specify they be checked.
## Timeline
spec: 10/27
implementation: 10/29
internal review: week of 11/1
deployment: TBD with PerpetualModule
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
