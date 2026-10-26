# Token Management v1: implementation plan

Written on 2026-10-26, after the design session. It builds on [GLOSSARY.md](../GLOSSARY.md), [docs/adr](adr/) (0001–0009) and [spec-v1.md](spec-v1.md), and uses their terms. Where this plan and those documents disagree, the documents win and this plan is wrong.

Planning only: no application code exists yet.

## 0. Decisions taken while planning

These came up during planning. Each is recorded where it belongs.

| Decision | Where |
| --- | --- |
| GitHub connects through a Provider GitHub App rather than a personal token. The Anthropic Admin API key is accepted although it cannot be read-only, with mitigations. | ADR 0006 (amends 0005) |
| The Client turns off concealed names in Microsoft 365 reports before connecting Microsoft 365 Copilot. | ADR 0007 |
| Client sign-in runs through Entra External ID, with OIDC/SAML federation per Client and an email one-time passcode as fallback. Provider staff use the Provider's own Entra ID. | ADR 0008 |
| Hosting is two Azure Container Apps (web, worker) from one image. Scheduling and retries use pg-boss in Postgres. | ADR 0009 |
| An AI Tool can be both Metered and Seat-licensed. The Rate Card prices Metered Units (AI Tokens by token type, premium requests, credits). | GLOSSARY, spec |
| Reporting Currency replaces "contract currency", matching the Delivery Platform's `clients.reporting_currency`. | GLOSSARY, spec |
| The walking skeleton carries the Microsoft 365 Copilot Connector. | §2 |
| New terms: Member, Publisher, Service Line, Tool Category, Metered Unit, Seat, Tool Account, Last Activity Date, Connector, Connection, Collection Run, Backfill, Seat Price, Reporting Currency. | GLOSSARY |

"Member" needs a word of explanation. The glossary already uses **User** for the Client person that AI Consumption is attributed to. The people who sign in to Token Management therefore need another name, or `users` would mean two things. The Delivery Platform calls its sign-in table `users`. The later integration maps our `members` onto it, which is a rename and nothing more.

## 1. Architecture

### 1.1 Shape

One repository and one Next.js + TypeScript codebase, using the same versions and conventions as the Delivery Platform: Next.js 16, React 19, Drizzle with postgres.js, zod 4, next-intl, Vitest, ESLint and Prettier, and `npm run check`. It is built into one container image that runs in two ways (ADR 0009):

- **web**: Next.js pages, server actions, sign-in, OAuth and consent callbacks.
- **worker**: the pg-boss job runner, which handles Collection Runs, directory sync, cost recalculation, Optimization Opportunity detection, notifications and retention.

Both talk only to Postgres, Key Vault and outbound vendor APIs. The worker has no inbound endpoint.

### 1.2 Modules (bounded areas)

The layout follows the Delivery Platform (`src/db`, `src/domain`, `src/server`, `src/app`), plus `src/connectors`, `src/jobs` and `src/auth`. Each module owns its tables and exposes functions that take a `ClientScope`. The arrows show who reads whom.

| Module | Owns | Depends on |
| --- | --- | --- |
| **Access** (`src/auth`, `src/server/access.ts`) | Members, Client assignments, roles, `ClientScope`, sign-in (ADR 0008) | none |
| **Clients and service** | Clients (Demo Client flag, Reporting Currency, Idle Seat settings), Managed Services with end-of-service dates | Access |
| **Directory** | Users, Cost Centres, Tool Accounts, Unmatched Account resolution | Clients |
| **Collection** | Connectors, Connections, Collection Runs, credentials (Key Vault / GitHub App / Entra), scheduling | Directory, Consumption (ingest only) |
| **Consumption** | Seats, daily activity, daily metered usage, Workloads, monthly rollups | Directory |
| **Cost** | Rate Cards, Client discounts, Seat Prices, exchange rates, Estimated Cost | Consumption |
| **Optimization** | Idle Seat and Duplicate Tools detection, Optimization Opportunity lifecycle, Dismissals, Realised Savings | Consumption, Cost, Directory |
| **Views** | Client landing page, cross-client Provider view, connection health | all (read only) |
| **Notifications** | Email (immediate or weekly digest) per Member preference, outbox | Optimization, Collection, Directory |
| **Audit** | Append-only `audit_events`, as in the Delivery Platform | all (write only) |
| **Delivery Platform export** (later, shapes only) | Pure functions that produce KPI values and Achievement drafts | Cost, Optimization |

Rules that hold across modules:

1. **Connectors never write tables.** A Connector returns normalised records. One shared ingest function writes them, scoped to the Connection's Client. This keeps Client isolation and idempotency in one place.
2. **Nothing per User leaves the Consumption module except SAM facts (ADR 0003).** Per-User read functions return only a `SeatFacts` type (Seats held, active or inactive, Last Activity Date). Token volumes and cost are only queryable grouped by Cost Centre, Workload or AI Tool. A test enforces this (§4.4).
3. **Every change to Client data writes an audit event in the same transaction**, using `recordAudit` as in the Delivery Platform.

### 1.3 Postgres data model

Conventions are copied from the Delivery Platform schema. Every Client-owned table has `client_id`. References between Client-owned rows use composite `(client_id, id)` foreign keys. A trigger covers references to rows that may be shared. Money shown to Clients is `numeric(14,2)` in the Reporting Currency, the same as the Delivery Platform's `achievements.value` and `kpi_values.value`. Internal quantities and Publisher-currency amounts are `numeric(20,6)`. Dates are `date` in UTC. Months are the first day of the month, matching `kpi_values.period`.

**Provider catalogue (shared, no `client_id`)**

