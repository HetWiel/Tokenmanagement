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
