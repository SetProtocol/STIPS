# STIP-005
*Using template v0.1*
## Abstract

This STIP explores possible designs for a PerpetualProtocol module.

## Motivation

We’re interested in perpetual futures protocols as a vehicle for designing:

* FLI products for a wider range of base tokens
* FLI products with larger leverage multiples (ex: BTC-FLI3X)
* Inverse FLI products
* Basis trading products, e.g for products that buy assets in the spot market and sell futures contracts for them, capturing the difference in price between these as revenue.

## Background

### Perpetual Contracts

A perpetual futures contract is an agreement to buy or sell an asset at some point in the future. Perpetual futures are cash-settled and are held indefinitely without the need to roll over contracts. Payments are periodically exchanged between holders of the two sides of the contracts, long and short, with the direction and magnitude of the settlement based on the difference between the contract price and that of the underlying asset, as well as  the difference in leverage between the two sides.

Cryptocurrency perpetuals are characterised by the availability of high leverage, sometimes over 10 times the margin, and by the use of auto-deleveraging, which compels high-leverage, profitable traders to forfeit a portion of their profits to cover the losses of the other side during periods of high market volatility, as well as insurance funds, pools of assets intended to prevent the need for auto-deleveraging.

Perpetual futures for the value of an asset require the payment of a periodic settlement, intended to mirror the value of the flow from one side of the contract to the other. At any time *t*, the dividend paid from shorts to longs, is defined as:

![eq](https://user-images.githubusercontent.com/7332026/134842980-5f8932fd-c402-4a07-bc46-2cb751d23176.png)

where ![eq2](https://user-images.githubusercontent.com/7332026/134842985-b5e3ef3b-2826-4570-97c7-0baefe08f218.png) is the price of the perpetual at time _t_, ![eq3](https://user-images.githubusercontent.com/7332026/134842987-7761f992-10e9-4c3e-955e-b94fb150ec4c.png) is the dividend paid to owners of the underlying asset at time _t_, and ![eq4](https://user-images.githubusercontent.com/7332026/134842989-4f0305b5-6f24-43bd-8c6e-4f92a36a0e5b.png)is the return on an alternative asset (expected to be a short-term, low-risk rate) between time _t_ and _t+1_.

In summary, the difference in crypto prices from one moment to another can be thought of as the dividend due to owners of the asset. Settlement between different sides of the trade, known as **funding**, typically occurs every eight hours, although other arrangements including per-block settlement are possible. In addition, the base interest on cryptocurrency perpetuals (the low-risk rate) is usually a fixed percentage set by the exchange.

### Liquidation and Insurance Funds

Contract liquidation in response to adverse price movement can occur when the contract's (or account’s) remaining margin reaches a pre-specified value (the maintenance margin, which determines the liquidation price). The liquidation proceeds are typically inserted into the exchange's insurance fund, which guarantees the profitable side when counterparty margin is insufficient, usually when the price of the asset has moved sharply in one direction and the exchange was unable to liquidate at a profit.


### Auto-Deleveraging

If the exchange's insurance fund is depleted, whether globally or for a particular contract, accounts are ranked according to their profit and leverage. This is used to form an "auto-deleveraging" queue, where the positions of traders at the front of the queue are closed, at the bankruptcy price, to prevent market losers from going into default. Auto-deleveraging is intended to reduce counterparty risk by penalising the riskiest traders, as opposed to the more evenly-spread "mutualised" model used by central clearing.

[(Source: Wiki)](https://en.wikipedia.org/wiki/Perpetual_futures)

[(Source: Perpetuals V2 (Curie) AMA)](https://medium.com/perpetual-protocol/perpetual-protocol-ama-the-curie-release-b8ecc0ad2fb4)


### Detailed Case Study

Belows we explore how the features above are implemented in Perpetual's V2 protocol, presenting

* a high level description of their protocol
* an overview of their trading API
* an analysis of how positions in the protocol could be represented on the SetToken, integrated into issuance/redemption flows, and mapped onto the existing leverage module design to enable products like FLIs
* a suite of simple scenario cases which shows the real behavior of the system (see appendix)

**[Perpetual Protocol V2](https://github.com/perpetual-protocol)**

Perpetual Protocol virtualizes assets on an AMM and allows users who have deposited USDC into the system to act as takers or makers for them.

* **Takers** open long or short positions and can redeem these at a profit if the price for a virtual asset _on the AMM_ moves in their favor. Takers can trade on margin, allowing them to lever their positions.
* **Makers** provide liquidity at or around the market price and receive fees in return.

#### Uniswap Pricing Engine

PerpV2 uses UniswapV3 as its pricing engine, minting virtual tokens to pools in response to liquidity provision events and relying on pool dynamics to determine settlement payments as positions are closed.

There is no secondary market for PERP’s virtual asset tokens; they are wholly controlled by the protocol and used for internal accounting to help with price binding.

The realizable value of a position is the sum of what it can be swapped for on the AMM and whatever funding flows (positive or negative) it accrues at the time of settlement.

#### Oracle Valuation Engine

Oracles (for spot prices) are used to:

* calculate funding payments
* value account positions for the purposes of margin calculations.

#### Trading

Traders interact with the protocol through the [ClearingHouse][1] contract which provides an API to open and close positions. The interface follows the Uniswap pattern for swaps and _openPosition_ can be configured to

[1]: https://github.com/perpetual-protocol/perp-lushan/blob/main/contracts/ClearingHouse.sol

* open a long position
* open a short position
* reduce the size of any position

Uniswap V3 returns_ token1/token0_  prices where _token0_ and _token1_ are calculated by comparing assets _(token0 < token1_). PERP fixes the order of base (synthetic) and quote (collateral) tokens as  **B0Q1** (base=token0, quote=token1). At the moment the system supports a single collateral token (USDC). Account and position values are denominated in USDC and all positions close to USDC.

For example, to open a long vETH position worth 1 USDC, the trader would configure openPosition as below

```ts
IClearingHouse.OpenPositionParams params = IClearingHouse.OpenPositionParams({
   baseToken: VETH_ADDR,
   isBaseToQuote: false, // false for longing
   isExactInput: true,
   amount: parseEther("1"),
   sqrtPriceLimitX96: 0 // 0 for no price limitation
   ...
})
```

The trader could sell their entire vETH position by inverting the _isBaseToQuote_ value as below

```ts
const basePositionSize = await ch.getPositionSize(_taker.address, VETH_ADDRESS);

IClearingHouse.OpenPositionParams params = IClearingHouse.OpenPositionParams({
   baseToken: VETH_ADDR,
   isBaseToQuote: true, // true for shorting
   isExactInput: true,
   amount: basePositionSize,
   sqrtPriceLimitX96: 0 // 0 for no price limitation
   ...
})
```

#### Margin/Leverage

PERP provides accounts with an initial margin ratio of 10% and sets their minimum margin ratio at 6.25%. Margin levels are evaluated as a sum of an account’s positions - e.g the system is cross-margin by default. In practice, traders can open positions whose spot market valuation is  ~9X the value of collateral they’ve deposited into the system. For margin requirement purposes, positions are priced using Chainlink oracles for the parent asset of a virtual asset.

As the oracle price for a parent asset changes, the total margin requirement for an account scales with it such that the leverage ratio for a position maintains a constant relation to the position’s value in sport markets. (See  Appendix/Simulations/Leverage for an example).

#### Funding

Funding is a payment flow between makers and takers in the Perp system that acts as a balancing force to bring the AMM price of the virtual asset closer to the oracle price.

A simplified formula for how these payments are calculated is:

```
   time = seconds since position opened / 86400
   pendingFundingPayment = _positionSize * (AMM market price - Oracle price) *  time
```

If a taker opens a long position and the oracle price is below AMM market, they pay funding fees. If the oracle price is above AMM market price, they receive them. Funding payments are settled when the trader closes a position, realizing their PNL. They are calculated using the  AMM price and the 15 min TWAP oracle price read during settlement.

Traders settle their pending funding payments whenever they open, modify or close a position. For the module's purposes this means funding is credited or debited during issuance, redemption, lever, and delever.

#### Insurance Fund

This is currently WIP with no unit tests in the perp-lushan repository, as of commit: 56ab296 (Sept 15)

## Integration with Set Protocol / Feasibility Analysis

**Fundamental Concepts**

* Issuance replicates the current position (e.g its leverage and ratio of components).  USDC enters our system and we allocate resources to preserve the sets existing properties as we open positions on the external protocol.
* Redemption is about replicating the current position as USDC leaves our system and we close positions on the external protocol.

**Design Goals**

* Works seamlessly with the existing DIM and SetToken
* Works well with rest of the SetProtocol architecture and processes for managing Sets
* Encapsulates all PerpV2 interactions in a dedicated module

**Summary**

We think PERP accounts should be modeled on the SetToken as an external USDC position and that PERP account virtual positions should be modeled as default positions on a virtual SetToken. This virtual SetToken is intended solely as a data structure which tracks position quantities within the PERP system. Using it lets us manage PERP accounts as a conventional SetToken component while preserving the ability to map PERP virtual positions onto SetProtocol’s existing logic for operations like replication and rebalancing.

For the purposes of issuance and redemption the size of a PERP external position is the realizable value of all of the account’s virtual positions and funding dividends, i.e the theoretical amount that can be withdrawn from the Perp system when an account’s positions are closed and settled in current PERP markets (excluding slippage). This value is synthesized from quotes for virtual assets from the AMM and Perp's internal pending fees accounting.

One drawback of this design is that we will need to  provide a new API for viewing PERP position details since current APIs for the SetToken would only register these as single aggregate USDC value.

**Required Contracts**

<table>
  <tr>
   <td><strong>Contract</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>SetToken
   </td>
   <td>Has an external USDC position managed by PerpV2 protocol.
   </td>
  </tr>
  <tr>
   <td>DebtIssuanceModule
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td valign="top">PerpV2LeverageModule
   </td>
   <td>
       <p>Exposes a standard leverage module API </p>
        <ul>
            <li>lever </li>
            <li>delever </li>
            <li>sync</li>
            <li>hooks </li>
       </ul>
       <p> <strong>New</strong> </p>
       <p> For each Set Token it implements a virtual SetToken that tracks the virtual asset positions of a PerpV2 account. This token allows us to use all our existing infrastructure and strategies for handling positions within Perps closed system. Virtual assets are added as default positions on the virtual SetToken. </p>
   </td>
  </tr>
  <tr>
   <td> Perp.Vault
   </td>
   <td> (External) Entry point to the Perp system, exposes deposit and withdraw methods
   </td>
  </tr>
  <tr>
   <td> PerpV2.ClearingHouse
   </td>
   <td> (External) Exposes Perp’s trading API
   </td>
  </tr>
</table>

### Similarities between the ALM/CLM flow and Perp flow

In general, Perp simplifies the lever and delver flows. Most of the actions required to  get into a leveraged position in ALM and CLM  are accomplished by opening positions on margin in the external protocol (e.g via _CH.openPosition_ in case of perp v2).

<table>
  <tr>
  <td><strong>Step</strong></td>
    <td><strong>ALM/CLM flow</strong></td>
    <td><strong>Perp flow</strong></td>
  </tr>
  <tr>
    <td valign="top"> <strong>Lever</strong></td>
          <td valign="top">
      <p>
    Use existing collateral to Borrow debt asset, trade it for collateral asset and deposit that back
          </p>
        </td>
      <td>
    <p>Open a position using <em>ClearingHouse.openPosition()</em></p>
        <p>
          Under the hood the perp V2 system does something similar to what our existing leverage modules do:
    <ul>
          <li>Use vault’s USDC as collateral to mint vUSDC </li>
          <li>Swap vUSDC for vETH </li>
          <li>Clearing house receives vETH and manages it on SetToken’s behalf </li>
    </ul>
        </p>
  <p> Amount of vUSDC minted depends on our leverage ratio. </p>
      <p> We can specify the exact amount of vUSDC we wish to send or the amount of vETH we wish to receive in the CH.openPosition() function. It is similar to how we do it in UniswapV2. </p>
   </td>
  </tr>
  <tr>
  <td valign="top"><strong>Delever</strong></td>
  <td valign="top">
    <p> Trade collateral asset for borrow asset and repay borrow asset </p>
    </td>
    <td>
    <p> Reduce a position using <em>CH.openPosition()</em>by inverting the value for <em>isBaseToQuote</em> used when increasing the position. </p>
          <p> Under the hood the perp V2 system does something similar to what our existing leverage modules do
          <ul>
      <li>Clearing house swaps vETH for vUSDC </li>
      <li>PnL is calculated, <em>PnL = vUSDC received from AMM - cost basis - owedRealizedPnL</em> </li>
      <li>Vault returns collateral + PnL </li>
    </ul>
          <p> (Cost basis = amount of SetTokens minted during lever) </p>
    </td>
  </tr>
</table>


### Differences between the ALM/CLM flow and Perp flow

Sync, issuance and redemption are fundamentally different and more complex.

<table>
  <tr>
     <td>Property</td>
     <td>ALM/CLM</td>
     <td>PerpV2</td>
  </tr>
  <tr>
      <td valign="top"><strong>Pricing Engine</strong></td>
      <td valign="top">
          <p>Open AMM </p>
          <p> Price always remained very close to oracle price. </p>
      </td>
      <td valign="top">
          <p>Closed AMM</p>
          <p> Since the pool is a futures market, indirectly anchored to the spot price by a funding mechanism, its quotes can deviate a lot from the asset prices we want to track. </p>
          <p> This  forces us to calculate our realizable value during syncing by using prices from the AMM and not just our Chainlink oracle. </p>
      </td>
  </tr>
  <tr>
      <td valign="top"> Sync </td>
      <td valign="top">
          <ul>
              <li> Fetch data from external protocol </li>
              <li> Update position units on SetToken based on fetched data </li>
          </ul>
      </td>
      <td valign="top">
        <ul>
            <li>Fetch data from external protocol</li>
            <li>Update position units on virtual SetToken based on fetched data if necessary</li>
            <li>Update external USDC position unit on real SetToken based on simulation of Perp account's realizable value </li>
        </ul>
      </td>
  </tr>
  <tr>
      <td valign="top">Issuance</td>
      <td valign="top">
        <p> The issuer acquires the asset to which he wants leveraged exposure and deposits that in our system. The amount of asset deposited is equal to the amount of exposure the issuer needs. </p>
        <p> After deposit, we return the issuer the minted SetToken along with a different asset (USDC) with returned amount = (deposited amount)/(leverage ratio). </p>
        </p> There is no AMM involved in this process. (Although it might have been used to acquire the initial asset) </p>
      </td>
      <td valign="top">
        <p> The issuer deposits only the collateral asset (USDC) to our system and gets exposure to any asset that is supported by the external protocol. </p>
        <p> There is an AMM involved in this process. Albeit a closed and lagging one. </p>
      </td>
  </tr>
  <tr>
    <td valign="top">Redemption</td>
    <td valign="top">
        <p> The redeemer sends the debt asset and SetToken to be redeemed. We burn the SetToken and unwind the leverage position. We return to the redeemer the amount of the leveraged asset that he had exposure to. </p>
        <p> There is no AMM involved in this process. (Although it might have been used to acquire the initial asset)
   </td>
   <td valign="top">
     <p> The redeemer brings only the SetToken. While we only return to the redeemer the USDC value of the amount of asset that he had exposure to. </p>
     <p> There is an AMM involved in this process. </p>
   </td>
  </tr>
  <tr>
    <td valign="top">Frequency of rebalances</td>
    <td valign="top">
        <p> In CLM/ALM, since our position was a CDP, we had to actively manage our leverage ratios, keep track of them and call lever/delever every 24 hours. </p>
   </td>
   <td valign="top">
    <p> Leveraged Perp positions are subject to funding flows which may move the Set substantially off its leverage target (negatively or positively). They're also exposed to liquidation risks when account values drop below minimum requirements. </p>
    <p> The module will need to support active maintenance of the leverage target although it's not immediately clear what the rebalance cadence might be since this depends on actual market behavior. </p>
   </td>
  </tr>
</table>

### PerpV2 issuance and redemption flow analysis

#### Issuance

<table>
  <tr>
   <td width="20%"><strong>Issuance Step</strong>
   </td>
   <td><strong>Action</strong>
   </td>
  </tr>
  <tr>
   <td>Initial call flow
   </td>
   <td>
<ul>

<li>User calls <strong><em>issue(minQuantity) </em></strong>on DIM

<li><em>DIM</em> calls <em>PerpV2Module</em> issuance hook

<li><em>PerpV2Module</em> calls <strong><em>sync()</em></strong> inside the module issuance hook
</li>
</ul>
   </td>
  </tr>
  <tr>
   <td><em>sync() </em>
<p>
(flow summary)
   </td>
   <td>
  <ul>
  <li>Reads the virtual token balances of the virtual SetToken from PerpV2Module</li>
  <li>Updates the default position units  on the virtual SetToken </li>
    <ul>
      <li>Only done if the new unit calculation does not equal existing unit </li>
      <li>Only necessary if a liquidation has occurred. </li>
      <li>Interest accrual is not an issue here </li>
    </ul>
  <li>Updates external USDC position on the real SetToken </li>
  </ul>
   </td>
  </tr>
<tr>
   <td>sync()
  <p>(updating virtual default positions if necessary)
   </td>
   <td><em>newUnitForVirtualAsset = vSetToken.balanceOf(vAsset) / realSetToken.totalSupply()</em>
   </td>
  </tr>
  <tr>
   <td>sync()
  <p>(Update external USDC position on the real SetToken)
   </td>
   <td>
     <p> First determine the additional amount of virtual assets we need exposure to: </p>
     <p> &nbsp;&nbsp;<em> vAssetAmount = mintQuantity * vSetToken.getDefaultPositionRealUnit(vAsset) </em> </p>
     <p> Calculate the amount of USDC required to buy vAssetAmount by simulating a trade on Perp’s vAMM. The delta quote returned by the simulation will include realized funding payments and fees. These will be distributed across the real SetToken's external USDC position. </p>
     <p> &nbsp;&nbsp;<em> deltaQuote = getQuoteForVAsset(vAsset, vAssetAmount, direction=long | short)</em> </p>
     <p> Update the real SetToken’s external position for USDC as below. (For a SetToken with multiple positions in multiple virtual assets we would have to sum up all delta) </p>
     <p> &nbsp;&nbsp;<em> newTotalSupply = realSetToken.totalSupply + mintQuantity </em> </p>
     <p> &nbsp;&nbsp;<em> newUSDCTotal = externalUSDCPositionUnit * realSetToken.totalSupply + deltaQuote </em> </p>
           <p> &nbsp;&nbsp;<em> newExternalPositionUnit = newUSDCTotal / newTotalSupply </em> </p>
   </td>
  </tr>
  <tr>
   <td>DIM: transfer USDC
   </td>
   <td>
    <p>DIM reads external USDC position updated during <em>sync()</em>  from real setToken and transfers in USDC amount </p>
    <p> <em> &nbsp;&nbsp; usdcAmount = mintQuantity * external position on real SetToken</em> </p>
   </td>
  </tr>
  <tr>
   <td>Component IssuanceHook flow
   </td>
   <td>First, Invoke SetToken to deposit the received USDC from issuer to PERP vault
<p>
Then, iterate on default positions of virtual SetToken, opening positions
   <p> &nbsp;&nbsp;<em>for asset[i], positionUnit[i] of virtual SetToken</em></p>
     <p> &nbsp;&nbsp;&nbsp;&nbsp;<em>  tradeAmount = positionUnit[i] * mintQuantity</em> </p>
     <p> &nbsp;&nbsp;&nbsp;&nbsp;<em>  ClearingHouse.openPosition(asset[i], tradeAmount)</em> </p>
   </td>
  </tr>
</table>



#### Redemption

<table>
  <tr>
   <td width="20%"><strong>Redemption Step</strong>
   </td>
   <td><strong>Action</strong>
   </td>
  </tr>
  <tr>
   <td>Initial call flow
   </td>
   <td>
<ul>

<li>User calls <em>redeem(redeemQuantity)</em> on DIM

<li>DIM calls PerpV2Module redeem hook

<li>PerpV2Module calls <em>sync</em> inside the component redeem hook
</li>
</ul>
   </td>
  </tr>
  <tr>
   <td><em>sync() </em>
<p>
(flow summary)
   </td>
   <td>
<ul>
  <li>Reads the virtual token balances of the virtual SetToken from <em>PerpV2Module</em> </li>
  <li>Updates the default position units  on the virtual SetToken </li>
    <ul>
      <li>Only done if the new unit calculation does not equal existing unit </li>
      <li>Only necessary here if a liquidation has occurred. </li>
      <li>Interest accrual is not an issue here</li>
    </ul>
  </li>
  <li>Updates external USDC position on the real SetToken </li>
</ul>
   </td>
  </tr>
  <tr>
   <td>sync()
<p>
(updating virtual default positions if necessary)
   </td>
   <td><em>newUnitForVirtualAsset = vSetToken.balanceOf(vAsset) / realSetToken.totalSupply()</em>
   </td>
  </tr>
  <tr>
   <td>sync()
<p>
(Update external USDC position on the real SetToken)
   </td>
   <td>
     <p>First determine the amount of virtual assets we need to reduce exposure to:</p>
     <p><em>&nbsp;&nbsp;vAssetAmount = redeemQuantity * vSetToken.getDefaultPositionRealUnit(vAsset)</em></p>
     <p>Calculate the amount of USDC we would receive by simulating closing a position on the external protocol. The delta quote returned by the simulation will include realized funding payments and fees. These will be distributed across the real SetToken's external USDC position. </p>
  <p>&nbsp;&nbsp;<em>deltaQuote = getQuoteForVAsset(vAsset, vAssetAmount, direction=long | short)</em></p>
  <p> Update the real SetToken’s external position for USDC as below. (For a SetToken with multiple positions in multiple virtual assets we would have to sum up all delta)<p>
  <p> &nbsp;&nbsp;<em>newTotalSupply = realSetToken.totalSupply - redeemQuantity</em> </p>
  <p> &nbsp;&nbsp;<em>newUSDCTotal = externalUSDCPositionUnit * realSetToken.totalSupply - deltaQuote</em></p>
  <p> &nbsp;&nbsp;<em>newExternalPositionUnit = newUSDCTotal / newTotalSupply</em></p>
   </td>
  </tr>
  <tr>
   <td>Component RedeemHook flow
   </td>
   <td>
     <p>First, iterate on default positions of virtual SetToken</p>
     <p>&nbsp;&nbsp;<em>for asset[i], positionUnit[i] of virtual SetToken</em> </p>
     <p><em> &nbsp;&nbsp;&nbsp;&nbsp;tradeAmount = positionUnit[i] * redeemQuantity</em></p>
     <p><em> &nbsp;&nbsp;&nbsp;&nbsp;ClearingHouse.openPositon(asset[i], tradeAmount)</em></p>
     <p></p>
     <p>Then, invoke <em>PerpVault.withdraw(deltaQuote)</em> to withdraw USDC from vault and deposit in the SetToken </p>
   </td>
  </tr>
  <tr>
   <td>DIM: transfer USDC
   </td>
   <td>
     <p>DIM reads external usdc position from real SetToken and transfers <em>quantity</em> amount of USDC to redeemer</p>
     <p>&nbsp;&nbsp;<em>quantity = redeemQuantity * external position unit on real SetToken</em>
   </td>
  </tr>
</table>

## Open Questions

* Sachin: how is owedRealizedPNL used in the protocol?
* Brian: how do we prevent sandwich attacks?
* Richard: how feasible is it for us to integrate perp v2 [maker / limit orders](https://cryptobriefing.com/earn-fees-for-trading-on-uniswap-v3-how-to-make-limit-orders/)? Is it  the same flows being a maker as taker, for Set accounting and issuance purposes?

* **Brian in slack**: one thing we need to think through and consider is the impact having to execute an actual trade in the vAMM may have on issuance. I think we know it'll probably alter the leverage ratio a little since there will be some slippage but what if someone makes a big trade? Can they throw it all out of whack? **Could they sandwich attack us through issuance to drain the pool**?
    * Speed bump ideas:
        * Minting limits
        * Issuance fees
        * Gas based throttling
        * ~~EOA only (not viable since contracts are the rebalancers)~~
        *
* Why does free collateral decline as the Oracle price moves up, independent of any other change?
* Total unrealized PNL seems highly decoupled from realizable values. To value our position at any moment, is synthesizing the owedRealizedPNL the right approach / What is the best way to get a trade quote? (Asked in Telegram) 
  + > (Shao Kang-Lee in Telegram): "since uniV3 can't calculate something like getAmountOut before, so we learn from their Periphery repo's - we can call our [Quoter.swap][300] to get the return amount, then calculate the pnl based on market price with slippage. However, we don't recommend to use it onchain because [it's very gas intensive]"
* What is an[ “index price spread attack][301]”
  + > (Shao Kang-Lee in Telegram) "..this is being solved": Perp team design discussion here: [index price spread attack][303].
  + > Summary: this is a potential Perp Protocol vulnerability in which someone could open a long position with zero collateral 
      under certain conditions. (Currently precluded by the "conservative" formula they use to calculate margin requirements) 
* What is a “[bad debt attack][302]” ? 
  + > (Shao Kang-Lee in Telegram) "..this is being solved": Perp team design discussion here: [bad debt attack][304]
  + > Summary: The attacker takes a large short position, driving AMM prices lower such that underwater longs can be liquidated. 
      Long positions are then closed by force, the market drops and the attacker closes their short position at a profit. 
* What exposure to flash loans does the funding mechanism have?
    * Can check this by reading pendingFunding between fnCall’s that move the pool price….
* What is the system’s behavior in a flash crash?
    * [2021/2/21 BTC Flash Crash](https://medium.com/perpetual-protocol/2021-2-21-btc-flash-crash-149eef35f7f8) (V1)
    * [2021/4/18 Flash Crash](https://perpetualprotocol.medium.com/2021-4-18-flash-crash-19d9a1a16047) (V1)

[300]: https://github.com/perpetual-protocol/perp-lushan/blob/main/contracts/lens/Quoter.sol
[301]: https://github.com/perpetual-protocol/perp-lushan/blob/67d7550fd0fd9fc1ca277a356c53e04bc9ac1985/test/clearingHouse/ClearingHouse.openPosition.test.ts#L204
[302]: https://github.com/perpetual-protocol/perp-lushan/blob/67d7550fd0fd9fc1ca277a356c53e04bc9ac1985/test/clearingHouse/ClearingHouse.partialClose.test.ts#L244-L252
[303]: https://perp.notion.site/Index-price-spread-attack-2f203d45b34f4cc3ab80ac835247030f
[304]: https://perp.notion.site/Bad-Debt-Attack-5cd74c9cc0b845ffa3cf13012c7fdb8c

### Other Perpetual Protocols

As part of the research for this STIP, we looked at 3 other perpetual futures protocols to evaluate the feasibility of building a generalizable module for this asset class. Only one of these was a fully decentralized on-chain system with open sourced code (MCDEX’s [mai-protocol-v3](https://github.com/mcdexio/mai-protocol-v3))

* [dydx](https://docs.dydx.exchange/#perpetual-contracts): This protocol’s order book is centralized on AWS servers.
* [futureswap](https://docs.futureswap.com/): This project’s code [is not open sourced](https://github.com/futureswap/Protocol) at the moment. We may be able to get access through their team. (Punia has contacts with them).

**[Mai-protocol-v3](https://docs.mcdex.io/)**

The Mai protocol shares many of the same properties as PERP but has a broader set of features for liquidity providers and manages leverage and funding differently.

**AMM / Orderbook**

Mai’s futures market is a custom AMM serviced by liquidity providers who receive funding payments. LPs can create their own perpetual markets using the price feed of arbitrary underlying assets and choose any ERC20 as collateral. They can also configure a perpetual market’s margin rate and certain risk parameters that help LPs manage market making operations.

It's worth noting that arbitrary market creation is regarded as a desireable feature and Mai seems to be only protocol offering this. A recent [Derbit market research paper][500] notes:

> The key value of the AMM model is the ability to easily and permissionlessly create new markets. As of yet, none of the protocols are able to offer that. The first protocol to crack a sustainable model, where users are incentivised to create their own perpetual futures markets on anything with a price feed, is likely to see a significant moat.

[500]: https://insights.deribit.com/market-research/the-quest-for-perp-amms/

Another major difference between Mai and PERP is the way their pricing engine incorporates oracle prices alongside AMM liquidity conditions to anchor futures prices in the spot market.

Per the [Pricing Strategy](https://docs.mcdex.io/amm-design#features-of-the-pricing-strategy) section of their docs:


* The AMM price automatically follows the spot price
* Liquidity tends to concentrate near the spot price, reducing slippage

**Trading**

Traders open and reduce positions using a [Trade API](https://github.com/mcdexio/mai-protocol-v3/blob/03d70a5b3801949ffcb7df82e57fd35c07ec91c7/contracts/Perpetual.sol#L146-L203). Each trade realizes funding payments and calculates the current margin available before executing.

**Summary**

Mai’s documentation is thin and difficult to draw conclusions from. However, on the surface it seems like it would be possible to build adapters to map its trading inputs and outputs to an interface which PERP could share.

Resources:

* [Read API](https://github.com/mcdexio/mai-protocol-v3/blob/master/contracts/reader/Reader.sol): this returns data about account balances, liquidity pool state and a way to simulate outcome of a trade.
* [Trade API](https://github.com/mcdexio/mai-protocol-v3/blob/03d70a5b3801949ffcb7df82e57fd35c07ec91c7/contracts/Perpetual.sol#L146-L203)

## Appendix

### **<span style="text-decoration:underline;">PERP Glossary</span>**

<table>
  <tr>
   <td><strong>System Constants</strong>
   </td>
   <td><strong>Description</strong>
   </td>
  </tr>
  <tr>
   <td> Initial margin ratio
   </td>
   <td>10%
   </td>
  </tr>
  <tr>
   <td>Minimum margin ratio
   </td>
   <td>6.25% (liquidation threshold)
   </td>
  </tr>
</table>



<table>
  <tr>
   <td><strong>Account Properties</strong>
   </td>
   <td><strong>Description</strong>
   </td>
  </tr>
  <tr>
   <td>  accountValue
   </td>
   <td>Total collateral value + total unrealized PNL, (10^6)
   </td>
  </tr>
  <tr>
   <td>freeCollateral
   </td>
   <td>
     <p>Your buying power, i.e. how much remaining collateral you have left for opening new positions. Opening positions locks a portion of collateral to secure them.</p>
     <p>&nbsp;&nbsp;<em>min(totalCollateral, accountValue) - total initial margin requirement </em></p>
     <p>(cannot be negative, 10^6)</p?
   </td>
  </tr>
  <tr>
   <td>realizedPNL
   </td>
   <td>The quote quantity received when closing a position plus the open notional quote as debt. For example, a 2 USDC position in vETH might close as below:

<table>
  <tr>
   <td>notionalOpenQuote</td>
   <td align="right"> <code>-2000000000000000000</code></td>
  </tr>
  <tr>
   <td>change in avail quote on sale</td>
   <td align="right"><code>  2367518434188766307</code></td>
  </tr>
  <tr>
   <td>realizedPNL</td>
   <td align="right"><code>367518434188766307</code></td>
  </tr>
</table>

realizedPNL will also include funding payments debited/credited to the position on close.

   </td>
  </tr>
  <tr>
   <td>owedRealizedPNL

   </td>
   <td>
    The sum of:



* Fees accrued when adding & removing liquidity
* Penalties deducted when being liquidated
* Rewards accrued when liquidating
* Realized PNL
   </td>
  </tr>
  <tr>
   <td>
totalUnrealizedPNL

   </td>
    <td>
      <p><em>Net quote balance + sum of all base position values </em> </p>
      <p>(as priced by Chainlink oracles)</p>
    </td>
   </td>
  </tr>
  <tr>
   <td>totalInitialMarginRequirement
   </td>
   <td>
     <p> Minimum funds (USDC) required to open any new position. It is defined per market by initial margin ratio and your current position values. If your account value is lower than it, you cannot open new positions, but since your margin ratio could be still higher than minimum margin Ratio (because imRatio > mmRatio), your existing position might not be liquidated yet. </p>
     <p>&nbsp;&nbsp;&nbsp;&nbsp;<em>_max(totalPositionValue, totalDebtValue) * initial margin ratio </em></p>
   </td>
  </tr>
  <tr>
   <td>pendingFundPayment</td>
   <td>
     <p><em>time = seconds since position opened / 86400</em></p>
     <p><em>payment = positionSize * (AMM market price - Oracle price) *  time</em></p>
   </td>
  </tr>
</table>

### PERP Resources:

* [V2 Docs (WIP)](https://perp.notion.site/perp/Curie-Docs-for-Partners-a2c316abfc1549c7b4d6e310d7a3987d#f6bcb1fadfd04a43a374ff5356bb0058)
* [Perpetual Protocol V2 Technical AMA](https://medium.com/perpetual-protocol/perpetual-protocol-ama-the-curie-release-b8ecc0ad2fb4)
* [Terminology](https://github.com/perpetual-protocol/perp-lushan/blob/db06d9cf8b6ac182cd7a3f61ddb8654b0ab0ffad/TERMINOLOGY.md)
* [Perpetual Swap Contracts Specs and Simulations](https://perp.notion.site/Perpetual-Swap-Contract-s-Specs-Simulations-96e6255bf77e4c90914855603ff7ddd1)
    * [Liquidation Scenario (w/ spreadsheet link)](https://perp.notion.site/Liquidation-ea04a59fe7524427a6ee6523e82a3d5a)
    * [Maker mints and taker swaps](https://docs.google.com/spreadsheets/d/1xcWBBcQYwWuWRdlHtNv64tOjrBCnnvj_t1WEJaQv8EY/edit#gid=1155466937) (spreadsheet)
    * [Uniswap V3 taker swaps](https://docs.google.com/spreadsheets/d/1H8Sn0YHwbnEjhhA03QOVfOFPPFZUX5Uasg14UY9Gszc/edit#gid=523274954) (spreadsheet)
    * [Margin Model Comparison](https://docs.google.com/spreadsheets/d/1Kmaw7ZawxhP6mbsi1P5UrFF1PJB8FdobH-FZ7Ht7Z04/edit#gid=0) (spreadsheet)
    * [Margin Simulation](https://docs.google.com/spreadsheets/d/1kjs6thR9hXP2CCgn9zDcQESV5sWWumWIsKjBKJJC7Oc/edit#gid=574020995) (spreadsheet)
    * [Trading Simulation](https://docs.google.com/spreadsheets/d/1jClwDZ93_hPScLmeVtGo-VFJJDQJPoqT3WVGl5TzM-E/edit#gid=21644967) (spreadsheet)
    * [Long/Short swaps (USD & ETH) State change diagrams](https://www.figma.com/file/xuue5qGH4RalX7uAbbzgP3/swap-accounting-%26-events?node-id=0%3A1)
* Flash Crashes (autopsies)
    * [2021/2/21 BTC Flash Crash](https://medium.com/perpetual-protocol/2021-2-21-btc-flash-crash-149eef35f7f8) (V1)
    * [2021/4/18 Flash Crash](https://perpetualprotocol.medium.com/2021-4-18-flash-crash-19d9a1a16047) (V1)


### **<span style="text-decoration:underline;">Simple PERP Simulations</span>**

[Code for Leverage / Oracle Scenarios ](https://gist.github.com/cgewecke/5be20b94f9166109b0aee72c5308831d)

**<span style="text-decoration:underline;">Chainlink Price Oracle Impact</span>**

**Scenario: Chainlink oracle price effects in isolation**

* Taker deposits 1 USD into protocol
* Initial BaseToken Chainlink Price = 10 USD / 1 BaseToken
* Final BaseToken Chainlink Price = 20USD / 1 BaseToken

![chainlink impact](https://user-images.githubusercontent.com/7332026/134934291-f7c0ea79-f41e-4ab7-b914-93935bfc0e97.png)

**<span style="text-decoration:underline;">Levering Up</span>**

**Scenario: 9X leverage from 1 USD**

This case represents opening a position for 9 USDC worth of BaseToken with a vault deposit of 1 USDC. (9X seems to be an upper bound, wasn’t able to open for 9.1) ** **

![lever 9x](https://user-images.githubusercontent.com/7332026/134934301-8dcccf4e-f8a9-4264-812c-a21172f0b4d3.png)

**Scenario: Adding to positions with leverage**

This case represents opening a series of 1 USDC positions given an initial 1 USDC vault deposit. It highlights the way the 1% fee is applied to account values as positions are opened.

![add position lev](https://user-images.githubusercontent.com/7332026/134934308-641d48bb-2d21-4c37-8e96-bd36700183c2.png)

**Scenario: Closing position at a profit with 2X leverage**

* **Deposit: 1 USDC**
* **Open position for 2 USDC**
* **vAMM Price moves up ~20% (10 -> 12.08)**

Nets a 36.7% gain using 2X leverage on a 20.07% increase in the vAMM market price. ** **

![close profix 2X](https://user-images.githubusercontent.com/7332026/134934319-4dfc8288-9371-4f12-9c4f-b6dab8da52a8.png)

**<span style="text-decoration:underline;">AMM Virtual Markets and Settlement</span>**

**Scenario 1: Taker long 100 USDC, AMM market price rises 10%, taker closes position**

This case shows how positions exit net of fees, as well as the way in which many account values remain static even as other traders open positions that move the AMM price.

![taker long](https://user-images.githubusercontent.com/7332026/134937767-20debf91-bd46-44b8-9a63-2749833084bb.png)

**Scenario 2: Taker long 100 USDC, Oracle price rises 100%, AMM market price rises 10%**

This case highlights the way in which many account values are tied to oracle price but the realizable value of a position depends on the AMM market price. The outcome here despite a large spot price increase is the same as in the above where spot price is static.

**Note**: this example excludes funding flows which capture the difference between AMM and Oracle prices over time.

![oracle 100](https://user-images.githubusercontent.com/7332026/134937777-295f5a5e-46e2-4cc7-a8c7-dc539e5a9317.png)

**Scenario 3: Taker short 10 BaseToken (100 USDC), AMM price drops 13%**

This case highlights the way in which short positions in the money are behaviorally symmetric to longs.

![amm drop 13](https://user-images.githubusercontent.com/7332026/134937794-136fa6fe-96cf-4a25-b5ca-b2d071a452fb.png)

**Scenario 4: Reducing positions**
* **Taker long 100 USDC**
* **AMM market price rises 10%**
* **Taker reduces position by half**

This scenario is the same as scenario 1 but models account state after a partial reduction of a long position.

![amm up 10](https://user-images.githubusercontent.com/7332026/134937806-f4df9eb1-76e7-44fe-8f87-ac6035024d67.png)

**<span style="text-decoration:underline;">Liquidations / Closing levered positions</span>**

**Scenario 1: Taker long 3 USDC, leveraged 3X, Oracle price drops from $10 to $7.15**

In this case, AMM and Oracle prices have dislocated. Position is above the liquidation threshold if oracle price is $7.20. Moving the AMM market to similarly low price levels *does not* expose account to liquidation.

![liquid 1](https://user-images.githubusercontent.com/7332026/134937817-eca5254e-5ece-416b-9671-97f6dd11ee99.png)

**Scenario 2: Close leveraged position at loss. Long 3 USDC, (3X),  AMM price to $7.12.**

In this case, Oracle and AMM market prices are dislocated, AMM price is below the liquidation threshold but can’t be closed by 3rd party. Outcome is interesting to compare with liquidation (above) since exiting position at these price levels incurs substantially greater loss.

![liquid 2](https://user-images.githubusercontent.com/7332026/134937869-9a9222d1-cd28-43ba-944f-dadcee04e9ae.png)

**<span style="text-decoration:underline;">Funding</span>**

[Gist of scenario case code](https://gist.github.com/cgewecke/24d60f628bebe9c998dffc62763030fc)

Funding payments between makers and takers are based on the difference between the AMM market price and the oracle price and accrue over time following the formula below.

```
positionSize * (AMM market price - Oracle price) *  ( time since position opened / day )
```

When taker opens a long position and the oracle price is below AMM market, taker pays funding fees. When oracle price is above AMM market price, taker receives funding fees.

**Scenario 0: Baseline: Taker opens long 100 USD for .648 Base and immediately closes.**

This case provides a comparison basis for how funding fees net out in the subsequent scenarios.

![funding 0](https://user-images.githubusercontent.com/7332026/134937889-1f332759-61ae-4026-b2b8-ef0f2c8902cc.png)

**Scenario 1: Paying funding fees as a taker**
* **long 100 USD**
* **AMM price $3 > Oracle price for 1 day**
* **Close position at a loss with funding debited**
* **No other traders (only liquidity providers)**

![funding 1](https://user-images.githubusercontent.com/7332026/134937894-92b12f9d-d8db-4d65-bf0a-ac115ea9fa29.png)

**Scenario 2: Collecting funding fees as a taker**
* **long 100 USD**
* **Oracle price $3 > AMM price for 1 day**
* **close position with funding credited**

![funding 2](https://user-images.githubusercontent.com/7332026/134937911-2770ee77-b21f-465d-9bb5-e162c1c73081.png)

**Scenario 3: Collecting funding fees as a leveraged taker**
* **long 200 USD (2X)**
* **Oracle price moves above AMM price for 1 day**
* **close position with funding credited**

Baseline value after closing this position (without funding) is 96020049

![funding 3](https://user-images.githubusercontent.com/7332026/134937920-b94af2c9-b468-4c3e-8ed2-71199d802018.png)

**Scenario 4: Collecting funding fees as a taker when AMM prices drops below Oracle**

* **long 100 USD**
* **other participant short positions move AMM price $3 &lt; Oracle price for 1 day**
* **close position at a loss with funding credited**

The baseline outcome for closing this position at a loss without funding payments is 94544705

![funding 4](https://user-images.githubusercontent.com/7332026/134937930-dcc3e90e-b9a5-4e11-b5e1-96432810a7e2.png)

**Scenario 5: Effect of time on funding settlement**
* **long 100 USD**
* **Oracle price moves above AMM price for 1 day**
* **Oracle price moves back in line with AMM**
* **close position**

In slack, Brian noted that funding settlement for accounts only occurs when positions are opened / closed.

“...so could be in your favor for the whole time then flips for one day, you exit, and you end up paying”

(This is accurate, compare with Funding scenario #2)

![funding 5](https://user-images.githubusercontent.com/7332026/134937954-d8d9458c-2e71-4788-a5eb-eb658fa8d00f.png)

**Scenario 6: Funding payments when position is reduced (rather than closed).**

The complete amount of pending payments is settled to _owedRealizedPNL_ when any increase or reduction in the position occurs.

![funding 6](https://user-images.githubusercontent.com/7332026/134937969-6eb7661d-fc5b-4984-8800-1407a1561c12.png)

**Reviewer**:

## Checkpoint #2 Proposed Architecture

**Reviewer**:

**Reviewer**:

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()
[Link to Deploy outputs PR]()