| Table | Key columns | Notes |
| --- | --- | --- |
| `service_lines` | code, name | `ai_consumption_management`. Same shape as in the Delivery Platform. |
| `publishers` | name | Microsoft, GitHub, OpenAI, Anthropic. Same shape as in the Delivery Platform. |
| `tool_categories` | code, name | `general_assistant`, `coding_assistant`, `api_platform`. |
| `ai_tools` | code, publisher_id, tool_category_id, name, is_seat_licensed, is_metered, connector_code (nullable for manual-upload tools) | e.g. `m365_copilot` (seat), `github_copilot` (seat + metered), `chatgpt_enterprise` (seat + metered credits), `azure_openai` (metered), `anthropic_api` (metered) |
| `models` | ai_tool_id, publisher_model_id, display_name | Publisher model ids normalised, e.g. `claude-opus-5-5`, `gpt-5`. Unknown ids are created on ingest and flagged for a Consultant. |
| `rate_cards` | publisher_id, currency, name | One per Publisher price list, e.g. "Anthropic list prices", "Azure OpenAI West Europe Global". |
| `rate_card_prices` | rate_card_id, model_id, metered_unit, price_dimensions jsonb (service tier, context window band, inference geo, batch), price_per_unit, unit_size (e.g. 1,000,000), effective_from, effective_to | Effective-dated. A change creates a new row and an audit event. Never updated in place. |
| `exchange_rates` | month, from_currency, to_currency, rate, source | Recorded monthly (spec). |

`metered_unit` is an enum: `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_5m_tokens`, `cache_write_1h_tokens`, `premium_requests`, `ai_credits`, `chatgpt_credits`, and others as Connectors need them.

**Access and service (Client-owned unless noted)**

| Table | Key columns | Notes |
| --- | --- | --- |
| `clients` | slug, name, reporting_currency, is_demo, idle_seat_days (30), idle_seat_threshold (0 activity days), external_ref (nullable; the future Delivery Platform client id) | Same shape as in the Delivery Platform, plus Idle Seat settings. |
| `managed_services` | client_id, starts_on, ends_on, read_only_until | Service Line `ai_consumption_management`. End of service is driven from here. |
| `members` | name, email, role (`delivery_manager`, `consultant`, `client_approver`, `client_viewer`), client_id (null for Provider staff), identity_issuer, identity_subject, notification_mode (`immediate` / `weekly_digest`) | The Delivery Platform's `users`, renamed (§0). |
| `client_assignments` | client_id, member_id | As in the Delivery Platform. |
| `client_identity_providers` | client_id, kind (`oidc` / `saml` / `email_otp`), external_id_federation_ref, email_domains[] | Which federation in External ID a Client's Members are sent to. |

**Directory**

| Table | Key columns | Notes |
| --- | --- | --- |
| `cost_centres` | client_id, name, source (`entra_department` / `manual`) | The default comes from the Entra `department` attribute (spec). |
| `users` | client_id, entra_object_id, user_principal_name, email, display_name, cost_centre_id, account_enabled, last_seen_in_directory_on | Glossary **User**. Synced daily from Entra ID. Leavers are kept, flagged by `account_enabled` and `last_seen_in_directory_on`. |
| `tool_accounts` | client_id, connection_id, publisher_account_id, login, email, user_id (nullable), match (`upn` / `email` / `saml_identity` / `manual` / `unmatched` / `not_a_person`), resolved_by, resolved_at | An **Unmatched Account** is a row with `match = 'unmatched'`. Resolving it writes an audit event (spec). `not_a_person` is for bots and service accounts. |
| `cost_centre_overrides` | client_id, user_id or workload_id, cost_centre_id | Client Approver mapping (spec). Audited. |

**Collection**

| Table | Key columns | Notes |
| --- | --- | --- |
| `connections` | client_id, ai_tool_id, publisher_account_ref (tenant id / org / enterprise slug / workspace id / Anthropic org id), auth_kind (`entra_consent` / `github_app` / `key_vault_secret` / `manual_upload` / `synthetic`), auth_ref (installation id, or Key Vault secret *name*, never the value), status (`pending` / `healthy` / `needs_action` / `failing` / `revoked`), status_reason, complete_through (date), backfill_from (date), last_success_at | A check constraint (plus a trigger on `clients.is_demo`) keeps `synthetic` for the Demo Client only, and the Demo Client on `synthetic` only. |
| `collection_runs` | client_id, connection_id, kind (`daily` / `backfill` / `upload`), window_from, window_to, status, attempt, error_code, error_detail (redacted), stats jsonb, started_at, finished_at | One row per attempt. The source of the "failed connectors" view. |
| `uploads` | client_id, connection_id, file blob ref, uploaded_by, row_count, status | For manual-upload AI Tools. The file is kept 90 days, then deleted. |

**Consumption** (ADR 0003: daily aggregates only, never content, no per-call rows)

| Table | Grain | Notes |
| --- | --- | --- |
| `workloads` | client_id, connection_id, kind (`api_key` / `project` / `workspace` / `deployment`), publisher_ref, name, cost_centre_id | Metered consumers. Usage with no key or no workspace (Anthropic Console playground, default workspace) goes to a per-Connection "Unattributed" Workload. |
| `seats` | client_id, connection_id, tool_account_id, plan, assigned_on, released_on | One row per assignment interval, so seat counts can be read for any day. Derived from daily seat snapshots. |
| `activity_days` | client_id, connection_id, tool_account_id, day, active, interaction_count (nullable) | Seat-licensed activity per Tool Account and day. Holds no AI Token volume. |
| `last_activity` | client_id, connection_id, tool_account_id, last_activity_on, as_reported_on | What the Publisher says the Last Activity Date is. It is the only history Microsoft 365 Copilot can backfill. |
| `usage_days` | client_id, connection_id, day, workload_id (nullable), tool_account_id (nullable), model_id, metered_unit, price_dimensions jsonb, quantity, publisher_amount (nullable), publisher_currency (nullable) | Metered consumption. Exactly one of workload or tool account is set. `publisher_amount` is what the Publisher's API reports, kept for later invoice reconciliation, never shown in v1. |
| `usage_months` / `activity_months` | the same keys per month | Rollups. Daily rows older than 25 months are rolled up and deleted (spec). |

