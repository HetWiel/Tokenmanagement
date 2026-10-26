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

**Member**:
A person who signs in to Token Management: a Delivery Manager, Consultant, Client Approver or Client Viewer.
_Avoid_: User (that is the person AI Consumption is attributed to), login, account

**Publisher**:
A company whose AI Tools a Client pays for, such as Microsoft, GitHub, OpenAI or Anthropic.
_Avoid_: Vendor, provider, supplier

## Service

**Managed Service**:
An ongoing service the Provider delivers to a Client under one contract, covering one or more Service Lines.
_Avoid_: Engagement, contract

**Service Line**:
A discipline the Provider offers as a Managed Service, such as SAM, FinOps or AI Consumption Management.
_Avoid_: Practice, offering, module

**Demo Client**:
A fictitious Client with synthetic AI Consumption, used to demonstrate Token Management and kept apart from real Clients.
_Avoid_: Test client, sandbox

**AI Consumption Management**:
The Service Line that measures a Client's AI Consumption across AI Tools, estimates its cost and finds where it is wasted.
_Avoid_: AI FinOps, AI SAM, tokenomics

## Consumption

**AI Tool**:
An AI product a Client pays for, Metered, Seat-licensed or both, such as the Anthropic API, Azure OpenAI or Microsoft 365 Copilot.
_Avoid_: AI app, AI service, LLM

**Tool Category**:
A Provider-defined group of AI Tools that serve the same purpose, such as general assistant or coding assistant.
_Avoid_: Tool type, class, segment

**AI Token**:
The unit of text a model reads or writes, by which AI usage is measured and Metered AI Tools are priced.
_Avoid_: Token (bare), credit

**AI Consumption**:
How much a Client uses its AI Tools, expressed in AI Tokens where the AI Tool exposes them and in activity otherwise.
_Avoid_: Usage, spend, tokenomics

**Metered**:
Describes AI Consumption the Client pays for per Metered Unit consumed; an AI Tool can be partly Metered, such as GitHub Copilot's premium requests on top of its Seats.
_Avoid_: Pay-as-you-go, API-based, consumption-based

**Metered Unit**:
The unit in which a Publisher charges Metered AI Consumption, such as an AI Token of one token type, a premium request or a credit.
_Avoid_: Meter, SKU, billing unit

**Seat-licensed**:
Describes an AI Tool, or the part of one, that the Client pays for per User and period, regardless of how much that User consumes.
_Avoid_: Subscription, per-user, licensed

**Seat**:
One User's paid assignment to a Seat-licensed AI Tool, from the day it is assigned until the day it is released.
_Avoid_: Licence, user (for the assignment itself)

**API Key**:
A credential that grants access to a Metered AI Tool; never called a token.
_Avoid_: Token, access token, secret

**User**:
A Client person to whom AI Consumption is attributed, typically holding a Seat in a Seat-licensed AI Tool.
_Avoid_: Employee, seat, account

**Workload**:
An application, integration or automation that consumes a Metered AI Tool, identified by the API Key, project or workspace it uses.
_Avoid_: App, service, project

**Tool Account**:
An identity inside one AI Tool, such as a GitHub login or a member of an Anthropic organization, which is linked to a User or else is an Unmatched Account.
_Avoid_: Vendor account, AI account, identity

**Last Activity Date**:
The most recent day on which a Publisher reports any AI Consumption for a Tool Account.
_Avoid_: Last login, last seen, last used

**Unmatched Account**:
An account in an AI Tool that cannot be linked to a User in the Client's directory, such as a personal email or a leaver's account.
_Avoid_: Orphan, unknown user

**Cost Centre**:
A Client-defined organizational unit to which Users and Workloads belong, used to split AI Consumption and cost.
_Avoid_: Department, team, business unit

## Collection

**Connector**:
The part of Token Management that pulls AI Consumption from one AI Tool's API, the same for every Client.
_Avoid_: Integration, plugin, adapter

**Connection**:
One Client's authorised access to its account in one AI Tool, such as its Microsoft tenant, GitHub organization or Anthropic organization, through which a Connector pulls that Client's AI Consumption.
_Avoid_: Data source, integration, link

**Collection Run**:
One execution of a Connector for one Connection over a range of days, either the daily pull or part of a Backfill.
_Avoid_: Sync, import, job

**Backfill**:
The Collection Runs that pull a Connection's history, as far back as the AI Tool's API allows, when the Connection is first made.
_Avoid_: Historic import, initial load

## Cost

**Rate Card**:
The Provider-maintained price per Metered Unit for each model and token type, in the Publisher's currency, with Client-specific discounts applied.
_Avoid_: Price list, pricing table

**Seat Price**:
What a Client pays per Seat and period for a Seat-licensed AI Tool under its contract with the Publisher.
_Avoid_: Licence cost, list price

**Reporting Currency**:
The one currency in which a Client sees Estimated Cost, converted from the Publisher's currency at a recorded monthly exchange rate.
_Avoid_: Contract currency, display currency

**Estimated Cost**:
The cost of AI Consumption calculated from a Rate Card or from Seat Prices, as opposed to what a Publisher actually invoices.
_Avoid_: Spend, actual cost, bill

## Optimization

**Optimization Opportunity**:
A detected case where a Client could spend less on AI Tools without losing value; distinct from an Exception.
_Avoid_: Finding, waste, Exception, recommendation

**Idle Seat**:
An Optimization Opportunity where a User holds a Seat in a Seat-licensed AI Tool but shows little or no AI Consumption over a period.
_Avoid_: Unused licence, shelfware

**Duplicate Tools**:
An Optimization Opportunity where one User holds Seats in several Seat-licensed AI Tools of the same Tool Category.
_Avoid_: Overlap, redundant licences

**Dismissal**:
A recorded decision, with a reason, not to act on an Optimization Opportunity; it holds while the situation stays the same.
_Avoid_: Ignore, won't fix, exemption

**Realised Saving**:
The saving from an actioned Optimization Opportunity, once later AI Consumption data confirms it.
_Avoid_: Saving (bare), benefit, cost avoidance
