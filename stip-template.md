# STIP-[xx]
*Using template v0.1*
## Abstract
Should be a brief description of the problem, and a small explainer of why it's a problem. There should be no discussion of potential solutions in the Abstract. 
## Motivation
This section should clearly answer the 4 big questions:  
- What is this feature?
- Why is this feature necessary
    - Ideally a lot of this is captured in a good PRD
- Who is this feature for?
    - Managers, issuers, arbers, app devs
- When is this feature going to be used? (At any time? Or only at certain times?)
- How is this feature intended to be used?
    - This may also include potential products/use cases this feature enables

If there are multiple user stories associated with this feature it may make sense to ask all of these questions in the context of each user story. Feel free to sub-divide this section however necessary.
## Background Information
This section should contain any relevant info required for understanding the problem at hand. This may include any of the following:
- Previous work done on the topic
- Discussion of any relevant parts of the Set system
- Documentation on any external protocols to consider when designing the solution. Links are great but providing relevant interfaces AND a brief description of how the protocol works is a big plus, highlighting any nuances (ie in AAVE interest accrues by creating more aTokens vs Compound accrues by updating cToken to underlying exchange rate)
## Open Questions
Pose any open questions you may still have about potential solutions here. We want to be sure that they have been resolved before moving ahead with talk about the implementation. This section should be living and breathing through out this process.
- [ ] Question
    - *Answer*
## Feasibility Analysis
Provide potential solution(s) including the pros and cons of those solutions and who are the different stakeholders in each solution. A recommended solution should be chosen here.
## Timeline
A proposed timeline for completion
## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. Additionally we should feel that all necessary information on external protocols has been gathered and potential solutions have all been well considered. It is up to the reviewer to determine whether we move onto the next step.

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