Ingest is **replace by window**. A Collection Run for Connection C over days [d1, d2] deletes C's rows in that window and inserts the new ones in one transaction. Re-running is therefore always safe, and late vendor corrections overwrite earlier values.

**Cost**

| Table | Notes |
| --- | --- |
| `client_rate_discounts` | client_id, rate_card_id, ai_tool_id or model_id, percent, effective_from/to. Audited. |
| `seat_prices` | client_id, ai_tool_id, plan, price, currency, per (`month` / `year`), effective_from/to. Audited. |
| `estimated_cost_days` | client_id, day, ai_tool_id, cost_centre_id, workload_id (nullable), amount_publisher_currency, amount_reporting_currency. A derived table: the worker recalculates whole months whenever usage, a price, a discount or an exchange rate changes. Never per User (ADR 0003). |

**Optimization**

| Table | Notes |
| --- | --- |
| `optimization_opportunities` | client_id, kind (`idle_seat` / `duplicate_tools`), status (`open` / `proposed` / `actioned` / `dismissed` / `realised`), user_id, seat_ids[], ai_tool_ids[], situation_fingerprint, detected_on, estimated_monthly_saving (Reporting Currency), proposed_by/at, decided_by/at, dismissal_reason, realised_on, realised_saving. |
| `realised_savings` | client_id, optimization_opportunity_id, month, amount. One row per month confirmed by later data. The shape a Delivery Platform Achievement (`cost_savings`) and the "Realised Savings to date" KPI read from. |

A Dismissal "holds while the situation stays the same" (glossary). `situation_fingerprint` is a hash of what made the Opportunity: the User, the Seats and AI Tools involved, and for Duplicate Tools the Tool Category. Detection skips a fingerprint that is already Dismissed. A new Seat or a released Seat changes the fingerprint and reopens the case.

**Cross-cutting**: `audit_events` (identical to the Delivery Platform's, subjects `optimization_opportunity`, `tool_account`, `rate_card`, `seat_price`, `connection`, `client_settings`), `notification_outbox`, and `pgboss.*` (owned by pg-boss).

**Delivery Platform compatibility.** No tables are needed for it in v1. A pure function `monthlyKpiValues(scope, month)` returns `{ kpiCode, period, value, unit }` rows matching the Delivery Platform's `kpi_values` (period = first of month, `numeric(14,2)`, units `money` / `percent`) for:
- `ai_estimated_cost_month` (money, lower is better),
- `ai_idle_seat_percent` (percent, lower is better),
- `ai_realised_savings_to_date` (money, higher is better).

`achievementDraft(realisedSaving)` returns the Delivery Platform's Achievement shape: category `cost_savings`, value in Reporting Currency, `realised_on`, Publisher. Both are unit-tested against copies of the Delivery Platform's zod shapes. Nothing is sent in v1.

### 1.4 Connector framework

```ts
interface Connector {
  code: AiToolCode;                   // "m365_copilot", ...
  authKind: AuthKind;
  capabilities: { seats: boolean; activity: boolean; metered: boolean; lastActivityOnly?: boolean };
  limits: { backfillDays: number | "last_activity_only"; lagDays: number; relookDays: number;
            maxDaysPerCall: number; concurrency: number };
  check(ctx): Promise<ConnectionCheck>;               // healthy | needs_action(reason) | failing(error)
  collect(ctx, window: DayRange): AsyncIterable<NormalisedBatch>;
}
// NormalisedBatch = ToolAccount[] | SeatSnapshot[] | ActivityDay[] | LastActivity[] | UsageDay[] | Workload[]
```

- **Adapters** live in `src/connectors/<code>/`. Each one is `client.ts` (HTTP calls), `schemas.ts` (zod schemas of vendor responses), `normalise.ts` (a pure mapping to `NormalisedBatch`) and `fixtures/` (§4.3). Adapters get an injected `VendorHttp` and credentials provider, so tests swap in fixtures.
- **VendorHttp** is shared. It adds timeouts, retries on 429 and 5xx with exponential backoff and jitter (honouring `Retry-After`), per-Publisher rate limiting, a `User-Agent` header, and redaction of credentials and email addresses from logs and errors. It maps errors to `credentials_expired`, `consent_revoked`, `names_concealed`, `forbidden`, `rate_limited`, `vendor_down` and `schema_changed`.
- **Scheduling** uses pg-boss:
  - At 03:15 Europe/Amsterdam, the cron job `plan-daily` enqueues one `collect` job per active Connection, with `singletonKey = connection_id` so they never overlap.
  - Queues are per Connector. Their concurrency comes from `limits.concurrency`. For example, Anthropic's Usage API is limited per organization, and the Claude Enterprise Analytics API allows 60 requests per minute.
  - Directory sync (Entra users, departments, licences) runs before collection for every Client with Microsoft consent.
- **Windows**: a daily run collects `[complete_through - relookDays, today - lagDays]`. Re-looking at recent days catches late and revised vendor data, such as Anthropic Enterprise cost revisions up to about 7 days after month end, or Azure cost data landing late. After a successful run, `complete_through` advances.
- **Backfill**: when a Connection first becomes healthy, it is split into `backfill` jobs of `maxDaysPerCall` (Anthropic 31 daily buckets, GitHub 1 day per report, Azure one month). The jobs run newest first, so recent data shows within minutes and the full history follows. They use a lower priority than daily jobs.
- **Retries and failure**: pg-boss retries a job 5 times with exponential backoff (1 min up to about 2 h). Errors that need the Client to act (`credentials_expired`, `consent_revoked`, `names_concealed`) don't retry. They set the Connection to `needs_action` and notify the Consultant at once. Other errors set it to `failing` after the last retry. The Consultant is notified when a Connection has failed for 24 hours, not on the first transient error.
- **After ingest**: the run enqueues `recalculate-cost(client, months touched)` and then `detect-opportunities(client)`. Both are idempotent.
- **Demo Client**: a `synthetic` Connector produces plausible data for all five AI Tools through the same ingest path, with dates relative to today, like the Delivery Platform's seed. The Demo Client therefore shows exactly what the code really computes, and never has real credentials.
- **Manual upload**: an `upload` Connector parses a Provider CSV template (Tool Account email, Seat, Last Activity Date, optional metered quantities per month) into the same batches.

### 1.5 Credentials

| Connector | Credential | Where it lives |
| --- | --- | --- |
| Microsoft 365 Copilot, Entra directory, Azure OpenAI | Admin consent to the Provider's multi-tenant Entra app (ADR 0005). App-only tokens per Client tenant. For Azure OpenAI, the Client also assigns *Cost Management Reader* and *Monitoring Reader* on the relevant subscriptions to our enterprise app. | The app authenticates with a **federated identity credential backed by the worker's managed identity**, so there is no Provider secret either. The fallback is a certificate in Key Vault. |
| GitHub Copilot | Installation of the Provider GitHub App (ADR 0006) | The App private key is in Key Vault once. 1-hour installation tokens are minted per run. |
| ChatGPT Enterprise | Workspace-scoped Admin key with the analytics read scope, entered by a Client Approver or Consultant | Key Vault secret `conn-<connection uuid>`. |
| Anthropic API | Admin API key (Console), or an Analytics key (`read:analytics`) for Claude Enterprise | Key Vault, as above (ADR 0006 mitigations). |

The web identity has a custom Key Vault role that can only set and delete secrets. Only the worker identity can read them. Key entry forms post straight to a server action that writes to Key Vault and stores only the secret name. The value never touches Postgres, logs or audit details. At end of service, the Connection's secret is deleted and purged, and the GitHub App and Entra consent are flagged for the Client to remove. We can't uninstall or unconsent for them, so the checklist and an email say so.

### 1.6 Auth and tenancy

- **Tenancy**: identical to the Delivery Platform (its ADR 0001). Client isolation is the only boundary, enforced in the database (composite FKs and triggers) and in queries (`ClientScope` handed out only by `openClientScope`).
  - The worker opens a scope per Connection through `openSystemScope(connectionId)`. That scope can only target the Connection's own Client.
  - Postgres row-level security is *not* added in v1, to stay identical to the Delivery Platform. It is noted as a possible hardening later.
- **Sign-in** (ADR 0008): two OIDC issuers, both handled by the `src/auth` module.
  - Client Members sign in through the Entra External ID tenant, which routes them by email domain to their Client's federated IdP, or to email one-time passcode.
  - Provider staff sign in through the Provider's workforce tenant, with roles taken from Entra groups `TM-DeliveryManagers` and `TM-Consultants`.
  - Sessions are encrypted HTTP-only cookies.
  - A Member must be invited (a `members` row exists) before sign-in succeeds. First sign-in binds the issuer and subject.
  - The dev user switcher from the Delivery Platform is kept for local development and tests only.
- **Roles**: permissions follow the spec in `src/domain/permissions.ts`.
  - Consultants run Connectors, resolve Unmatched Accounts, maintain Rate Cards and propose Opportunities.
  - Delivery Managers can also change Client settings.
  - Client Approvers action or dismiss Opportunities and maintain the Cost Centre mapping. They can also enter API keys, which the spec doesn't forbid.
  - Client Viewers have read-only access.
- **End of service**: once `managed_services.ends_on` passes, the Client's Members get read-only access until `read_only_until` (90 days). Connections are revoked at once. A retention job purges data after 7 years (the spec marks these periods as still to be checked against contracts).

### 1.7 Azure resources (West Europe, one resource group per environment: `staging`, `prod`)

| Resource | SKU / setting | Purpose |
| --- | --- | --- |
| Container Apps environment (workload profiles, VNet-integrated) | Consumption profile | Hosts `tm-web` (external ingress, custom domain, managed certificate, min 1 replica) and `tm-worker` (no ingress, 1 replica in v1) |
| Azure Container Registry | Basic | The single image |
| Azure Database for PostgreSQL Flexible Server | PG 17. Staging: Burstable B2s. Prod: General Purpose D2ds_v5 with zone-redundant HA. 35-day backups, geo-backup to North Europe (EU). | All data, including pg-boss queues. Private access in the VNet, Entra auth for the managed identities. |
| Key Vault | Standard, RBAC mode, soft delete + purge protection, private endpoint | Per-Client API keys, GitHub App private key, Entra certificate fallback |
| User-assigned managed identities | `id-tm-web`, `id-tm-worker` | Postgres, Key Vault, the Entra federated credential |
| Storage account | Standard LRS, private | Manual uploads (deleted after 90 days) |
| Azure Communication Services + Email | EU data location, custom sender domain | Notifications and digest |
| Log Analytics + Application Insights | OpenTelemetry from web and worker, PII scrubbing processor | Tracing, Collection Run metrics, alerts (failed runs, job backlog, 5xx) |
| Entra app registrations | Multi-tenant "Token Management Connector" (Graph application permissions: `Reports.Read.All`, `ReportSettings.Read.All`, `User.Read.All`, `Organization.Read.All`); a workforce sign-in app | ADR 0005, ADR 0008 |
| Entra External ID tenant | EU | Client sign-in (ADR 0008) |
| GitHub App (owned by the Provider's GitHub organization) | Read-only: Copilot metrics, Copilot seat management, billing/administration read as the endpoints require | ADR 0006 |

Infrastructure as code: Bicep in `infra/`, deployed by GitHub Actions through OIDC federation to Azure (no deployment secrets). There is no Azure Front Door in v1, because Container Apps ingress with a managed certificate is enough for one region.

## 2. Delivery plan (vertical slices)

Every slice ends deployed to staging and demoable on the Demo Client plus at least one real Connection. "Client zero" is the Provider's own Microsoft tenant, GitHub organization and Anthropic organization. It is a real Client in staging, so every Connector is first proven on data we are allowed to see.

### Slice 1: Walking skeleton (Microsoft 365 Copilot)

**Goal**: a Consultant and a Client Approver sign in to a deployed app. The Approver sees one view of real Microsoft 365 Copilot Seats for one Client, refreshed daily without anyone touching it. The Demo Client shows the same view with synthetic data.

**Scope**, in internal milestones that can be merged separately:
- 1a. Repo scaffold copied from the Delivery Platform's conventions, then CI, Bicep, staging deployment of web and worker, and the Demo Client with the `synthetic` Connector feeding Microsoft 365 Copilot data. The view is "Microsoft 365 Copilot Seats: active vs. idle (30 days), and per User the Seat, active or inactive, and Last Activity Date". This milestone signs in with the dev switcher on staging only.
- 1b. Real sign-in (ADR 0008): External ID with email OTP plus one federated IdP (Client zero's own Entra counts as an OIDC federation), and the Provider workforce tenant for staff. Invitations for Members.
- 1c. The Microsoft consent flow, done by a Consultant from the Connections page:
  - admin-consent URL, callback and status;
  - a concealed-names check, with needs-action guidance (ADR 0007);
  - Entra directory sync for Users, `department` as Cost Centre, and Copilot licence assignment with dates from `assignedPlans`.
- 1d. The Microsoft 365 Copilot Connector:
  - It takes a daily snapshot of `getMicrosoft365CopilotUsageUserDetail(period='D7', version='v2')`, which gives per-User Last Activity Date, prompts and active days.
  - Backfill uses `period='D180'` for Last Activity Dates only.
  - It also records `activity_days` from the day of connection, and runs the daily pg-boss schedule.

**Main risks**:
- The External ID setup and federation per Client take time. Mitigation: start on day 1, with the email OTP path first.
- Client zero may lack enough Microsoft 365 Copilot licences for a meaningful demo.
- The report's refresh lag (typically 2–3 days) confuses the demo. We show "data as of `reportRefreshDate`" everywhere.
- The skeleton is thick. 1a is a demoable checkpoint on its own if 1b–1d slip.

**Done when**:
- Staging runs on Bicep-provisioned resources, and CI is green on every PR.
- A Client zero Approver signs in through SSO and cannot reach the Demo Client, and the reverse. Isolation tests prove both.
- Three consecutive daily runs succeed unattended, and a revoked consent turns the Connection to needs-action with an email to the Consultant.
- The view matches the Microsoft 365 admin center's Copilot usage report for the same day.

### Slice 2: Idle Seats and the Optimization lifecycle

**Goal**: a Client Approver acts on a real Idle Seat and later sees its Realised Saving.

**Scope**:
- Seat Prices per Client and AI Tool, with Estimated Cost for Seats (Seats × Seat Price) in Reporting Currency.
- Exchange rates entered monthly by a Consultant.
- Idle Seat detection uses the Client's settings: by default no activity in 30 days, excluding Seats assigned in that period.
- The lifecycle runs Open → Proposed (Consultant) → Actioned or Dismissed with a reason (Client Approver) → Realised. A Seat counts as Realised once its release appears in data and stays released for a full month. Dismissal fingerprints are in place.
- Audit trail and history per Opportunity.
- Email on Proposed to Client Approvers, with weekly-digest preference.
- Landing page tiles: Estimated Cost per month (Seats only so far), active vs. idle Seats per AI Tool, Opportunities awaiting a decision, and Realised Savings this year.

**Risks**:
- False-positive Idle Seats, for example a User on leave. The Consultant's Proposed step is the guard.
- Realised Saving rules are subtle, for example a Seat released and then reassigned to someone else. They must be pinned down in unit tests first.

**Done when**:
- The full lifecycle has been demoed on the Demo Client and done once on Client zero.
- Each decision shows in the audit trail.
- A Dismissed Opportunity stays dismissed across daily runs until its situation changes, which a test proves.

### Slice 3: The Metered path (Anthropic API)

**Goal**: Estimated Cost per Workload and Cost Centre for a Metered AI Tool.

**Scope**:
- Key entry (write-only, Key Vault, ADR 0006 mitigations).
- The Anthropic Connector:
  - The Usage API in `1d` buckets grouped by `api_key_id`, `workspace_id`, `model`, `service_tier`, `context_window` and `inference_geo`, giving token types per Metered Unit.
  - API key and workspace lists, which become Workloads.
  - Cost report amounts stored as `publisher_amount`.
  - Claude Code Analytics per Tool Account, rolled up to Cost Centre only.
  - The Claude Enterprise Analytics API variant for Enterprise organizations.
- Rate Card maintenance UI (Provider), Client discounts, effective dating with audit, and cost recalculation.
- The Workload → Cost Centre mapping for Client Approvers.
- The Estimated Cost trend split by AI Tool and Cost Centre on the landing page.
- Connector failure and expired-credential emails.

**Risks**:
- Rate Card dimensions: prices differ by service tier, batch, context window band, data-residency multiplier and cache duration. The `price_dimensions` matching must be exact, with a "no price found" state that is visible rather than zero.
- The unscoped Admin key, if a Client refuses it (ADR 0006).

**Done when**: for Client zero, the Rate Card-based Estimated Cost per month is within 2% of Anthropic's cost report for the same month. This is a sanity check, not reconciliation. Every unpriced model or unit is listed for the Consultant.

### Slice 4: GitHub Copilot and Unmatched Accounts

**Goal**: GitHub Copilot Seats, activity and premium requests, with GitHub logins linked to Users.

**Scope**:
- GitHub App install flow (organization or enterprise).
- Seats from `/orgs/{org}/copilot/billing/seats`, with `last_activity_at` and `created_at` as the assignment date.
- Daily per-User activity from the `users-1-day` NDJSON reports. Backfill goes back up to 1 year and never before 2025-10-10.
- Premium requests and AI credits from the billing usage endpoints, by day, user and model (24 months), as Metered Units on the same AI Tool.
- Tool Account matching: verified email, then SAML external identity when the organization uses SAML SSO, otherwise Unmatched.
- An Unmatched Account resolution screen and an email to the Consultant when a new Unmatched Account appears.

**Risks**:
- GitHub logins often don't carry a corporate email, so Unmatched Accounts will be common for organizations without SAML or Enterprise Managed Users. Resolution must be fast, in bulk with suggestions.
- `last_activity_at` misses users with IDE telemetry turned off, so they look idle. Mitigation: combine it with the usage reports, and the Consultant reviews before proposing.
- Enterprise-level and org-level reports attribute users differently. We pick per Connection.

**Done when**:
- Client zero's GitHub Copilot Seats and Last Activity Dates match the GitHub UI.
- Every Tool Account is either matched or listed as Unmatched.
- Idle Seats from GitHub Copilot flow through the Slice 2 lifecycle unchanged.

### Slice 5: ChatGPT Enterprise and Duplicate Tools

**Goal**: ChatGPT Enterprise Seats and activity, and Duplicate Tools across Microsoft 365 Copilot and ChatGPT Enterprise.

**Scope**:
- First, a **two-day spike** with a real ChatGPT Enterprise workspace, Client zero's or a pilot Client's, to confirm the analytics API's fields, identifiers, history and access (§3).
- Then the Connector: per-User daily activity and credits, and Seats.
- A manual upload fallback for the admin console export, if the API falls short.
- Tool Categories in the catalogue.
- Duplicate Tools detection: Seats in two or more AI Tools of one Tool Category for one User, with the estimated saving of the cheaper Seat.

**Risks**:
- The ChatGPT Enterprise analytics API went generally available only in September 2026, and we couldn't read its reference from here.
- OpenAI must enable it per workspace.
- Workspace owners must create the key.

**Done when**:
- The spike's findings are written into §3 of this plan.
- Duplicate Tools are demoed on the Demo Client and detected on a real Client.

### Slice 6: Azure OpenAI

**Goal**: Estimated Cost per Azure OpenAI deployment (Workload).

**Scope**:
- Subscription selection after consent, and a check that the RBAC roles are assigned.
- Token volumes per day, deployment and model from Azure Monitor metrics (`ProcessedPromptTokens` / `GeneratedTokens`, split by `ModelDeploymentName` / `ModelVersion`). Backfill is about 93 days, the platform metric retention.
- Daily cost per resource and meter from the Cost Management Query API, stored as `publisher_amount`. Backfill runs month by month as far as the billing account allows, typically about 13 months.
- Workloads = Azure OpenAI resource + deployment, with resource tags proposed as Cost Centre.

**Risks**:
- Clients who buy Azure through CSP may have no cost visibility at subscription scope.
- Azure meter prices (regional, provisioned throughput units, Global vs. Data Zone) need their own Rate Card. Provisioned throughput (PTU) deployments are capacity, not per-token, so they are flagged as "not priced by Rate Card" in v1.
- RBAC role assignment is a manual Client step.

**Done when**: Client zero's Estimated Cost per deployment matches Cost Management's actual cost within 5% for pay-as-you-go deployments, and PTU deployments are visibly excluded.

### Slice 7: Provider operations, uploads and the full landing page

**Goal**: the Provider can run the service for many Clients, and AI Tools without an API come in by upload.

**Scope**:
- The cross-client view: failed Connections, open Unmatched Accounts and Opportunities still Open.
- Manual upload (template, preview, validation, replace by window, audit).
- Weekly digests.
- Landing page completed and polished.
- `monthlyKpiValues` and `achievementDraft` shape functions, with tests against the Delivery Platform's shapes.

**Done when**: a Delivery Manager can answer "what needs my attention across all Clients" from one page, and an upload of a non-API AI Tool shows in the Client's Estimated Cost.

### Slice 8: End of service, retention and go-live

**Goal**: production-ready.

**Scope**:
- End of service: read-only for 90 days, immediate revocation and Key Vault purge, and the 7-year purge job (periods still to be confirmed against contracts).
- The 25-month daily → monthly rollup.
- A translation check (all strings in `messages/en.json`, as in the Delivery Platform).
- Security review and pen test.
- A DPIA and works-council information pack: what is stored per User and what is never shown, from ADR 0003.
- Production Bicep, backups, alerts and runbooks.

**Done when**:
- Production is live with the first real Client.
- A restore from backup has been rehearsed.
- The retention jobs have been proven on time-shifted test data.

## 3. Vendor API facts

Researched on 2026-10-26. Sources:
- GitHub's published OpenAPI description (`github/rest-api-description`).
- Anthropic's docs at platform.claude.com.
- Microsoft's Copilot API reference (`MicrosoftDocs/m365copilot-docs`).
- Web search for OpenAI and Azure.

Several vendor doc sites (learn.microsoft.com, help.openai.com, learn.chatgpt.com) couldn't be fetched directly from the planning environment. Facts that rest only on secondary sources are marked **unverified** and must be confirmed in the slice that builds the Connector.

| | Microsoft 365 Copilot | GitHub Copilot | ChatGPT Enterprise | Azure OpenAI | Anthropic API |
| --- | --- | --- | --- | --- | --- |
| **Paid as** | Seat | Seat + metered premium requests and AI credits | Seat + shared credit pool | Metered (pay-as-you-go) or capacity (PTU) | Metered |
| **API** | Graph `GET /v1.0/copilot/reports/getMicrosoft365CopilotUsageUserDetail(period, version)` (CSV on v1.0, JSON on beta). Seats from Graph users' `assignedLicenses` / `assignedPlans`. | `GET /orgs/{org}/copilot/metrics/reports/users-1-day?day=` (signed NDJSON download links), the `organization-1-day` and `users-28-day/latest` variants, also under `/enterprises/{enterprise}`. Seats: `GET /orgs/{org}/copilot/billing/seats`. Metered: `GET /organizations/{org}/settings/billing/premium_request/usage` and `/ai_credit/usage` (filter by year, month, day, user, model). | Analytics API "Daily Usage" (per user per UTC day, or per workspace). **Unverified**: path reported as `/v1/analytics/workspaces/{workspace_id}/usage`. Users/seats via SCIM or the Compliance API (unverified). | Cost Management Query API at subscription scope (daily cost by meter and resource). Azure Monitor metrics `ProcessedPromptTokens` and `GeneratedTokens` on `Microsoft.CognitiveServices/accounts`. | `GET /v1/organizations/usage_report/messages`, `GET /v1/organizations/cost_report`, `GET /v1/organizations/usage_report/claude_code`. Claude Enterprise: `/v1/organizations/analytics/*`. |
| **Granularity** | Per User, **not daily**: a snapshot over a period (D7/D30 or D28/D90/D180) with Last Activity Date overall and per app. v2 adds prompts submitted and active usage days. | Per User per day: features, IDE, model and engagement. Seats: `last_activity_at`, `last_activity_editor`, `last_authenticated_at`, `created_at`, `pending_cancellation_date`. Premium requests: by day, user and model, with price per unit. | Per User per day: chats, messages, credits, breakdowns by client and model (secondary sources). | Per day, per resource and meter, in billing currency. Metrics per deployment and model version at 1-minute resolution, aggregated to days. | Buckets of 1m/1h/1d. Group by `api_key_id`, `workspace_id`, `account_id`, `service_account_id`, `model`, `service_tier`, `context_window`, `inference_geo`, `speed`. Token types: uncached input, cache creation (5m/1h), cache read, output, plus server tools. Cost in USD cents, daily. Claude Code: per user per day, with estimated cost per model. |
| **History** | Last Activity Date up to 180 days back. No daily history: daily data starts at connection. | Usage reports from 2025-10-10, up to 1 year back. Billing usage up to 24 months. | Default window 30 days. Maximum not found (**unverified**). | Monitor metrics ~93 days. Cost data typically ~13 months (billing account dependent). | Not limited in the docs we read. Max 31 daily buckets per request, so Backfill pages. Claude Enterprise Analytics from 2026-01-01. |
| **Lag** | Typically 2–3 days. Use `reportRefreshDate`. | 1–2 days for daily reports. Seat activity up to 24 h. | Usually 6–12 h, at most 24 h (secondary sources). | Cost 8–24 h, revised to month close. Metrics minutes. | Usage and cost about 5 min. Claude Code about 1 h. Enterprise cost up to 24 h, revised until about 7 days after month end. |
| **Permissions** | Application `Reports.Read.All`, plus `ReportSettings.Read.All` (concealment check) and `User.Read.All` (licences, department). Concealed names must be off (ADR 0007). | GitHub App installation, read-only (ADR 0006). The endpoints accept installation tokens. Seats: owner-level read. | Workspace-scoped Admin key with scope `enterprise.analytics.usage.read`. OpenAI may need to enable it per workspace. | Entra consent plus RBAC *Cost Management Reader* and *Monitoring Reader* per subscription. | Console: Admin API key (`sk-ant-admin01-…`), **full admin, cannot be scoped** (ADR 0006). Claude Enterprise: Analytics key `read:analytics`. |
| **Supports Idle Seat?** | **Yes**, through Last Activity Date. Seat assignment date from `assignedPlans`. | **Yes**, through seats `last_activity_at` and the daily user reports. Users with IDE telemetry off can look idle. | **Probably**, if the per-user rows carry an email. Confirm in the Slice 5 spike, with manual upload as fallback. | n/a (Metered) | n/a (Metered) |
| **Workload / User attribution** | User (UPN) | Tool Account (GitHub login → User by email or SAML identity) | Tool Account (email, unverified) | Workload = resource + deployment. Calls are not attributable to Users. | Workload = API key or workspace. Claude Code and personal keys can be attributed to a Tool Account (`account_id` / email). |

### Findings that change or threaten the spec

1. **Microsoft 365 names are concealed by default.** Decided: the Client turns concealment off (ADR 0007).
2. **Microsoft 365 Copilot has no daily per-User history.** The spec's "Backfill as far as each API allows" yields only Last Activity Dates, up to 180 days. Daily `activity_days` and trends start on the day of connection. Idle Seat works from day one; trend charts don't. The spec now says so.
3. **"Read-only admin API keys" isn't possible for the Anthropic Console.** Decided: accepted with mitigations (ADR 0006).
4. **GitHub has a consent-like option.** Decided: GitHub App (ADR 0006). Expect many Unmatched Accounts for organizations without SAML or EMU, which is why Slice 4 is where the Unmatched Account UI lands.
5. **Hybrid pricing.** GitHub Copilot and ChatGPT Enterprise are Seat-licensed *and* Metered. Decided: an AI Tool can be both, and the Rate Card prices Metered Units (glossary, spec).
6. **ChatGPT Enterprise's API is new and partly unverified.** A two-day spike opens Slice 5, and manual upload is the fallback.
7. **Azure OpenAI needs more than Entra consent.** Reading cost and metrics needs Azure RBAC on each subscription, which the Client assigns. CSP-bought subscriptions may expose no cost. PTU capacity isn't priced per token. Token metrics only go back about 93 days.
8. **Unattributed usage.** Anthropic playground usage has no API key and default-workspace usage has no workspace. It goes to an "Unattributed" Workload rather than being dropped.
9. **Per-User data is richer than we show.** The GitHub user reports and Claude Code analytics contain per-User productivity-like numbers (lines of code, sessions). Per ADR 0003 we store only activity facts per Tool Account, plus metered quantities needed for Cost Centre totals. We drop the rest at normalisation, and a contract test checks this.

## 4. Testing and quality

### 4.1 Unit tests (Vitest, no database)
- `src/domain`:
  - Idle Seat rule: period, threshold, Seats assigned in the period, leavers.
  - Duplicate Tools rule.
  - Opportunity lifecycle and who may move it.
  - Dismissal fingerprints.
  - Realised Saving confirmation.
  - Rate Card matching (effective dating, `price_dimensions`, discounts, missing price).
  - FX conversion and rounding to `numeric(14,2)`.
  - Collection windows (lag, relook, Backfill chunking).
  - Permissions per role.
  - KPI and Achievement shape functions against the Delivery Platform's shapes.
- Connector `normalise.ts` functions are pure and tested on fixtures (§4.3).

### 4.2 Integration tests (Vitest + real Postgres, as in the Delivery Platform)
- Migrations apply from empty. The audit trail refuses updates and deletes. Composite FKs and triggers refuse cross-Client references.
- Ingest is idempotent: the same Collection Run twice gives identical rows, and a later run with corrected values replaces the window.
- pg-boss handlers run in-process against the test database: retry-then-fail sets `failing` and queues one notification, and `needs_action` errors don't retry.
- Cost recalculation after a Rate Card change only touches affected months.
- Retention: time-shifted data is rolled up at 25 months and purged at 7 years after end of service.

### 4.3 Connector contract tests on recorded fixtures
- `npm run fixtures:record -- <connector>` calls the real API with Client zero's credentials (or a sandbox). It saves raw responses (JSON, NDJSON, CSV) under `src/connectors/<code>/fixtures/` and runs a **scrubber** first. The scrubber replaces emails, names, logins, tenant and org ids with deterministic fakes and fails if anything that looks like an email or GUID from the source is left.
- The tests replay fixtures through a fake `VendorHttp` and check:
  - the zod schema parses;
  - the normalised output matches a snapshot;
  - pagination is followed to the end;
  - errors are mapped correctly: 401 → `credentials_expired`, 403 with concealed names → `names_concealed`, 429 with `Retry-After` → retry;
  - nothing beyond the allowed per-User facts survives normalisation (§3 finding 9).
- A **nightly live-contract workflow** (GitHub Actions, not blocking PRs) runs `check()` and one day of `collect()` against Client zero for every Connector, using Key Vault through OIDC. It opens an issue when a schema stops parsing. Vendor APIs change (GitHub's Copilot metrics API was replaced in 2025), so this catches drift before Clients do.

### 4.4 Client isolation and privacy tests
- `seedTwoClients()` creates two look-alike Clients, as in the Delivery Platform. Every query module has a test that a scope for Client A never returns Client B's rows, and the worker's `openSystemScope` cannot write another Client's rows.
- A **catalogue test** reads `pg_catalog` and fails if any table with `client_id` has a foreign key to another Client-owned table that isn't composite, or lacks a `(client_id, id)` unique key.
- A **type-level and runtime privacy test**:
  - per-User server functions may only return `SeatFacts`;
  - a test enumerates the exports of `src/server/**` and fails if any result type has token or cost fields next to a `userId`;
  - no column named like `prompt`, `content`, `message_text` or `response` may exist.
- The Demo Client cannot get a non-synthetic Connection, and real Clients cannot get a synthetic one (constraint test).

### 4.5 End-to-end
- Playwright smoke tests on the Demo Client: sign in with the dev switcher, landing page, Idle Seat lifecycle, Unmatched Account resolution. They include axe accessibility checks.
- One Playwright run against staging after each deploy, with a synthetic Member signing in through email OTP using a test mailbox.

### 4.6 CI/CD (GitHub Actions)
- `ci.yml` on every push and PR mirrors the Delivery Platform:
  - Postgres 17 service;
  - `npm run check` (lint, format, typecheck, tests);
  - migrations and seed;
  - `next build`;
  - plus a Docker image build, `az bicep build` with linting, `npm audit --audit-level=high`, and Playwright on the Demo Client.
- `deploy.yml`: merges to `main` deploy to staging (image to ACR, Bicep `what-if` then apply, migrations as a one-off Container Apps job before rolling the apps). A GitHub Release deploys the same image to prod behind a protected environment with required reviewers.
- `live-contracts.yml`: nightly, as in §4.3.
- Dependabot for npm and GitHub Actions. CodeQL for TypeScript.

## 5. Open items not decided here

- Retention periods (90 days read-only, 7 years) still need checking against Client contracts (spec).
- Whether Client Approvers may enter API keys, or only Consultants. The spec names neither. The plan allows both, and it is easy to change.
- The concrete Tool Category list and which AI Tools count as duplicates. This is Provider catalogue content, needed by Slice 5.
- The Delivery Platform should record ADR 0008 (External ID) and ADR 0009 (Container Apps + pg-boss) itself if it adopts them.
