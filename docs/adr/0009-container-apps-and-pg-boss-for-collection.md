---
status: accepted
---

# Azure Container Apps for web and worker; pg-boss queues in Postgres for scheduling and retries

Token Management runs as one container image deployed as two Azure Container Apps in West Europe: `web`, the Next.js app, and `worker`, which runs Collection Runs, cost recalculation, Optimization Opportunity detection, notifications and retention. Scheduling, retries with backoff, per-Publisher concurrency limits and Backfill chunking use pg-boss job queues in the application's own PostgreSQL database. There is no separate message broker or scheduler service.

Collection is a small number of slow, rate-limited jobs per Client per day, so a queue in the database we already run is enough. It is transactional with the data it writes, and the whole system, worker included, runs locally with Docker Compose. Container Apps runs one image as both a web app and a background worker, scales to zero where wanted, and has no execution time limit, which long Backfills need.

## Considered options

- App Service for the web app with timer-triggered Azure Functions for Connectors: rejected because it means two runtimes and deployment models, and Function time limits complicate long Backfills.
- Azure Service Bus with Container Apps Jobs on a cron schedule: more Azure-native, but more resources to run and pay for, and harder to reproduce locally, with no need for it at our volume.

## Consequences

- The Delivery Platform will probably adopt the same runtime. Its own ADR should say so.
- If job volume ever outgrows Postgres, pg-boss can be swapped for Service Bus behind the job interface in `src/jobs`.
