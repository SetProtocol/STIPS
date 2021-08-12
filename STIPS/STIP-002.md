# STIP-002
*Using template v0.1*
## Abstract
The WrapModule is set up so that its adapters do not have access to critical information such as the SetToken that it is wrapping on the behalf of. Additionally, it does not work in certain instances due to the lack of setting appropriate approvals when unwrapping.

## Motivation
We will build an upgraded WrapModule called WrapModuleV2 that includes several changes that designed to both future-proof the module and make its interfaces conform more closely with the other modules. This will allow Set designers to more easily implement new wrap adapters.

## Timeline
spec: 8/13
implementation 8/16
internal review 8/20
deployment: 8/25

## Checkpoint 1
**Reviewer**: @richardliang

## Proposed Architecture Changes
- Add a `_wrapData` parameter to the `wrap` and `wrapWithEther` functions of `WrapModuleV2`
- Add a `_unwrapData` parameter to the `unwrap` and `unwrapWithEther` function of `WrapModuleV2`
- Add a `_setToken` parameter to the `getWrapCallData` and `getUnwrapCalldata` function of `IWrapV2Adapter`
- Add a `_wrapData` parameter to the `getWrapCallData` function of `IWrapV2Adapter`
- Add a `_unwrapData` parameter to the `getUnwrapCallData` function of `IWrapV2Adapter`
- Add a `invokeApproval` call to the `_validateUnwrapAndUpdate` internal function of `WrapModuleV2`

## Requirements
- Minimal changes required for legacy wrap adapters
- Appropriate token approvals for unwrapping
- Ability to pass in arbitrary wrapData and unwrapData bytes

## Checkpoint 2
**Reviewer**: @richardliang

## Specification
Note: only functions that will be modified  between WrapModule and WrapModuleV2 are shown

### WrapModuleV2
#### Functions
| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|wrap|Manager|wraps a set component|
|wrapWithEther|Manager|wraps a set component but first converts WETH to ETH|
|unwrap|Manager|unwraps a set component|
|unwrapWithEther|Manager|unwraps a set component but converts received ETH to WETH|

#### Functions
> function wrap(ISetToken _setToken, address _underlyingToken, address _wrappedToken, uint256 _underlyingUnits, string calldata_integrationName, bytes memory _wrapData)

> function wrapWithEther(ISetToken _setToken, address _wrappedToken, uint256 _underlyingUnits, string calldata_integrationName, bytes memory _wrapData)

> function unwrap(ISetToken _setToken, address _underlyingToken, address _wrappedToken, uint256 _wrappedUnits, string calldata_integrationName, bytes memory _unwrapData)

> function unwrapWithEther(ISetToken _setToken, address _wrappedToken, uint256 _wrappedUnits, string calldata_integrationName, bytes memory _unwrapData)

## Specification
### IWrapV2Adapter
#### Functions
| Name  | Caller  | Description 	|
|------	|------	|-------------	|
|getWrapCalldata|WrapModuleV2|gets calldata to wrap a set component|
|getUnwrapCalldata|WrapModuleV2|gets calldata to unwrap a set component|

#### Functions
> function getWrapCallData(address _underlyingToken, address _wrappedToken,uint256 _underlyingUnits, address _toAddress, bytes memory _wrapData)

> function getUnwrapCallData(address _underlyingToken, address _wrappedToken,uint256 _wrappedUnits, address _toAddress, bytes memory _unwrapData)

## Checkpoint 3
**Reviewer**: @richardliang

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()  
[Link to Deploy outputs PR]()
