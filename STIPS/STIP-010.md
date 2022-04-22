
# PerpV2 Basis Trading Module STIP


## Motivation

This STIP proposes to develop a module contract to enable asset managers to build all types of basis trading products on Perpetual Protocol V2.


## Background Information

PerpV2LeverageModule is a generic module that allows asset managers to perform various actions on the Perpetual Protocol V2. It settles funding on the Perpetual protocol during issuance, redemption, trade, withdrawal and other operations. It doesn’t track funding as a separate entity.  \
For basis trading products, we have to reinvest accrued funding to generate yield. Hence funding plays a crucial role and we need a way to track funding before they are settled due to SetToken’s interactions with the Perpetual protocol. Along with that, we need a way to withdraw accrued funding, collect performance fees and then absorb it as a default position on the Set for reinvestment.


## Open Questions



1. We would have to withdraw 100% of our accrued funding. What are the implications of that?
    - Increased gas costs. But since [deposit is not gas intensive](https://kovan-optimistic.etherscan.io/tx/971019) hence not a major issue.
2. How does solutions proposed for tracking funding and performance fee collection scale for non-USDC collateral?
    - Settlement token is always USDC, hence it should work for all product types irrespective of the collateral asset.
3. How does this solution of tracking funding scale for other product types in the basis trading category?
    - It adds an additional withdrawal and deposit step in USDC margin single-sided short basis products.
    - Gas used in [withdrawal](https://kovan-optimistic.etherscan.io/tx/965486) and [deposit](https://kovan-optimistic.etherscan.io/tx/971019). Net ~3M additional gas.
5. Can we resolve issuance/redemption funding fees discount/premium by removing funding as part of the equation to determine usdc transfer amount in / out?
    - No because then we introduce issuance/redemption premium/discount.


## Feasibility Analysis


### PerpV2 Funding Explained


#### When does PerpV2 settle funding and owedRealizedPnl?



1. Funding is settled to owedRealizedPnl on certain Perpetual protocol actions
    - OwedRealizedPnl also contains market maker fees and realized Pnl due to trades.
    - Hence it is not possible to extract historical funding out of it once funding has been settled to it.
2. OwedRealizedPnl is settled to collateral on every withdraw


#### Which PerpV2 function calls lead to settlement of funding? 

Clearing house provides an API to manually settle funding.



* **settleAllFunding**
    * External function that iterates over passed in markets and calls _settleFunding for each of them.

ClearingHouse auto settles funding by calling _settleFunding as the first step during the following PerpV2 operations.



* addLiquidity
* removeLiquidity
* openPosition
* closePosition
* Liquidate
* cancelExcessOrders
* cancelAllExcessOrders
* quitMarket

Vault auto settles funding during the following PerpV2 operations



* Withdraw


#### PerpV2 Vault#withdraw

Vault.withdraw



* Calls settleAllFunding
* Settle owedRealizedPnl in accountBalance
    * Sets owed realized pnl to zero
    * Return owedRealizedPnl
* Checks enough to withdraw
* Updates users balance
    * Balances[trader][token] = Balances[trader][token] -  amountWithdraw + owedRealizedPnL
    * Hence settles owedRealizedPnl to balance


#### How does PerpV2 Settle funding?

ClearingHouse._settleFunding



* On a high-level, the calls are
* Exchange.settleFunding
    * Calculates funding and fundingGrowthGlobal
        * Funding is calculated using the same formula as the one used in getPendingFundingPayment
    * Updates state so that it performs calculations only once per block
    * Returns funding and fundingGrowthGlobal
* AccountBalance.modifyOwedRealizedPnl
    * Just adds to _owedRealizedPnlMap[trader]
* AccountBalance.updateTwPremiumGrowthGlobal
    * Just updates _accountMarketMap[trader][baseToken].astTwPremiumGrowthGlobalx96


### **Option 1**

Write a new PerpV2 Basis Trading module that extends the PerpV2 Leverage module. It keeps track of funding that is about to be settled due to an action on Perpetual Protocol. 

Operations before which we should track funding


<table>
  <tr>
   <td><strong>Function</strong>
   </td>
   <td><strong>Why</strong>
   </td>
  </tr>
  <tr>
   <td>moduleIssueHook
   </td>
   <td>Tracks funding that will be settled during issuance.
   </td>
  </tr>
  <tr>
   <td>moduleRedeemHook
   </td>
   <td>Tracks funding that will be settled during redemption.
   </td>
  </tr>
  <tr>
   <td>withdraw
   </td>
   <td>Tracks funding that will be settled when withdrawing USDC.
   </td>
  </tr>
  <tr>
   <td>Withdraw Funding
   </td>
   <td>Tracks funding that will be settled when withdrawing & absorbing funding.
   </td>
  </tr>
  <tr>
   <td>trade
   </td>
   <td>Tracks funding that will be settled when opening a position on PerpV2.
   </td>
  </tr>
</table>


How and when do we collect performance fees?


<table>
  <tr>
   <td><strong>Function</strong>
   </td>
   <td><strong>Performance Fee</strong>
   </td>
  </tr>
  <tr>
   <td>Withdraw Funding
   </td>
   <td>Performance fee is charged on the notional funding withdrawn.
<p>
<em>performanceFee = FundingWithdrawn * performanceFeePercentage</em>
   </td>
  </tr>
  <tr>
   <td>moduleRedeemHook
   </td>
   <td>Performance fee is charged on the funding accrued after the last reinvestment.
<p>
<em>performanceFee = (Total funding accrued / total supply) * redeemAmount * performanceFeePercentage</em>
   </td>
  </tr>
  <tr>
   <td>moduleIssueHook
   </td>
   <td>An issuer pays the pending funding amount while entering the set, and he ends up paying performance fee on total funding accrued at the end of the reinvestment period. 

Example,
<ul>
    <li> Set supply: 1, Set NAV except funding: 100, Revinesting period: 7 days
    <li> Pending funding on the 3rd day: 5
    <li> A issues 1 Set on the 3rd day for (100+5) USDC
    <li> New Set supply: 2
</ul>
<p>
Case 1: Give performance fee discount during issuance.
<p>
Scenario 1:
<ul>

<li>Pending funding on the 7th day: 20

<li>Reinvested amount: 18, Performance Fee at 10%: 2

<li>New Set NAV including funding: 100 + (18/2) = 109

<li>Value of Set held by A: 109

<li>Net fees paid by A: 1 (which is 10% of 10, which is the amount of funding that A would have received if he had been their from the beginning of the reinvestment period)

<p>
</ul>
Scenario 2:
<ul>

<li>Pending funding on the 7th day: 10

<li>Reinvested amount: 9, Performance Fee at 10%: 1

<li>New Set NAV including funding: 100 + (9/2) = 4.5

<li>Value of Set held by A: 104.5

<li>Net fees paid by A: 0.5 (which is 10% of5, which is the amount of funding that A would have received if he had been their from the beginning of the reinvestment period)

<li>Also notice A is in loss at the end of this reinvestment interval, due to paying fee on funding that he didn’t receive as yield.

<p>
</ul>
Case 2: Performance Fee is discounted during issuance.
<p>
Scenario 1:
<ul>

<li>Fee discounted during issuance on 3rd day: 5 * 10% = .5

<li>USDC used to issue 1 set : 100 + 5 - .5 = 104.5

<li>Pending funding on 7th day: 20

<li>Reinvested amount: 18

<li>New Set NAV including funding: 100 + 18/2 = 109

<li>Value of Set held by A: 109

<li>Net funding received by A: 109-104.5 = 4.5
</ul>
<p>

Scenario 2:
<ul>

<li>Fee discounted during issuance on 3rd day: = .5

<li>Pending funding on 7th day: 10

<li>Reinvested amount: 9

<li>Value of set held by A: 104.5

<li>Net fees paid by A: .5

<li>Net funding received by A: 0

<p>
This solution works only if net tracked funding on the reinvestment date is greater than or equal to the funding during issuance. Otherwise, the issuer gets a greater discount which is coming from the existing set token holders. Hence, this solution is not desirable.
<p>
Conclusion: We will not give a discount during issuance.
</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
</ul>
   </td>
  </tr>
</table>



### Timeline

* Spec + Review: 3-4 days
* Implementation and unit tests: 3 days
* Internal review: 2 days
* Deployment script: 1 day
* On-chain testing: 1 day


### Checkpoint 1

Reviewer: 


### Requirements

1. Track and store funding before any operation which leads to funding being settled to owed realized PnL.
2. Withdraws collateral, takes fees and then absorbs it as a default position.
3. Works for the following [all possible basis trading product types](https://docs.google.com/spreadsheets/d/1QPSmHUHcGmfHT4UaNhuY7GzCfRzBSEHKfe4zM9YqkkI/edit#gid=1106601698)
    1. Delta neutral strategy for ETH
    2. Delta neutral strategy on all perp assets
    3. Meta structured product that maximizes delta neutral yield
    4. Single-sided yield for ETH
    5. Single-sided yield for all perp assets
    6. Index of single-sided yield for all perp assets
    7. Delta neutral leveraged yield for ETH
    8. Delta neutral leveraged yield for all perp assets


### User Flows

User issues Set using SlippageIssuanceModule

* SlippageIssuanceModule calls BasisTradingModule issuance hook
* BasisTradingModule reads funding from Perpetual protocol and updates stored funding value
* BasisTradingModule opens required positions and updates USDC external position unit on the SetToken
* Like before, all pending funding payments and accrued owedRealizedPnl are attributed to the current set holders

User redeems Set using SlippageIssuanceModule

* SlippageIssuanceModule calls BasisTradingModule redeem hook
* BasisTradingModule reads funding from Perpetual protocol and updates stored funding value
* BasisTradingModule closes required positions and updates USDC external position unit on the SetToken
* Like before, any pending funding payments and ‘owedRealizedPnl’ are socialized in this step so that redeemers pay/receive their share of them.

Manager deposit 100 USDC to PerpV2

* Deposit required collateral to PerpV2
* Funding isn’t tracked because deposit doesn’t settle funding on PerpV2.

Manager withdraws 100$ of funding for reinvestment

* Reads pending funding and updates funding value.
* Settles funding and withdraws 100$ of funding in settlement token (USDC) and reduces pending funding value.
* Transfers x% of 100$ to manager fee recipient as performance fees
* Absorbs (100-x)% as default position unit. This default position can then be traded for any spot asset using the TradeModule or deposited back to perpetual protocol for opening perpetual position.


## Checkpoint 2

Reviewer: 


## Specifications


### PerpV2BasisTradingModule 


#### Inheritance

PerpV2LeverageModule


#### Events

FundingWithdrawn



* ISetToken _setToken
* IERC20 _settlementToken
* Uint256 _amountWithdrawn
* Uint256 _managerPerformanceFee
* Uint256 _protocolPerformanceFee


#### Struct

PerformanceFeeState



* feeRecipient
* maxPerformanceFeePercentage
* performanceFeePercentage


#### Public Variables


<table>
  <tr>
   <td>Type
   </td>
   <td>Name
   </td>
   <td>Description
   </td>
  </tr>
  <tr>
   <td>mapping(ISetToken => uint256)
   </td>
   <td>settledFunding
   </td>
   <td>Keeps track of settled funding yet to be reinvested.
<p>
Not storing in units because we update it continuously and small amounts added multiple times can amount to >1 unit of funding, provided we are storing notional amounts.
   </td>
  </tr>
  <tr>
   <td>mapping(ISetToken => PerformanceFeeState)
   </td>
   <td>performanceFeeStates
   </td>
   <td>Stores performance fees state for given SetToken.
   </td>
  </tr>
  <tr>
   <td>uint256
   </td>
   <td>PROTOCOL_PERFORMANCE_FEE_INDEX
   </td>
   <td>Index that stores protocol performance fee % on the controller
   </td>
  </tr>
</table>



#### Functions

Different places in contract where we would have to call __updateOwedSettledFunding_



* moduleIssueHook
* moduleRedeemHook
* withdraw(_setToken, _quantityCollateralUnits)
* withdrawFunding(_setToken, _fundingNotional)
* trade

initialize(ISetToken _setToken, PerformanceFeeState memory _performanceFeeSettings)



* _validatePerformanceFeeState(_performanceFeeSettings)
* …. Existing initialize function
* performanceFeeStates[_setToken] = _peroformanceFeeSettings;

_moduleIssueHook(ISetToken _setToken, uint256 _setTokenQuantity)_



* _Existing checks_
* __updateSettledFunding(_setToken)_
* int256 newExternalPositionUnit = _executePositionTrades(_setToken, _setTokenQuantity, true, false);
* __setToken.editExternalPositionUnit(_
    * _address(collateralToken),_
    * _address(this),_
    * _newExternalPositionUnit_
    * _)_

moduleRedeemHook_(ISetToken _setToken, uint256 _setTokenQuantity)_



* _Existing checks_
* __updateSettledFunding(_setToken)_
* int256 newExternalPositionUnit = _executePositionTrades(_setToken, _setTokenQuantity, false, false);
* _if(settledFunding[_setToken] > 0)_
    * _// Charge performance fee_
    * _performanceFeeUnit = settledFunding[_setToken].preciseDiv(_setTokenQuantity).preciseMul(performanceFee)_
    * _newExternalPositionUnit = newExternalPositionUnit.sub(performanceFeeUnit)_
* __setToken.editExternalPositionUnit(_
    * _address(collateralToken),_
    * _address(this),_
    * _newExternalPositionUnit_
    * _)_

trade(ISetToken _setToken, address _baseToken, int256 _baseQuantityUnits, uint256 _quoteBoundQuantityUnits, bool _trackSettledFunding)



* _ActionInfo memory actionInfo = _createAndValidateActionInfo(.....);_
* _if(_trackSettledFunding)_
    * __updateSettledFunding(_setToken)_
* (uint256 deltaBase, uint256 deltaQuote) = _executeTrade(actionInfo);
* _…. Existing trading function_

_withdrawFunding(ISetToken _setToken, uint256 _fundingNotional, bool _trackSettledFunding)_



* _require(_fundingNotional &lt;= settledFunding[_setToken], “No rugging investors”)_
* _if(_trackSettledFunding)_
    * __updateSettledFunding(_setToken)_
* __withdraw(_setToken, _fundingNotional)_
* __absorb(_setToken, _fundingNotional)_
* _Update External USDC Position Unit_
    * _setToken.editExternalPositionUnit(..)_
* __settledFunding[_setToken] -= _fundingNotional_

_updatePerformanceFee(ISetToken _setToken, uint256 _newFee)_



* _require(_newFee &lt; performanceFeeStates[_setToken].maxPerformanceFeePercentage, “!>”);_
* _uint256 feesNotional = _settledFunding[_setToken].preciseMul(performanceFeeStates[_setToken].performanceFeePercentage.add(controller.getModuleFee(address(this), _PROTOCOL_PERFORMANCE_FEE_INDEX_));_
* __withdraw(_setToken, feesNotional)_
* _(managerFee, protocolFee, totalFee) = _handleFees(_setToken, settledFunding[_setToken])_
    * _// distributes performance fees to manager and protocol_
    * _// fees = amount passed in * feePercentage_
* _performanceFeeStates[_setToken].performanceFeePercentage = _newFee;_
* _Emit PerformanceFeesUpdated()_

_updatePerformanceFeeRecipient(ISetToken _setToken, address _newFeeRecipient)_



* _require(_newFeeRecipient != address(0), “! zero address”);_
* _performanceFeeStates[_setToken].feeReceipient = _newFeeRecipient;_
* _Emit PerformanceFeeRecipientUpdated(_setToken, _newfeeRecipient)_

__validatePeroformanceFeeState(PerformanceFeeState memory _performancFeeSettings)_



* require(_performanceFeeSettings.feeRecipient != address(0), “!zero address”);
* require(_performanceFeeSettings.maxStreamingFeePercentage &lt; PreciseUnitMath.preciseUnit, “!100%”);
* require(_performanceFeeSettings.performnceFeePercentage &lt;= _performanceFeeSettings.maxPerformanceFeePercentage, “!>”);

__updateSettledFunding(ISetToken _setToken)_



* _Uint256 funding = getPendingFundingPayment(address _setToken, address _baseToken)_
* __settledFunding[_setToken] += funding_

__absorb(_setToken, _fundingNotional)_



* _// Collateral token Should have been named settlementToken in PerpV2 module_
* _// will be named settlement token from here on_
* _If _fundingNotional> 0_
    * _(managerFee, protocolFee, totalFee) = _handleFees(_setToken, _fundingNotional)_
        * _// distributes performance fees to manager and protocol_
        * _// fees = _fundingNotional * feePercentage_
    * _newUnit = settlementTokenBalance.sub(totalFee).preciseDiv(_setToken.totalSupply)_
    * __setToken.editDefaultPosition(settlementToken, newUnit)_
    * _Emit fundingWithdrawn_

__handleFees(_setToken, _fundingNotional)_



* _Similar implementation as AirdropModule # _handleFees_


## Checkpoint 3

Reviewer: 
