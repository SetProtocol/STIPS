# STIP-[xx]
## Abstract
As Set Protocol begins to support larger products, liquidity will increasingly become an issue for rebalances. Currently, we are limiting the size of trades for the FLI products, and using a TWAP strategy to accommodate these liquidity constraints. An on-chain aggregator that allows us to direct our trades to multiple sources can allow us to solve these problems trustlessly.

## Motivation
This feature would be a trade adapter that uses some strategy to direct trades to the deepest liquidity, potentially splitting trades to use multiple liquidity sources. This is most useful for managers and product developers. It provides the most benefits for automatic products such as the FLI suite, since rebalances must be able to occur trustlessly, with all computation done on-chain. This feature would likely become integrated deeply into current and future products that require making large trades automatically.

## Background Information
Three of the largest sources for on-chain liquidity are Uniswap V3, Uniswap V2, and Sushiswap. While Uniswap V2 and Sushiswap are constant coefficient AMMs, making computing optimal trade splits between the two easy, Uniswap V3 uses a much more nuanced strategy, which makes computing optimally sized trades much more inefficient.

Useful links:  
https://docs.uniswap.org/  
https://uniswap.org/docs/v2/  
https://github.com/ncitron/TradeSplitterResearch  


## Open Questions
- What is the proper balance of gas optimization to trade price optimization?

## Feasibility Analysis
## Computing efficient splits between Uniswap V2 and Sushiswap:
Since price impact scales linearly with liquidity, the trade size must be split proportionally to the pool size for single hop trades. For multihop trades, the ratio between Sushiswap and Uniswap is:
```
Ts/Tu = ((Pua + Pub) * Psa * Psb) / ((Psa + Psb) * Pua * Pub)  

Ts  = Sushiswap trade size
Tu  = Uniswap trade size
Pua = Uniswap liquidity for pool A
Pub = Uniswap liquidity for pool B
Psa = Sushiswap liquidity for pool A
Psb = Sushiswap liquidity for pool B
```
There derivation of this equation is:
```
If we begin with the approximation that the price impact for a trade is:
I = 2T / P
This approximation holds under the assumption that the trade size does not make up a significant percentage of the total trade volume.

Next, we approximate that the total price impact of a multi hop trade is equal to the sum of the price impacts:
I = 2Ta / Pa + 2Tb / Pb
This approximation holds under the assumption that the price impact is a relatively low percentage (after 20% or so it become a bad approximation)

The optimal trade split will be when the Uniswap and Sushiswap trades experience the same price impact. This situation can be represented as:
2Ts / Psa + 2Ts / Psb = 2Tu / Pua + 2Tu / Pub

Which can be simplified as follows
Ts / Psa + Ts / Psb = Tu / Pua + Tu / Pub
(Ts * Psa + Ts * Psb) / (Psa * Psb) = (Tu * Pua + Tu * Pub) / (Pua * Pub)
Ts * (Psa + Psb) / (Psa * Psb) = Tu * (Pua + Pub) / (Pua * Pub)
Ts * (Psa + Psb) * Pua * Pub = Tu * (Pua + Pub) * Psa * Psb
Ts/Tu = ((Pua + Pub) * Psa * Psb) / ((Psa + Psb) * Pua * Pub)
 ```
This strategy assumes that the price on Uniswap is equal to the price on Sushiswap. In cases where the trade size is exceedingly large, it is likely that this assumption is fair, as the price difference is likely much less than the price impact.

