# **STIP - Aave Leverage Module**

**_Using template v0.1_**


## Abstract

We are currently in the midst of a bull market. Demand for leverage is extremely high, and we want to continue building products to satisfy this demand. Currently, our only leverage product in the market. ETH2x-FLI which has already exceeded our expectations and is our most profitable product. We will likely be launching more FLI products on Compound based on this initial success. These include BTC 2x, ETH -1x, BTC -1x, UNI 2x, ETH/BTC Ratio 2x etc. However, Compound does not support other high volume ERC20 tokens such as LINK, YFI, DPI etc. 


## Motivation


#### Feature

Add AaveLeverageModule to enable launching products based on Aave.


#### Why is this feature necessary?



1. To launch the LINK 2x which is only available on Aave. LINK has over $1B in liquidity and is the most traded ERC20 token higher than UNI, MATIC etc.
2. To enable third party Set managers to margin trade using Aave.
3. To deploy on Matic L2 and enable MATIC2x token


## Background Information


#### Set System / Previous Work



*   CompoundLeverageModule
    *   Sync positions in module hooks
    *   Entering markets to enable collateral
    *   Lever interface: borrow -> trade -> mint
    *   Delever interface: redeem -> trade -> repay
*   Compound External Library
    *   An external library that houses all the Compound external interactions


#### Aave Research



*   [Docs](https://docs.aave.com/developers/the-core-protocol/lendingpool/ilendingpool)
*   LendingPool
    *   LendingPool address should be fetched from the LendingPoolAddressesProvider as contract addresses could change in Aave
    *   All deposit and withdraw occur through the LendingPool contract
    *   Need ability to sync in AaveLeverageModule
    *   WETH is supported natively in Aave unlike Compound
    *   Tokens are approved to LendingPool
    *   getReserveData() returns aToken and debtToken addresses
    *   setUserUseReserveAsCollateral() is similar to Compound enterMarkets to use as collateral
*   AToken
    *   BalanceOf returns latest collateral balance
    *   Tokens are approved to LendingPool
    *   1 aToken = 1 underlying. As balances change every protocol interaction, we can just call balanceOf to get latest collateral balance
*   VariableDebtToken
    *   Debt positions are tokenized so can call balanceOf() to always return the most up to date accumulated debt of the user.
    *   [Variable debt USDC](https://etherscan.io/token/0x619beb58998eD2278e08620f97007e1116D5D25b#balances)


#### Resources



*   [Aave docs](https://docs.aave.com/developers/the-core-protocol/lendingpool/ilendingpool)
*   [Aave flash loans](https://docs.aave.com/developers/guides/flash-loans)
*   [Github](https://github.com/aave/protocol-v2/tree/ice/mainnet-deployment-03-12-2020/contracts/protocol)
*   [Aave addresses](https://docs.aave.com/developers/deployed-contracts/deployed-contracts)
*   [PRD](https://docs.google.com/document/d/1A_-MfSUm7C7dU5YqhH4NtUK9L6Jz7OBV5rSpxOE_bn8/edit#)
*   [Presentation](https://docs.google.com/presentation/d/12dhVzlVXFLOVV_hipoxOoLloTcr_pfLird54qKBgYoM/edit#slide=id.g38420732af_0_142)


## Open Questions



*   Allow a Set to have multiple leverage positions on Aave (similar to CLM)? I.e. leverage ETH with USDC as collateral and leverage WBTC with DAI as collateral in the same SetToken
*   How do we handle stAAVE LM rewards?
    *   Can use a similar approach as Comp Reinvestment adapter
    *   Difference between both approaches is AAVE rewards stAAVE which would need to be burned after a cooldown period (10 days) to redeem AAVE tokens
    *   Can not directly trade stAAVE tokens to collateral (to reinvest) due to absence of liquidity on DEXes
*   Do we incorporate Aave flash loans directly into the AaveLeverageModule?
    *   No, max LTV is not as big of a limiting factor as max trade size, punt on flashloans for now
*   How does ETH support work in AAVE?
    *   ETH is not natively supported in Aave. ETH should be wrapped into WETH before interacting with the protocol. 
    *   [Aave WETH Gateway](https://docs.aave.com/developers/the-core-protocol/weth-gateway)
*   Should the AaveLeverageModule work for recursive leverage?
*   When does sync() get called?
*   Do we want to have a protocol fee that we can turn on for trade() the same as TradeModule?
    *   Yes


## Feasibility Analysis

Option 1: AaveLeverageModule (similar pattern to CompoundLeverageModule)



*   Allow for multiple positions and potentially open it to 3rd party managers

Option 2: AaveLeverageTokenModule



*   Allow only 1 default and 1 debt position to simplify code


## Timeline

|  Action               |  Start Date   |  End Date  |
|---                    |---            |---         |
| Kickoff               |               |            |
| Feasibility Analysis  |               |            |
| Technical Spec        |               |            |
| Implementation        |               |            |
| Auditors              |               |            |
| Launch                |               |            |
