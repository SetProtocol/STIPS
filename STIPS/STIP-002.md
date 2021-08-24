# STIP-[002]: Fix aToken issuance bug
*Using template v0.1*
## Abstract
Issuance modules including BasicIssuanceModule and DebtIssuanceModule revert while trying to issue SetTokens with Aave's aToken as components.
## Motivation
We need to fix this bug to launch our next set of structured products which would include SetTokens holding aTokens as components. Fixing this bug would also enable third party managers to gain exposure to yield bearing Aave aTokens in their SetTokens.
## Background Information
In all our issuance modules, during issuing a SetToken, we make sure the sum of existing balance of the component token in the SetToken and the quantity of component token being transferred from the issuer to the SetToken is equal to the new balance of the component token in the SetToken.
```javascript
uint256 existingBalance = _componentToken.balanceOf(_setToken);
SafeERC20.transferFrom(
    _componentToken,
    _issuer,    // from
    _setToken,  // to
    _quantity
);
uint256 newBalance = _componentToken.balanceOf(_setToken);
require(newBalance == existingBalance.add(_quantity), "Invalid post transfer balance");
```
Similarly, during redemption of a SetToken, we make sure the quantity of component token being transferred from the SetToken to the redeemer subtracted from the old component token balance of the SetToken is equal to new component token balance of the SetToken.
In both cases, if the requirement is not met then we revert with "Invalid post transfer balance" message.

- In _DebtIssuanceModule#issue_ and _BasicIssuanceModule#issue_, we transfer the component token from the issuer to the SetToken using the _ModuleBase#transferFrom_ function, which internally calls the _ExplicitERC20#transferFrom_ function, which performs the above mentioned check.
- In _DebtIssuanceModule#redeem_ and _BasicIssuanceModule#redeem_, we transfer the component from the SetToken to the redeemer using _Invoke#invokeTransfer_ library function along with performing the the above mentioned check.

Now, when issuing/redeeming a SetToken which contain aToken as components, the above check fails. Cause of the failure:

Aave accrues interest every block for an aToken. Internally it updates it's state only when the core functions of LendingPool are called, but view functions such as _aToken.balanceOf()_ always return the most up-to-date value, as they account for the current block number in calculating the balance. Basically they store a scaledBalance for every user and multiply it by the current liquidity index to determine the total aToken balance of the user.
- aToken.balanceOf() returns scaledBalance * liquidityIndex
- aToken.transfer(quantity) updates the scaledBalance by adding (quantity/liquidityIndex) to it.
    - liquidityIndex = (timeDiff/secondsPerYear)*interestRate + 1, hence liquidityIndex is always > 1
    - timeDiff = block.timestamp - lastUpdatedTimestamp
    - lastUpdatedTimestamp = timestamp of last block in which there was an interaction with this aToken specific reserve, interactions include calling deposit, withdraw, repay, borrow, flashloan and swapBorrowRate functions on the reserve.

Now in our ExplicitERC20#transferFrom function,
- We call existingBalance = aToken.balanceOf(setToken), which sets 
existingBalance = initialScaledBalance * index
- By calling aToken.transfer(quantity), we update baseBalance to newScaledBalance
newScaledBalance = initialScaledBalance + (quantity/index)
- Then we set newBalance = aToken.balanceOf(setToken)
    - newBalance = newScaledBalance * index
- Finally we require, newBalance == existingBalance + quantity
    - LHS = newBalance
        - = newScaledBalance * index
        - = (initialScaledBalance + (quantity/index)) * index
        - = (initialScaledBalance * index) + (quantity/index)*index
    - RHS = existingBalance + quantity
        - = (initialScaledBalance * index) + quantity
- Cancelling, initialScaledBalance * index from LHS and RHS,
- We essentially require, (quantity/index) * index == quantity, with index > 1
- Which will not always be true because solidity uint256 has limited precision and there is a chance of `(quantity/index) * index` to be greater/lesser than `quantity` by 1 wei.



### Understanding flow of tokens through SetToken during **ISSUANCE**


* Let, initial set token supply = s
* And, issue amount **with** **fees** = i


<table>
  <tr>
   <td>
    Default Position units (in 10^18)
   </td>
   <td>
    External Position units (in 10^ 18)
   </td>
   <td>
    Description
   </td>
   <td>
    <strong>BEFORE</strong> issuance
   </td>
   <td>
    In<strong> </strong>resolve equity positions <strong>AFTER </strong>the equity is transferred from the user but <strong>BEFORE</strong> the external position hooks are called (in 10 ^ 18)
   </td>
   <td>
    In resolve equity positions call <strong>AFTER </strong>external position hooks are called (10 ^ 18)
   </td>
   <td>
    In<strong> </strong>resolve Debt Positions <strong>AFTER </strong>the external position hooks are called but <strong>BEFORE</strong> the debt is transferred to the  issuer (in 10 ^ 18)
   </td>
   <td>
    In resolve debt positions <strong>AFTER </strong>the borrowed debt is transferred to the issuer (10 ^ 18)
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
0
   </td>
   <td>
Only default position
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
-500
   </td>
   <td>
