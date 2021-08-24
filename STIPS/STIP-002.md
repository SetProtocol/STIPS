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



## Feasibility Analysis

### Option 1:

Instead of over-optimizing our checks to check for balances which effectively prevent both undercollaterlization and overcollaterlization. We can just have checks to prevent undercollaterlization. 
Pseudocode:
```javascript
defaultPositionUnit = setToken.getDefaultPositionRealUnit(component)
newBalance = component.balanceOf(setToken)
require(newBalance >= (s-r) * defaultPositionUnit)
```

- We have reduced the strictness in the above checks and replaced balance checks with checks to prevent undercollateralization.
- The change is contained within the smart contracts and doesn't bubble up the stack leading to bigger changes in our backend.

`NOTE`: Introduction of above checks means a token which charges a fee upon transfer could in theory be used to get any excess amount of that token held by the Set. Excess amount being defined as a remaining amount of wei from a manager action OR any potential tokens accrued to the Set via farming but not yet "absorbed" into a position via the AirdropModule. Although, this can NOT affect other tokens in the Set but only the token that charges the transfer fees (i.e. any other token that has not been absorbed into a position could not be stolen).

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
    - Yes, SetToken can resync the extra assets present to increase their default positions with help of modules
        - ALM and CLM can resync positions for leveraged tokens
        - Airdrop module can add extra assets as positions for other SetTokens
    - Also, issuance/redemption generally would not lead to overcollaterlization
        - For it to happen, we would need the special case of a token which rewards holders on a transfer
        - we can handle that case using Airdrop module

3. Do we restrict these new modified issuance modules (which replace balance checks with collateralization checks) only to SetToken which hold aTokens? Or do we make these issuance modules the default going forward?
    -  Current approach is leaning towards not restricting module usage but providing it as a potential different offering for manager's to choose when selecting the issuance module they want to use
    - They can choose the default issuance modules with stricter checks if they are sure of not using any aToken or tokens which take fees on transfer as components in their SetToken
    - But if they plan to use aTokens or tokens which take fees on transfer as SetToken components, then they can use the new issuance modules


## Checkpoint 1

**Reviewer**: