---
status: accepted
---

# GitHub through a Provider GitHub App; the Anthropic Admin API key accepted although it cannot be read-only

This amends ADR 0005, which assumed that GitHub Copilot, ChatGPT Enterprise and the Anthropic API all need a read-only admin API key stored in Key Vault. Two facts found during implementation planning (2026-10-26) differ.

GitHub has an equivalent of Entra admin consent. Each Client installs one Provider-owned GitHub App on its organization or enterprise, with read-only permissions for Copilot seats, Copilot usage metrics and billing usage. The Copilot metrics, seat and billing usage endpoints all accept GitHub App installation tokens. No secret is shared, the Client sees exactly what is granted and can uninstall at any time, and short-lived installation tokens replace personal access tokens that expire silently. Only the App's own private key is stored, once for the Provider, in Key Vault.

The Anthropic Usage and Cost API accepts only Admin API credentials, and a Claude Console Admin API key cannot be scoped: it can also manage members, workspaces and API keys. We accept this rather than drop the connector, with these mitigations:
- The Client creates a dedicated key with an expiry date for Token Management, and is told in writing that it is a full admin key and which endpoints we call.
- The web app can only write and delete the secret in Key Vault. Only the worker's managed identity can read it.
- The Anthropic Connector calls only the usage report, cost report and Claude Code analytics endpoints.

Claude Enterprise organizations instead use a scoped `read:analytics` key, which is read-only. ChatGPT Enterprise uses a workspace-scoped Admin key with a read-only analytics scope, so ADR 0005 holds for it unchanged.

## Considered options

- Fine-grained personal access tokens for GitHub: rejected because they are tied to one person, expire silently and need an organization owner to create them.
- Dropping the Anthropic API Connector, or taking its data by manual upload only: rejected because it is one of the five v1 Connectors and Clients would have to export by hand every month.
