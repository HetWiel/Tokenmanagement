---
status: accepted, amended by ADR 0006 (GitHub App; unscoped Anthropic Admin API key)
---

# Admin consent to one multi-tenant Entra app for Microsoft sources; other API keys in Key Vault

Token Management reads Microsoft sources (Microsoft 365 Copilot usage, Azure OpenAI cost and the Client's Entra ID directory) through one Provider-owned multi-tenant Entra app, to which each Client grants read-only admin consent. No secret is shared, and the Client can revoke consent at any time. Other AI Tools (GitHub Copilot, ChatGPT Enterprise, the Anthropic API) have no equivalent, so their read-only admin API keys are entered by a Client Approver or Consultant and stored per Client in Azure Key Vault, never in the database. We rejected asking Clients for Microsoft credentials or service principal secrets, because enterprise Clients refuse to hand them over and they expire silently.
