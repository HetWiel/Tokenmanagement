# Token Management v1 scope

Agreed during the design session of 2026-10-26. The vocabulary is in [GLOSSARY.md](../GLOSSARY.md) and the hard-to-reverse decisions are in [docs/adr](adr/).

## Goal

Measure each Client's AI Consumption across Metered and Seat-licensed AI Tools from a software asset management perspective: who uses which AI Tool, what it costs, and where money is wasted. It is the first tool of the new AI Consumption Management Service Line.

## In v1

### Parties and access
- One Provider, many Clients, with strict isolation per Client (ADR 0004, following Delivery Platform ADR 0001).
- Provider roles: Delivery Manager and Consultant. Consultants run connectors, resolve Unmatched Accounts, maintain Rate Cards and propose Optimization Opportunities. Delivery Managers can also change per-Client settings.
- Client roles: Client Approver (marks Optimization Opportunities as Actioned or Dismissed, maintains the Cost Centre mapping) and Client Viewer (read-only).
- Client users sign in through their own SSO with a magic-link fallback, as in the Delivery Platform (ADR 0004).
- A Demo Client with synthetic data, kept fully apart from real Clients.

### Data collection
- Usage is pulled from vendor APIs with read-only access. The tool never sits in AI traffic (ADR 0002).
- Connectors: Microsoft 365 Copilot, GitHub Copilot, ChatGPT Enterprise, Azure OpenAI and the Anthropic API. Other AI Tools come in by manual file upload.
- Microsoft sources use admin consent to one multi-tenant Entra app. Other API keys are stored per Client in Azure Key Vault (ADR 0005).
- Pulled daily, with a backfill as far as each API allows when a Client connects.
- Stored as daily aggregates per User or Workload, per AI Tool and per model, with no prompt or response content (ADR 0003). Daily data is kept for 25 months, then rolled up to monthly.

### Attribution
- Users come from the Client's Entra ID, matched by UPN or email. Accounts that cannot be matched become Unmatched Accounts for a Consultant to resolve.
- Workloads are identified by the API key, project or workspace of a Metered AI Tool.
- Users and Workloads belong to a Cost Centre, which defaults to the Entra department attribute.
- Per User, the tool shows only seats held, active or inactive status, and last activity date. AI Token volumes and cost are shown per Cost Centre, Workload and AI Tool, never per User (ADR 0003).

### Cost
- Estimated Cost comes from a Provider-maintained Rate Card (price per AI Token per model and token type, with Client-specific discounts) for Metered AI Tools, and from seats × contract seat price for Seat-licensed AI Tools.
- Rate Cards are kept in the Publisher's currency. Cost is shown in the Client's contract currency at a recorded monthly exchange rate.

### Optimization
- Detected Optimization Opportunities: Idle Seat and Duplicate Tools.
- Idle Seat by default means no AI Consumption in 30 days, excluding seats assigned in that period. The period and threshold are configurable per Client.
- Lifecycle: Open, Proposed, then Actioned or Dismissed (with a reason), then Realised once later data confirms the saving. A Dismissal holds while the situation stays the same.

### Cross-cutting
- TypeScript (Next.js and Postgres) on Azure West Europe, the same stack as the Delivery Platform (ADR 0001).

## Later

- Integration with the Delivery Platform, which becomes the Clients' single interface (ADR 0004). Token Management sends KPI values (Estimated AI Cost per month, % Idle Seats, Realised Savings to date), and Realised Savings become proposed Achievements.
- Optimization Opportunities for a seat versus metered mismatch and for an oversized model.
- Reconciliation of Estimated Cost against invoices.
- Budgets, alerts and chargeback to Cost Centres.
- More connectors, such as the OpenAI API, Claude Team/Enterprise and Bedrock, and collection through the ITAM Agent for Clients that will not grant access.
