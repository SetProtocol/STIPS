# STIP-007: Issuance Transformation Pipeline
*Using template v0.1*
## Abstract
This STIP explores an implementation for a transformation pipeline that can take a specified amount of tokens and transform them into all the positions held by a Set for issuance. Conversely, also allow someone to redeem all the positions from a Set into some amount of a specified token.
## Motivation
This feature is to be used by issuers and redeemers to pay/receive one asset when issuing/redeeming a SetToken. This "transformation pipeline" will take the one asset and trade it into the necessary positions using a DEX. This feature provides a considerable UX improvment for users over having to replicate every position in a Set in the exact proportions then issuing a Set. Additionally, it does not require seeding an AMM pool with Set liquidity in order to allow users to get into the Set by buying it off of an exchange. The "transformation pipeline" will allow the longer tail of Set managers to enter complex positions without hindering the ability for users to issue or redeem the Set. This feature will however be gas intensive so will only be viable on cheaper EVM compatible L1s and L2s.

This feature should be able to handle the following asset types:
- Spot tokens
- Wrapped tokens (AAVE, Compound)
- LP tokens (Curve, UniswapV2, Sushiswap)
- Sets that charge a mint or redeem fee

Nice-to-have:
- Leveraged Positions
- Perpetuals (since still under development)

## Background Information
### Previous Work
[SetV1 Exchange Issuance](https://github.com/SetProtocol/set-protocol-contracts/blob/master/contracts/core/modules/RebalancingSetExchangeIssuanceModule.sol) - Set V1 had a version of a transformation pipeline that sourced liquidity from market makers to help transform a pay asset into the necessary asset mix to mint a V1 Set. In order to replicate the position, bytedata that encoded the necessary trades was passed in and then sent to an exchange specific wrapper which parsed the data and executed the trades on the exchange. A SetToken was then minted from the resulting assets and any change was swept back to the user.

[0x Features/Transformers](https://protocol.0x.org/en/latest/architecture/overview.html) - The equivalent of 0x's transformation pipeline uses something they call a `Flash Wallet`. This `Flash Wallet` is stateless which safely allows it to `delegatecall` transformers which execute their logic within the context of the `Flash Wallet`. Transformers are trustless contracts identified by a nonce which execute a transformation according to data passed from the `Flash Wallet`. Since the `Flash Wallet` is stateless, the whole ordering and execution of the transformation process is managed by a `Feature` contract. The feature contract is where calls originate within the pipeline, different transformations are invoked on the `Flash Wallet`, and any checks to make sure correct amount of tokens are received are done. In architecture, there are a lot of similarities to SetV1 except we did not `delegatecall` into our equivalent of transformers and instead transferred tokens to and from the transformer (what we called a wrapper).

### Relevant Pieces of Set System
#### BasicIssuanceModule
BasicIssuance will attempt to transfer in all the assets required to mint a specified amount of Sets. There are no mint/redeem fees that the pipeline needs to take into account. All we need to do is make sure we have enough assets to mint the amount of Sets we ask for.
#### DebtIssuanceModule
DebtIssuance adds the complication of mint/redeem fees and issuing of debt positions requires returning debt to the issuer. Ideally, a flash loan flow could be built that borrows the extra collateral needed upfront and then uses the returned debt to help pay down the loan by trading it back to the flash loaned asset. Mint/redeem fees should be eaily handled though.
#### Adapters
Our currently deployed adapters are relevant because we could *potentially* use them to generate the necessary bytecode to interact with any protocol we need for transformations. In order to meet our current requirements we would need to use the adapters from 1) `TradeModule`, 2) `WrapModule`, and 3) `AmmModule`. For leveraged positions we can use the `WrapModule` assuming we have an adapter for wrapping into the underlying protocol. To make this method viable we also need to make sure there is some overlap between adapter interfaces, otherwise we may as well just pass in bytedata that can be forwarded to the relevant protocols. A summary of interfaces for each adapter is below:
| Adapter/Module 	| Interface                                                                                                                                                                              	| Notes                                                                                                                                    	|
|----------------	|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|------------------------------------------------------------------------------------------------------------------------------------------	|
| TradeModule    	| function getTradeCalldata( address _sourceToken, address _destinationToken, address _destinationAddress, uint256 _sourceQuantity, uint256 _minDestinationQuantity, bytes memory _data) 	| `_data` parameter encodes special info depending on the integration, for 0x it encodes the whole trade, for UniswapV3 it encodes the fee 	|
| WrapModule     	| function getWrapCallData( address _underlyingToken, address _wrappedToken, uint256 _underlyingUnits, address _to, bytes memory _wrapData)                                              	| `_wrapData` param has not been used in the wild but could be used to encode some arbitrary data.                                         	|
| AmmModule      	| function getProvideLiquidityCalldata( address _setToken, address _pool, address[] calldata _components, uint256[] calldata _maxTokensIn, uint256 _minLiquidity)                        	| `setToken` parameter is only used for a balance check and then as a `to` address, which would be compatible with this implementation.    	|

You will notice that these different adapters have the same general pattern in common:
1. There is a source token(s) and an amount of source token to transform (we can make the source tokens/amounts param an array to support `AmmModule`)
2. There is a destination token and an expected minimum amount to get back (except WrapModule where we could 0 it out)
3. There is a recipient address (`TradeModule`: `destinationAddress`, `WrapModule`: `_to`, `AmmModule`: `_setToken`)

## Open Questions
- [ ] Do we set a defined order of actions? (ie trades, wraps, LPs)
- [ ] Do we utilize our adapters to facilitate the transformations?
- [ ] What level of decentralization is necessary? Can we just pass in parsable bytedata or do trades have to be figured out all on-chain?
    - *Start with decentralization as a lower priority. Ok with relying on centralized API to generate the necessary transformations*
## Feasibility Analysis
Provide potential solution(s) including the pros and cons of those solutions and who are the different stakeholders in each solution. A recommended solution should be chosen here.
### Option 1: Use current adapters to create invokable bytedata
**Summary:** Pass in standardized inputs which point to an adapter that generates bytecode that is invoked by the wallet

**Pros:**
- Pushes most of the work off-chain to figuring out how many assets to trade for, what to wrap, etc.
- Allows us to be sure that we're doing transformations that are supported by the protocol
- Fairly quick to market
- Inputs are far more human-readable

**Cons:**
- Reliant on a more centralized off-chain API
- May be issues with on-going compatibility since adapters are built more for modules
- May require additional contracts/code to turn standardized data inputs into a function call to the adapter

### Option 2: Use 0x-like implementation
**Summary**: Pass in bytecode that is executed sequentially without much bytecode validation.

**Pros:**
- Pushes most of the work off-chain to figuring out how many assets to trade for, what to wrap, etc.
- Probably the quickest to market and may only require 1 contract

**Cons:**
- Reliant on a centralized off-chain API
- Inputs are much harder to read/verify

### Option 3: Determine all trades purely on-chain
**Summary**: Pass in a pay token amount and an amount of SetToken's you would like to receive. All trades and transformations are then figured out on-chain.

**Pros:**
- Most decentralized solution

**Cons:**
- Slowest to market
- Probably would require the most on-chain upkeep

### Proposed Solution
I would select either Option 1 or 2. Option 1 is more user friendly but may require a bit more on-going maintenance, it would also still require the use of a centralized API to calculate the ideal transformations. Option 2 is essentially like Option 1 but there is no attempt to standardize all the inputs, it's is just raw bytedata that is executed sequentially, would not require much additional upkeep but would be much harder for others to use/verify without our API.

## Timeline
A proposed timeline for completion
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
