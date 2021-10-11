# STIP-005
*Using template v0.1*
## Abstract
The current TradeModule and GeneralIndexModule do not support volume based fees or rebates for the manager. This is in contrast to the issuance module. There is demand for additional fee types that managers can charge, specifically to be on par with exchanges where there are rebate / volume fee discounts.

## Motivation
This feature is a new version of a TradeModule (and GeneralIndexModule) that allows the manager to charge a volume fee from their followers. This will be denominated in the output token similar to the protocol fee assessed. In contrast to centralized exchanges, the fee rebates will be automatic per trade.

TradeModuleV2 will be used in upcoming TokenSets deployments on other chains, and can replace the existing on Polygon and Mainnet. GeneralIndexModuleV2 can be used by index coop products to generate additional revenue

## Background Information
We have previously implemented the TradeModule and GeneralIndexModule. This will be an update to allow manager trade rebates charged on the output token. We have previously implemented modules for both manager and protocol fees (NAVIssuanceModule, DebtIssuanceModule) with differing mechanisms

## Open Questions
- [ ] Should we ensure that manager must have capital in the Set to charge volume fees?
    - No, we can encourage in the UI instead for retail managers. This would also break for cases where the manager is a smart contract
- [ ] Should we name this TradeModuleV2 or TradeModuleWithRebates (GeneralIndexModuleV2 or GeneralIndexModuleWithRebates)?
- [ ] How do we prevent managers charging an 100% fee and rugging the Set on a trade?

## Feasibility Analysis

#### Option 1
Adding a managerFee % which is assessed on top of the protocol fee %. This will be least code changes to the TradeModule and General Index Module. To avoid rugging, there will be a maxManagerFee enforced.
```
uint256 protocolFeeTotal = getModuleFee(TRADE_MODULE_PROTOCOL_FEE_INDEX, _exchangedQuantity);

// Get manager fee here
```

#### Option 2
Adding a managerFee and a protocol feeSplit % (similar to DebtIssuanceModule). This will make TradeModule and GeneralIndexModule fees variable in the protocol (if a manager charges 0 fees here, then protocol will always make 0) which is a departure from TradeModule V1

```
uint256 protocolFeeSplit = controller.getModuleFee(address(this), ISSUANCE_MODULE_PROTOCOL_FEE_SPLIT_INDEX);
uint256 totalFeeRate = _isIssue ? setIssuanceSettings.managerIssueFee : setIssuanceSettings.managerRedeemFee;

uint256 totalFee = totalFeeRate.preciseMul(_quantity);
protocolFee = totalFee.preciseMul(protocolFeeSplit);
managerFee = totalFee.sub(protocolFee);
```

#### Option 3
Manager charges a managerFee that is split with the protocol based on a feeSplit % determined by governance. Additionally, the protocol can charge a direct fee % (similar to NAVIssuanceModule). This will be a superset of TradeModule fees charged

```
uint256 protocolDirectFeePercent = controller.getModuleFee(address(this), _protocolDirectFeeIndex);
```

#### Recommendation
Option 1 will require the least code changes while giving us the ability to emulate fee rebates. The maxManagerFee can be enforced either on initialization (still susceptible to manager rugging by removing and reinitializing) or stored on the Controller by governance

## Timeline
- Spec + review: 2 days
- Implementation: 2 days
- Internal review: 1 days
- Deployment scripts: 1 day
- Deploy to testnet: 1 day
- Testing: 1 day
- Write docs: 1 day

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
| Type  | Name  | Description   |
|------ |------ |-------------  |
|address|manager|Address of the manager|
|uint256|iterations|Number of times manager has called contract|  
#### Constants
| Type  | Name  | Description   | Value     |
|------ |------ |-------------  |-------    |
|uint256|ONE    | The number one| 1         |
#### Public Variables
| Type  | Name  | Description   |
|------ |------ |-------------  |
|uint256|hodlers|Number of holders of this token|
#### Functions
| Name  | Caller  | Description     |
|------ |------ |-------------  |
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
