# Token Management

The Provider's tool for measuring how each Client's people use AI Tools and what that AI Consumption costs, so the Provider can show where AI spend goes and where it is wasted.

## Parties

**Provider**:
Our consulting company, the single organization that runs Token Management for its Clients.
_Avoid_: Vendor, supplier, MSP

**Client**:
An organization the Provider measures AI Consumption for; each Client sees only its own.
_Avoid_: Customer, tenant, account

**Delivery Manager**:
A Provider employee accountable for the Managed Service of one or more Clients.
_Avoid_: Account manager, service manager

**Consultant**:
A Provider employee assigned to Clients who carries out the delivery work.
_Avoid_: Analyst, engineer, resource

**Client Approver**:
A Client user who may decide on Optimization Opportunities on the Client's behalf.
_Avoid_: Client admin, sponsor

**Client Viewer**:
A Client user with read-only access to their Client's AI Consumption.
_Avoid_: Read-only user, guest

## Service

**Managed Service**:
An ongoing service the Provider delivers to a Client under one contract, covering one or more Service Lines.
_Avoid_: Engagement, contract

**Demo Client**:
A fictitious Client with synthetic AI Consumption, used to demonstrate Token Management and kept apart from real Clients.
_Avoid_: Test client, sandbox

**AI Consumption Management**:
The Service Line that measures a Client's AI Consumption across AI Tools, estimates its cost and finds where it is wasted.
_Avoid_: AI FinOps, AI SAM, tokenomics

## Consumption

**AI Tool**:
An AI product a Client pays for, either Metered or Seat-licensed, such as the Anthropic API, Azure OpenAI or Microsoft 365 Copilot.
_Avoid_: AI app, AI service, LLM

**AI Token**:
The unit of text a model reads or writes, by which AI usage is measured and Metered AI Tools are priced.
_Avoid_: Token (bare), credit

**AI Consumption**:
How much a Client uses its AI Tools, expressed in AI Tokens where the AI Tool exposes them and in activity otherwise.
_Avoid_: Usage, spend, tokenomics

**Metered**:
Describes an AI Tool the Client pays for per AI Token consumed.
_Avoid_: Pay-as-you-go, API-based, consumption-based

**Seat-licensed**:
Describes an AI Tool the Client pays for per user, regardless of how many AI Tokens that user consumes.
_Avoid_: Subscription, per-user, licensed

**API Key**:
A credential that grants access to a Metered AI Tool; never called a token.
_Avoid_: Token, access token, secret

**User**:
A Client person to whom AI Consumption is attributed, typically holding a seat in a Seat-licensed AI Tool.
_Avoid_: Employee, seat, account

**Workload**:
An application, integration or automation that consumes a Metered AI Tool, identified by the API Key, project or workspace it uses.
_Avoid_: App, service, project

**Unmatched Account**:
An account in an AI Tool that cannot be linked to a User in the Client's directory, such as a personal email or a leaver's account.
_Avoid_: Orphan, unknown user

**Cost Centre**:
A Client-defined organizational unit to which Users and Workloads belong, used to split AI Consumption and cost.
_Avoid_: Department, team, business unit

## Cost

**Rate Card**:
The Provider-maintained price per AI Token for each model and token type, with Client-specific discounts applied.
_Avoid_: Price list, pricing table

**Estimated Cost**:
The cost of AI Consumption calculated from a Rate Card or from seat prices, as opposed to what a Publisher actually invoices.
_Avoid_: Spend, actual cost, bill

## Optimization

**Optimization Opportunity**:
A detected case where a Client could spend less on AI Tools without losing value; distinct from an Exception.
_Avoid_: Finding, waste, Exception, recommendation

**Idle Seat**:
An Optimization Opportunity where a User holds a seat in a Seat-licensed AI Tool but shows little or no AI Consumption over a period.
_Avoid_: Unused licence, shelfware

**Duplicate Tools**:
An Optimization Opportunity where one User holds seats in several Seat-licensed AI Tools that serve the same purpose.
_Avoid_: Overlap, redundant licences

**Dismissal**:
A recorded decision, with a reason, not to act on an Optimization Opportunity; it holds while the situation stays the same.
_Avoid_: Ignore, won't fix, exemption

**Realised Saving**:
The saving from an actioned Optimization Opportunity, once later AI Consumption data confirms it.
_Avoid_: Saving (bare), benefit, cost avoidance