## Computing splits between Uniswap V3 and Uniswap V2
During this section, we will establish different strategies for aggregating trades between Uniswap V2 and V3. One thing to note here is that each of these strategies can be easily modified to aggregate liquidity between Uniswap V3, Uniswap V2, and Sushiswap.
### Strategy 1: Off-chain exact split
This is likely the simplest, but least trustless strategy. This requires that we calculate the price impact experienced at regular interval splits between v2 and v3. When we find a percentage split where the price impact is identical for v2 and v3, we have found the one that experiences the least total slippage. An example of this method can be seen here: https://github.com/ncitron/TradeSplitterResearch/blob/master/scripts/offchainV2V3Splitter.ts. This method's main drawback is requiring a trusted keeper to calculate this split off-chain. Additionally, this method is slow to compute with a high degree of accuracy. There may be some optimizations we can make to increase its speed.
### Strategy 2: On-chain exact split
It may be possible to efficiently compute the split on-chain. This strategy would require completing a similar computation as Uniswap V3 does when it performs a swap. We would perform a mock v3 swap in our own contract, where we calculate the amount of liquidity that has been swapped as we pass each liquidity tick. At the end of each tick, we know our current tick position (and thus the amount of price impact we have so far experienced), as well as the amount we have already swapped (and thus the amount left to be swapped). At each tick, we would use the amount left to be swapped, and check what the price impact would be in a simulated v2 swap for that amount. When the price impact of the simulated v2 swap is equal to the price impact of the ongoing simulated v3 swap, we have determined the optimal split.
This strategy has several downsides. The first is gas costs. Although this is likely to be significantly cheaper than the naive approach, it still requires simulating a mock v3 swap, and many mock v2 swaps (fortunately the mock v2 price impact calculation is very cheap). Another downside to this strategy is its complexity. Successfully executing this requires a deep understanding of Uniswap V3, and is therefore prone to bugs. Finally, I am unsure how this strategy can be expanded to accommodate multihop transactions. An implementation for this would add even more complexity, and would, unfortunately, be required for BTC2x-FLI.
### Strategy 3: V3 with sqrtPriceLimit then V2
Uniswap V3's sqrtPriceLimitX96 parameter provides a min/max on what the price ratio of the assets will be after a swap is executed. This means that we could instead execute our swap in terms of a maximum acceptable price impact, rather than just aiming to be the most efficient. For example, we can tune this parameter to execute a trade with a maximum of 1% price impact. We can also calculate the V2 trade to aim for a 1% impact as well. For example, at the time of writing, a 1% price impact would allow us to sell ~580 ETH into USDC on Uniswap V2, plus an additional 5300 ETH on Uniswap V3. If our trade size was just 500 ETH, 100% of the trade would flow through V3. If our trade size though was 5500 ETH, then the first 5300 would flow through V3, and the additional 200 ETH would flow through V2. Under normal market conditions, this strategy would likely route 100% of the liquidity to V3 (which would provide us with a better trade than using just V2, but be slightly suboptimal compared to a perfect V2/V3 split). In abnormal market conditions, when V3 is performing poorly due to a large amount of liquidity being out of range of the current price, our trade would be partially routed through V3 (until we hit the 1% price impact limit), then the spillover of the trade would be routed through V2 (which would complete the trade, since a V2 pool can absorb relatively large trades at 1% price impact even in abnormal market circumstances). Unlike slippage, setting a high price impact cannot be sandwich attacked, meaning we do not have to worry about this strategy leaking additional MEV to bots. An example of executing a trade with a ~1% price impact limit can be seen here: https://github.com/ncitron/TradeSplitterResearch/blob/master/scripts/sqrtLimitSwap.ts. This strategy becomes more difficult for multihop trades, since there is no sqrtPriceLimit parameter for these trades. One potential solution to this is to split a multihop trades into two single hop trades with a price impact target of 0.5% each. Unfortunately if the second pool has less liquidity, it is possible to be stuck with some leftover amount of the intermediate token, which would require a third trade to swap back to the original asset.
This strategy has the benefits of occurring entirely on-chain, while also being more gas efficient. Additionally, this strategy will perform almost as well a strategies 1 and 2 during normal market conditions. Even in abnormal market conditions, this strategy limits the price impact to a known value (1% in these examples).
### Strategy 4: Optimistic execution of V3, with V2 as a fallback
This strategy makes use of the Solidity's try/catch statement, to produce outcomes similar to strategy 3, but with a reduced gas cost for multihop trades. First calculate the price impact of a V2 swap. Next calculate a minOutput parameter for the V3 swap guarantees that the V3 swap perform better than V2. We can wrap this is a try/catch. If the V3 swap is successful then we finish execution. If the swap reverts (since it could not provide a better price than V2), then it catches and executes the swap on V2. The reason we execute the V3 swap optimistically is because getting a price quote for Uniswap V3 is nearly as expensive as actually executing the trade. This strategy has many of the same positive properties as strategy 3, but at a much lower gas price.

## Implementation Discussion
### Trade Routing:
Trade routing, where only the deepest liquidity source is used in each transaction, can be implemented in a variety of different ways. The simplest allows us to only use a single adapter with no need for any periphery contracts. This adapter would get quotes for trades using Uniswap V3, Uniswap V2, and Sushiswap, and then produces the calldata for whichever exchange gave the best quote. Additionally, this strategy can use the adapters of each exchange as a helper for producing the required calldata. This strategy has the benefit of simplicity, but also has additional gas overhead since receiving a quote for Uniswap V3 costs nearly as much as actually performing the swap.

A more gas efficient strategy for routing trades using an optimistic execution of Uniswap V3. This strategy requires that we use a periphery contract to execute the trade, and write an exchange adapter that routes trades through our custom exchange contract. To start, we receive quotes for the trade using Uniswap V2 and Sushiswap, and provide the best quote as the minOutput parameter for a Uniswap V3 swap. If Uniswap V3 provides a better price than Uniswap V2 or Sushiswap, then this trade will execute successfully. If it does not, this trade will revert, and using solidity's try/catch functionality, we can continue execution and instead use Uniswap V2 or Sushiswap. This strategy has the benefit of increased gas efficiency over the prior strategy, since it does not have to make the redundant call to get a Uniswap V3 quote. The main con of this system is the additional complexity.

### Trade Splitting
Trade splitting, where we send liquidity through multiple liquidity sources in a single transaction can be implemented using a periphery exchange contract, or through a new module. The first strategy is likely the simplest, since it uses the existing trade adapter system, while giving us the most flexibility. This strategy would also allow us to do a combination of trade routing and trade splitting, where we perform strategy four, with a perfect Uniswap V2 / Sushiswap split as the fallback to Uniswap V3.

Creating a new module would give us a system that may be more reusable, but with additional handicaps, since many splits would have to be precomputed, removing the possibility of executing trades optimistically. The TradeSplitterModule would work much like the current TradeModule, but would allow for making multiple trades using different exchanges.

## Recommendation
The trade splitting and trade routing combo using a peripheral contract is gas efficient, while still performing well in the worst case scenario when V3 cannot be used, since it will then split the trade using Uniswap V2 and Sushiswap. This strategy requires a peripheral contract to optimistically execute the V3 trade, which avoids making redundant calls to fetch a Uniswap V3 price quote.

## Table

| Strategy | Naive Trade Splitting | Trade Routing | Trade Split / Route Combo |
|----------|------------------------------|-------------------|---------------------------------------|
| Pros | Creates most efficient trade | Gas efficient, can be done with no peripheral contracts or new modules| Gas efficient, better trades          |
| Cons | Very gas inefficient, requires peripheral contract or new module | Suboptimal trades | Trades not as good as the naive split, requires peripheral contract or new module  |

## Timeline
- Spec + review: 4-5 days
- Implementation: 4-5 days
- Internal review: 2 days
- Deployment scripts: 1 day
- Deploy to testnet: 1 day
- Testing: 3-5 days
- Write docs: 1-2 days

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
