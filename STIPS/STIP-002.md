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
- In _DebtIssuanceModule#redeem_ and _BasicIssuanceModule#redeem_, we transfer the component from the SetToken to the redeemer using _Invoke#strictInvokeTransfer_ library function which internally calls _ExplicitERC20#transferFrom_ function, which performs the above mentioned check.

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

## Feasibility Analysis

### Option 1:
Instead of over-optimizing our checks to test for both undercollaterlization and overcollaterlization. We can just have checks to prevent undercollaterlization. This would prevent reverting in cases when `newBalance` of aToken is greater than `existingBalance + quantity` by 1 wei, and thus decrease the failure rate of issuance/redemption (maybe to 33%, cause we can consider less than, equal to, and greater than to be 3 equal cases). This option is the best considering time. The change is contained within the smart contracts and doesn't bubble up the stack leading to bigger changes in our backend.



## Open Questions

1. Do we restrict usage of these modified issuance modules (which are linked to the new libraries) from third-party use?
    - Why did we have the strict checks in the first place?
        - To make sure the set is not undercollateralized 
            - Eg. To stop tokens which collects fees on transfer from being a component in a SetToken
        - This strict check achieved the above requirement, but at the same time it also prevented overcollateralization
    - What are the only requirements for our system to function well?
        - We do not want Set to be undercollateralized
        - We are fine with overcollaterlization (discussed below)
    - Thus, the new checks full fill our requirements.
        - So, we need not restrict usage of these modified issuance modules

2. Do we restrict these new modified issuance modules (which are linked to the new libraries) only to SetToken which hold aTokens? Or do we make these issuance modules the default going forward?
    
2. Are we fine with Set being overcollaterlized?
    - Yes, SetToken can resync the extra assets present to increase their default positions with help of modules
        - ALM and CLM can resync positions for leveraged tokens
        - Airdrop module can add extra assets as positions for other SetTokens
    - Also, issuance/redemption generally would not lead to overcollaterlization
        - For it to happen, we would need the special case of a token which rewards holders on a transfer
        - we can handle that case using Airdrop module

3. How to handle `newBalance < existingBalance.add(_quantity))`?
    - This is a possible case for aTokens, since `(quantity/index) * index` can be less than `quantity`.
    - We have seen this case once during testing.


## Checkpoint 1

**Reviewer**:

## Proposed Architecture Changes

Pseudocode of option1 :
```javascript

library Invoke {
    ... // other functions from Invoke library
    function strictInvokeTransfer(ISetToken _setToken, address _token, address _to, uint256 _quantity) internal {
        ../// existing implementations
    }
    function invokeTransferWithCollaterlizationChecks(ISetToken _setToken, address _token, address _to, uint256 _quantity) internal {
        if (_quantity > 0) {
            // Retrieve current balance of token for the SetToken
            uint256 existingBalance = IERC20(_token).balanceOf(address(_setToken));

            Invoke.invokeTransfer(_setToken, _token, _to, _quantity);

            // Get new balance of transferred token for SetToken
            uint256 newBalance = IERC20(_token).balanceOf(address(_setToken));

            // prevent undercollateralization
            require(newBalance >= existingBalance.sub(_quantity)), "Invalid post transfer balance");
        }
    }
}


// Here we can either add a new library with a new transferFrom function
// or add a new function in the ExplicitERC20 library.
library ExplicitERC20V2 {
    function transferFromWithCollaterlizationChecks(IERC20 _token, address _from, address _to, uint256 _quantity) internal {
        if (_quantity > 0) {
            uint256 existingBalance = _token.balanceOf(_to);
            SafeERC20.safeTransferFrom(_token, _from, _to, _quantity);
            uint256 newBalance = _token.balanceOf(_to);

            // prevent undercollateralization
            require(
                newBalance >= existingBalance.add(_quantity)), "Invalid post transfer balance"
            );
        }
    }
}

// Since we use ModuleBase#transferFrom in our issuance modules we would also need to add a new function in the ModuleBase contract which uses the new ExplicitERC20#transferFromWithCollaterlizationChecks
// Or we can skip calling ModuleBase#transferFrom in our new issuance modules and directly call ExplicitERC20#transferFromWithCollaterlizationChecks
abstract contract ModuleBase is IModule {
    ...//other existing functions
    function transferFromWithCollaterlizationChecks(IERC20 _token, address _from, address _to, uint256 _quantity) internal {
        ExplicitERC20.transferFromWithCollaterlizationChecks(_token, _from, _to, _quantity);
    }
}
```
Next, when deploying the new issuance modules we link the modified new libraries to them.


## Checkpoint 2

**Reviewer**: