---
status: accepted
---

# Client sign-in through Microsoft Entra External ID; Provider staff through the Provider's own Entra ID

ADR 0004 adopts the Delivery Platform's sign-in unchanged: each Client signs in through its own SSO, with a magic-link fallback (Delivery Platform ADR 0002). The Delivery Platform has not built it yet, so Token Management builds it first and the Delivery Platform reuses it. Client sign-in runs through one Provider-owned Microsoft Entra External ID tenant in the EU. That tenant federates each Client's own identity provider over OpenID Connect or SAML, configured per Client, and offers email one-time passcodes to Clients without SSO. Provider staff sign in with the Provider's own workforce Entra ID, with Delivery Manager and Consultant roles taken from Entra groups. Token Management keeps its own record of each Member (role, Client) and grants access only to invited Members.

We chose a managed identity service because per-Client SSO, SAML handling and passwordless email are security-critical and costly to maintain ourselves, and Entra External ID keeps that data in our Azure tenant in the EU. It does not tie Clients to Microsoft: any OIDC or SAML identity provider can be federated, which was the Delivery Platform's objection to Entra guest accounts.

## Consequences

- The fallback is an emailed one-time passcode rather than a magic link. The Delivery Platform should record the same change when it adopts this.
- Sign-in code lives in its own module (`src/auth`) so the Delivery Platform can lift it unchanged.
- Each new Client with SSO needs a federation set up in the External ID tenant. This is a Delivery Manager onboarding step with a checklist.

## Considered options

- Building sign-in ourselves in Next.js (an OIDC client per Client, a SAML library, our own magic links): rejected because it is security-critical code to own and maintain.
- A third-party identity service such as WorkOS or Auth0: rejected because it puts a non-Azure processor in the sign-in path of every Client and is priced per SSO connection.
