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
- [ ] Should we generalize this further to allow add view functions to manager hooks that allow for better estimating of post-sync balances during issuance and redemption
    - *Answer*
## Feasibility Analysis
### The Solution
Allow the issuer and redeemer to specify a max amount of tokens they are willing to pay on issuance and a redeemer a min amount of tokens they are willing to receive on redemption. Doing this ensures that any AMM pool transactions cannot be sandwich attacked nor can a lack of liquidity inadvertently affect the issuer or redeemer. For greater gas savings we could allow the user to specify components that they want to be checked since not all components will require an interaction with an AMM pool. This allows the module to also be easily usable with components that do not require a trade since the issuer or redeemer would simply not specify they be checked.

Additionally, we will modify the view functions used to fetch the amount of tokens required for issuance and redemption to take into account any syncing that may happen during issuance/redemption. This will require modules that implement hooks to calculate the expected position unit changes of a issue/redeem transaction and return the position unit adjustments to the issuance module. The issuance module can then add these adjustments to the implied issuance token amounts from the position units to give a clearer picture to the issuer/redeemer of the expected token flows for the transaction. This will allow the issuer to set better token limits on issue/redeem in an easy manner by just applying a buffer to the returned token amounts.
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

**SlippageIssuanceModule** - Module that enables issuance and redemption for positions that incur some form of "slippage" upon issuance. Slippage occurs when positions aren't exactly replicable because internal or external protocol state must be updated during the transaction (i.e. executing Perp trade or accruing AAVE interest).
## Requirements
- Any solution should support issuance of Sets holding the following assets:
    - Perpetuals
    - Non-rebasing ERC20s (i.e. not aTokens)
- Issuers should not be vulnerable to unexpected change in SetToken price such that they pay significantly more than they expect. Possible causes:
    - Sandwich attack during Perpetual issuance
    - Realization of interest payments on issuance leading to increase in Set value
- Good UX for the issuers/redeemers such that they will not accidentally have transactions revert or make it difficult to approve the right amount of tokens
## Interfaces
In order to meet the above requirements we propose two main interfaces for issuance and redemption:
```solidity
    function issueWithSlippage(
        ISetToken _setToken,
        uint256 _setQuantity,
        address[] memory _checkedComponents,
        uint256[] memory _maxTokenAmountsIn,
        address _to
    )
        external
```
The main additions to this interface from the typical interface are:
- `_checkedComponents` - This is an array of components the user wants checked for slippage (does not need to be all components in the Set)
- `_maxTokenAmountsIn` - Max amounts of the checked components a user is willing to pay, maps to index in `_checkedComponents`


```solidity
    function redeemWithSlippage(
        ISetToken _setToken,
        uint256 _setQuantity,
        address[] memory _checkedComponents,
        uint256[] memory _minTokenAmountsOut,
        address _to
    )
        external
```
- `_checkedComponents` - This is an array of components the user wants checked for slippage (does not need to be all components in the Set)
- `_minTokenAmountsOut` - Min amounts of the checked components a user is willing to receive, maps to index in `_checkedComponents`


## User Flows
### issueWithSlippage
An issuer wants to issue a Set containing a Perp in order to get leveraged exposure to ETH
1. Issuer calls getRequiredIssuanceUnits to get the amount of tokens necessary to collateralize 1 Set containing a USDC collateralized ETH Perp position
2. Issuer (optionally) calculates a slippage buffer in case the liquidity profile of the ETH perp market changes (similar to Uniswap/Sushiswap etc trades) to create a max amount of USDC they are willing to collateralize the Set with (since the Perp position is USDC collateralized)
3. Issuer approves the amount calculated in Step 2 to the SlippageIssuanceModule
4. Issuer calls issue
5. SlippageIssuanceModule calls hook on PerpModule
6. PerpModule trades into required amount of base token collateral (note: this temporarily spikes the leverage ratio)
7. Slippage is calculated and added to amount of perpetual collateral per Set to determine how much of the USDC token is required to collateralize the ETH perpetual position
8. Required collateral amounts are checked against max amounts passed in by issuer, revert if greater than max
8. Required perpetual collateral is transferred in to PerpIssuanceModule
9. Collateral is deposited into Perpetual Protocol via PerpModule (note: this puts the leverage ratio back in line)
10. Any other collateral is transferred in
11. SetToken is minted
### redeemWithSlippage
The previous issuer wants to redeem the Set they minted in order to close down their leveraged ETH exposure
1. Redeemer calls getRequiredRedemptionUnits to get the amount of tokens they expect to receive for burning their 1 Set 
2. Redeemer calculates a slippage buffer in case the liquidity profile of the ETH perp market changes to create a min amount of USDC they are willing to receive for redeeming the Set
3. Redeemer calls redeem
4. PerpModule trades out of required amount of base token collateral (note: this temporarily lowers the leverage ratio)
5. Slippage is calculated and removed from the amount of perpetual collateral per Set to determine how much of the USDC token is returned to the redeemer (redeemer gets value of position net of slippage)
6. SetToken is burned
7. Calulated returned collateral amounts are checked against min amounts passed in by issuer, revert if less than min
8. Collateral is withdrawn from Perpetual protocol
9. Collateral is transferred back to redeemer
## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: []
## Specification
### [Contract Name]
#### Inheritance
- DebtIssuanceModule

#### Functions
| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|issueWithSlippage|Anyone|Issue a set with limits on amount of collateral that can be spent to collateralize|
|redeemWithSlippage|Anyone|Redeem a set with minimum on amount of collateral expected to be returned|
|getRequiredComponentIssuanceUnits|Anyone|Gets expected amount of tokens for issuance accounting for position changes during issuance|
|getRequiredComponentRedemptionUnits|Anyone|Gets expected amount of tokens for redemptions accounting for position changes during redemption|
#### Functions
> issueWithSlippage(ISetToken _setToken, uint256 _setQuantity, address[] memory _checkedComponents, uint256[] memory _maxTokenAmountsIn, address _to) external
- Validate setQuantity > 0 and `_checkedComponents` has unique entries
- Call manager hook contract
- Cycle through and call module pre issuance hooks
- Calculate any issuance fees, manager and protocol
- Calculate the required issuance units
- Validate token transfer limits, cycle through `_checkedComponents` array and check that corresponding `_maxTokenAmountsIn` is greater than required transfer amount for that component
- Resolve equity positions (transfer in tokens to Set and send out to any external positions)
- Resolve debt positions (add to Set debt and transfer new debt to user)
- Mint extra SetTokens for fees to manager and protocol (if necessary)
- Mint SetToken to `_to` address

> redeemWithSlippage(ISetToken _setToken, uint256 _setQuantity, address[] memory _checkedComponents, uint256[] memory _minTokenAmountsOut, address _to) external
- Validate setQuantity > 0 and `_checkedComponents` has unique entries
- Cycle through and call module pre issuance hooks
- Burn `_setQuantity` amount of SetToken from `msg.sender`
- Calculate any issuance fees, manager and protocol
- Calculate the required issuance units
- Validate token transfer limits, cycle through `_checkedComponents` array and check that corresponding `_minTokenAmountsOut` is less than required transfer amount for that component
- Resolve debt positions (transfer in debt position from `msg.sender` and use to pay back debt in external protocol)
- Resolve equity positions (withdraw any tokens from external protocols and return equity to `_to` address)
- (Re)mint extra SetTokens for fees to manager and protocol (if necessary)

> getRequiredComponentIssuanceUnits(ISetToken _setToken, uint256 _quantity) external view
- Calculate the total amount of Sets to issue including fees, `totalQuantity`
- Cycle through each registered module issuance hook doing the following
    - Call `getIssuanceAdjustments` on each module
    - Keep a running sum of each component's equity and debt adjustments
- Calculate and return the amount of tokens required for issuance
    - Get the equity and debt units for each component in the Set, sum any external equity positions to Default positions
    - Add adjustments calculated in previous step to equity and debt units calculated in this step
    - Multiply resulting units by `totalQuantity`

> getRequiredComponentRedemptionUnits(ISetToken _setToken, uint256 _quantity) external view
- Calculate the total amount of Sets to issue including fees, `totalQuantity`
- Cycle through each registered module issuance hook doing the following
    - Call `getRedemptionAdjustments` on each module
    - Keep a running sum of each component's equity and debt adjustments
- Calculate and return the amount of tokens required for issuance
    - Get the equity and debt units for each component in the Set, sum any external equity positions to Default positions
    - Add adjustments calculated in previous step to equity and debt units calculated in this step
    - Multiply resulting units by `totalQuantity`

## Checkpoint 3

**Reviewer**:

## Implementation
[Link to implementation PR](https://github.com/SetProtocol/set-protocol-v2/pull/162)
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()  
[Link to Deploy outputs PR]()