Default & negative external. 
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1500
   </td>
   <td>
s * 1000 + i * 1000
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
500
   </td>
   <td>
Default & positive external
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + i * 1500
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
-500, 1000
   </td>
   <td>
Default, positive and negative external
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + i * 2000
   </td>
   <td>
s * 1000 + i * 1000
   </td>
   <td>
s * 1000 + i * 1500
   </td>
   <td>
s * 1000 + i * 1000
   </td>
  </tr>
  <tr>
   <td>
0
   </td>
   <td>
-500
   </td>
   <td>
Only negative external 
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
   <td>
i * 500
   </td>
   <td>
0
   </td>
  </tr>
  <tr>
   <td>
0
   </td>
   <td>
500
   </td>
   <td>
Only positive external
   </td>
   <td>
0
   </td>
   <td>
i * 500
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
  </tr>
  <tr>
   <td>
0
   </td>
   <td>
-500, 1000
   </td>
   <td>
Negative and positive external
   </td>
   <td>
0
   </td>
   <td>
i * 1000
   </td>
   <td>
0
   </td>
   <td>
i * 500
   </td>
   <td>
0
   </td>
  </tr>
</table>


### Understanding flow of tokens through SetToken during **REDEEMING**


* Let, initial set token supply = s
* And, redeem amount **net fees** = r


<table>
  <tr>
   <td>
Default Position unit (in 10^18)
   </td>
   <td>
External Position units (in 10^18)
   </td>
   <td>
Description
   </td>
   <td>
<strong>BEFORE</strong> redeeming (10 ^ 18)
   </td>
   <td>
In<strong> </strong>resolve debt positions <strong>AFTER </strong>the debt is transferred from the user but  <strong>BEFORE</strong> the external position hooks are called (in 10 ^ 18)
   </td>
   <td>
In resolve debt positions <strong>AFTER </strong>external position hooks are called (10 ^ 18)
   </td>
   <td>
In<strong> </strong>resolve  equity positions <strong>AFTER </strong>the external position hooks are called but <strong>BEFORE</strong> the equity is transferred to the user (in 10 ^ 18)
   </td>
   <td>
In resolve equity positions <strong>AFTER </strong>the equity is transferred to the user (10 ^ 18)
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
0
   </td>
   <td>
Only default position
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + r * 0
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 - r * 1000
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
-500
   </td>
   <td>
Default & negative external. 
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + r * 500
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 - r * 1000
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
500
   </td>
   <td>
Default & positive external
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + r * 500
   </td>
   <td>
s * 1000 - r * 1000
   </td>
  </tr>
  <tr>
   <td>
1000
   </td>
   <td>
-500, 1000
   </td>
   <td>
Default, positive and negative external
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + r * 500
   </td>
   <td>
s * 1000
   </td>
   <td>
s * 1000 + r * 1000
   </td>
   <td>
s * 1000 - r * 1000
   </td>
  </tr>
  <tr>
   <td>
0
   </td>
   <td>
-500
   </td>
   <td>
Only negative external 
   </td>
   <td>
0
   </td>
   <td>
r * 500
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
  </tr>
  <tr>
   <td>
0
   </td>
   <td>
500
   </td>
   <td>
Only positive external
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
   <td>
0
   </td>
   <td>
r * 500
   </td>
   <td>
0
   </td>
  </tr>
  <tr>
   <td>
0
   </td>
   <td>
-500, 1000
   </td>
   <td>
Negative and positive external
   </td>
   <td>
0
   </td>
   <td>
r * 500
   </td>
   <td>
0
   </td>
   <td>
r * 1000
   </td>
   <td>
0
   </td>
  </tr>
</table>




#### Note
* All 7 possible combination of position units that a given component can have are covered in the above table.
    * Default: 0, External: n, p, np (3 cases)
    * Default: 1, External:  0, n, p, np (4 cases)
* Multiple negative external positions are the same as a single negative external position because we are summing up all the negative external positions in `DebtIssuanceModule#_getTotalIssuanceUnits`.
* Multiple positive external positions are the same as a single positive external position because we are summing up all the positive external positions in `DebtIssuanceModule#_getTotalIssuanceUnits`.


#### Note (for redeeming): 

* SetToken's are burned at the very beginning of the DebtIssuanceModule#redeem() function, hence in columns 5,6,7,8 (count starting with 1) of the above table, setToken.totalSupply() is `s - r'`, where `r'` is the redeem quantity **without** fees.
* After column 8, fees are minted to the fee receiver and setToken’s total supply becomes `s - r`.

## Feasibility Analysis

### Option 1:

Instead of over-optimizing our checks to check for balances which effectively prevent both undercollaterlization and overcollaterlization. We can just have checks to prevent undercollaterlization. The change is contained within the smart contracts and doesn't bubble up the stack leading to bigger changes in our backend.

The proposed checks for undercollateralization are performed right after the transfer happens, so that we revert immediately and save gas. When token is transferred in, we wish to check the right amount of equity/debt was passed in before we try to transfer it out to external protocols. When token is transferred out, we validate that we are not undercollateralized after the transfer.

