# STIP-003
## Abstract
The `DebtIssuance` module for issuing leveraged products has poor UX. For issuing, users initially pay a greater value of the collateral asset than the received leverage token value, and returns an additional refund denominated in the borrow asset. For redeeming, users must pay off all debt for the Set token by providing an additional amount of the borrow asset along with the leveraged token.

## Motivation
A new exchange issuance contract can significantly improve UX, with the downside of increased gas costs. In short we would allow users to issue Sets by sending the exact price of the amount they wish to issue. It would remove the need to send extra funds, and the user would not receive a refund at the end. For redemption, users would not have to manually pay off the debt of the leveraged token.

This feature has two main uses. First, large whales may prefer the better UX, especially since the increased gas costs are negligible compared to their trade size. An additional use case for this feature is for leveraged products issued on Polygon. Since the gas price is so low on Polygon, we can perform issuance and redemption using this contract for regular users. This removes the need to fund large liquidity mining campaigns to incentivize liquidity provision.

## Background Information
Flash loan notes
- Aave is one of the largest issuers of flash loans.
- They currently provide this service on both Ethereum and Polygon.

Leverage token notes:
- For exact input issuances and exact output redemptions we need to call sync on `CompoundLeverageModule` before calculating the price
- For Aave and Compound, we need different ways to fetch the values of the underlying asset. In Aave, it is a 1 to 1 relationship. In Compound, we need to call `balanceOfUnderlying` on the CToken contract.
- Before fetching the required collateral and debt amounts for a leveraged token, we must call sync on the associated leverage module.


Useful links:  
Aave flash loans: https://docs.aave.com/developers/guides/flash-loans  
DebtIssuanceModule: https://github.com/SetProtocol/set-protocol-v2/blob/master/contracts/protocol/modules/DebtIssuanceModule.sol  
ExchangeIssuance: https://github.com/SetProtocol/index-coop-smart-contracts/blob/master/contracts/exchangeIssuance/ExchangeIssuance.sol  

## Open Questions
- How to we calculate the issuance amount for an exact input issuance.
- How to we calculate the redemption amount for an exact output redemption
- Can we assume the leveraged tokens only have one debt position?

## Feasibility Analysis
Solution 1: Aave flash loans with a Periphery smart contract  
In this solution we would write a periphery contract called to abstract away the interactions with `DebtIssuanceModule`.   

Issuance:
- user sends an input token
- entire amount of the input token is swapped into the collateral asset (if collateral asset is not equal to input asset)
- sync called on leverage module
- the amount of additional collateral including Uniswap and Aave flash loan fees needed is calculated
- the additional amount needed is flash loaned from Aave
- collateral is wrapped into CToken / AToken
- Issuance is performed
- Refund amount is swapped back to collateral asset
- Aave flash loan is repaid

Redemption:
- user sends input leveraged token
- sync called on leverage module
- calculate amount of borrow asset required to redeem including flash loan and Uniswap fees
- flash loan borrow asset amount from Aave
- redeem leveraged token
- unwrap CToken / AToken
- sell some of returned collateral asset back to the borrow asset
- pay back Aave flash loan
- swap the rest of the collateral asset into the output asset (if borrow asset is not equal to output asset)
- send output asset to user

Calculating the required amount to flash loan for an exact input issuance is quite complicated. For an overview of how to derive the required equations, take a look at [this pdf](../assets/STIP-003/debt_exchange_issuance_math.pdf)

Solution 2: New issuance module  
In this solution, we would write a new `DebtIssuanceModule` that does much of this internal accounting itself. This removes the need to perform redundant syncs and swaps to pay off debt or to collect refunds. To do this, very careful accounting would be needed to ensure that these actions do not change the leverage ratios by too much. If the leverage ratios are changed by a certain unacceptable amount, then it would perform a correcting rebalance directly before the issuance or redemption. While this solution may require significantly less gas for most users, it comes at the cost of increased complexity and risk. For that reason, I do not recommend we go forward with this solution.

## Timeline
Spec: 7/23  
Implementation: 7/28  
Internal audit: 8/6  
Audit: 8/8  
Staging deployment and tests: 8/12  
Production deployment: 8/15  

## Checkpoint 1
**Reviewer**:

## Proposed Architecture Changes
A new contract will be built, called `LeverageTokenExchangeIssuance` which can be utilized to issue basic leveraged tokens that contain just one collateral asset and one debt position. This contract interfaces with the `DebtIssuanceModule`, `CompoundLeverageModule` and `AaveLeverageModule`. Since it is a periphery contract, it will not be a dependency of any core Set Protocol systems, nor will it have any privileged access.

## Requirements
- exact input issuance
- exact output issuance
- exact input redemption
- users can user above functions with either and ERC20 or WETH
- users can get quotes for the above trades
- users can supply exchanges and trade paths for the input/out token -> collateral/debt trades
- users can supply exchanges and trade paths for the debt -> collateral trades
- trade paths can be up to three in length
- works for set tokens with a single collateral and debt position (only basic leveraged tokens)
- produces 0 dust

## User Flows
User wants to use 1000 DAI to purchase at last 20 ETH2xFLI
- user approves at least 100 DAI to `LeverageTokenExchangeIssuance`
- user calls `issueExactInput` on `LeverageTokenExchangeIssuance`
- `LeverageTokenExchangeIssuance` returns as much ETH2xFLI as it can issue with the full 1000 DAI
- if the purchase amount is below 20, revert

User want to issue 20 ETH2xFLI for at max 0.7  ETH
- user calls `issueExactOutputETH` on `LeverageTokenExchangeIssuance` with a msg.value of 0.7
- `LeverageTokenExchangeIssuance` issues 20 ETH2xFLI with the sent ETH
- `LeverageTokenExchangeIssuance` returns 20 ETH2xFLI and refunds the unused ETH
- if the issuance costs more than 0.7 ETH, revert

User wants to redeem 10 BTC2xFLI and receive a minimum of 300 USDC
- user calls `redeemExactInput` on `LeverageTokenExchangeIssuance`
- `LeverageTokenExchangeIssuance` redeems 10 BTC2xFLI
- `LeverageTokenExchangeIssuance` returns proceeds of the redemption in USDC
- if USDC proceeds are less than 300, revert

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