### Proposed checks during Issuance
* In `_resolveEquityPositions` we can validate balances after the equity is transferred but before the external position hooks are called, because
* Proposed check:

```javascript
uint256   defaultPositionUnit = setToken.getDefaultPositionRealUnit(component)
cumulativeEquity = defaultPositionUnit
externalPositionModules = setToken.getExternalPositionModules(component)
For each externalPositionModule in externalPositionModules
   externalPositionUnit = setToken.getExternalPositionRealUnit(component, externalPositionModule)
   If (externalPositionUnit > 0)
      cumulativeEquity = cumulativeEquity.add(externalPositionUnit);
newBalance = component.balanceOf(setToken)
require(newBalance > s * defaultPositionUnit + i * cumulativeEquity)
```

* In `_resolveDebtPositions` we validate set collateralization after debt is transferred back to the user.
* Proposed check:

```javascript
defaultPositionUnit = setToken.getDefaultPositionRealUnit(component)
newBalance = component.balanceOf(setToken)
require(newBalance >= (s+i) * defaultPositionUnit)
```


### Proposed checks during Redeeming

* In `_resolveDebtPositions` we can check balances after debt is transferred but before the external position hooks are called.
* Check:

```javascript
defaultPositionUnit = setToken.getDefaultPositionRealUnit(component)
cumulativeDebt = 0
externalPositionModules = setToken.getExternalPositionModules(component)
For each externalPositionModule in externalPositionModules
   externalPositionUnit = setToken.getExternalPositionRealUnit(component, externalPositionModule)
   If (externalPositionUnit &lt; 0)
      cumulativeDebt = cumulativeDebt.add(externalPositionUnit);
newBalance = component.balanceOf(setToken)
require(newBalance >= s * defaultPositionUnit + r *  cumulativeDebt.mul(-1).toUint256())
```

* In `_resolveEquityPositions` we validate collateralization after equity is transferred from the SetToken to the redeemer
* Check:

```javascript
defaultPositionUnit = setToken.getDefaultPositionRealUnit(component)
newBalance = component.balanceOf(setToken)
require(newBalance >= (s-r) * defaultPositionUnit)
```

`NOTE`: Introduction of above checks means a token which charges a fee upon transfer could in theory be used to get any excess amount of that token held by the Set. Excess amount being defined as a remaining amount of wei from a manager action OR any potential tokens accrued to the Set via farming but not yet "absorbed" into a position via the AirdropModule or via syncing positions. Although, this can *NOT* affect other tokens in the Set but only the token that charges the transfer fees (i.e. any other token that has not been absorbed into a position could not be stolen).

`Example`. Consider a SetToken with component A, B & C. Let token A charge fees on transfer. Let default position unit of A be 1000 units (1 unit = 10^18 wei). Let set total supply be 2, then minimum amount of A held in the set is 2 * 1000 units. Now, if there is any extra A held in the Set, which can be due to airdrops or due to A accruing interest. While issuing 1 more SetToken, when we transfer 1000 units from the issuer, let's say 10 units is charged as the transfer fee, thus the effective amount transferred in is 990 units. Now, if the Set balance had been 10 units more than the min balance, i.e. >= 2 * 1000 + 10 units, the extra 10 units is absorbed into the set as position units, and the transaction doesn't revert. Also, note that these actions do not affect other components, B & C in the Set. Because during issuance/redemption we deal with each component individually.

## Open Questions

1. Do we restrict usage of these modified issuance modules (which replace balance checks with collateralization checks) from third-party use?
    - Why did we have the strict checks in the first place?
        - To make sure the set is not undercollateralized 
            - Eg. To stop tokens which collects fees on transfer from being a component in a SetToken
        - This strict check achieved the above requirement, but at the same time it also prevented overcollateralization during issuance/redemption.
    - What are the only requirements for our system to function well?
        - We do not want Set to be undercollateralized
        - We are fine with overcollateralization (discussed below)
    - Thus, the new checks fulfill our requirements.
        - So, we need not restrict usage of these modified issuance modules
    
2. Are we fine with Set being overcollaterlized?
   * Yes, SetToken can resync the extra assets present to increase their default positions with help of modules
      * ALM and CLM can resync positions for leveraged tokens
      * Airdrop module can add extra assets as positions for other SetTokens
   * Also, issuance/redemption generally would not lead to overcollaterlization of more than 1 wei
      * Over-collateralization of 1 wei can be introduced to rounding errors
      * For over collateralization greater than 1 wei, we would need the special case of a token which rewards holders on a transfer
      * We can handle that case using Airdrop module

3. Do we restrict these new modified issuance modules (which replace balance checks with collateralization checks) only to SetToken which hold aTokens? Or do we make these issuance modules the default going forward?
    -  Current approach is leaning towards not restricting module usage but providing it as a potential different offering for manager's to choose when selecting the issuance module they want to use
    - They can choose the default issuance modules with stricter checks if they are sure of not using any aToken or tokens which take fees on transfer as components in their SetToken
    - But if they plan to use aTokens or tokens which take fees on transfer as SetToken components, then they can use the new issuance modules


## Checkpoint 1

**Reviewer**: