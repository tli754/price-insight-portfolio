# Price Insight

[![Build](https://github.com/acme-pricewatch/price-insight/actions/workflows/build.yml/badge.svg)](https://github.com/acme-pricewatch/price-insight/actions/workflows/build.yml)
[![Deploy](https://github.com/acme-pricewatch/price-insight/actions/workflows/deploy.yml/badge.svg)](https://github.com/acme-pricewatch/price-insight/actions/workflows/deploy.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![pnpm](https://img.shields.io/badge/pnpm-10.28.2-orange)](https://pnpm.io)
[![Turborepo](https://img.shields.io/badge/Turborepo-monorepo-blueviolet)](https://turbo.build)
[![Cloud Run](https://img.shields.io/badge/Deploy-Cloud%20Run-4285F4?logo=google-cloud)](https://cloud.google.com/run)

> **AI-native eCommerce competitor price monitoring platform** — automated scraping, structured AI extraction, and pricing analysis for Shopify merchants, deployed on Cloud Run with Terraform-managed infrastructure.

> This is a sanitized portfolio copy of a real production project. All GCP project IDs, domains, and store names below are fictional placeholders.

---

## Overview

Price Insight continuously monitors competitor product listings, extracts structured pricing data using AI, and surfaces market-position and margin insights for Shopify merchants. It compares a merchant's catalogue against live competitor data and recommends pricing.

**Core workflow:**
1. Competitor candidates are discovered per product via DataForSEO's shopping/SERP API (async task + pingback webhook)
2. An AI report service builds a structured payload from the product, its confirmed competitors, and recent sales history, and asks OpenAI (structured, schema-validated output) for a pricing recommendation — caching by input hash so unchanged inputs don't trigger a re-run
3. Prices are compared using shared statistics logic in `packages/core/`
4. The merchant's own Shopify orders/products sync in near-real-time via webhooks, with a scheduled daily reconciliation as a backstop
5. Results surface in the Nuxt dashboard

---

## Quick Start

### Prerequisites

| Tool | Version |
|------|---------|
| Node.js | ≥ 22 |
| pnpm | 10.28.2 |
| MySQL | 8.x |
| Docker | (optional, for local containers) |

```bash
corepack enable && corepack prepare pnpm@10.28.2 --activate

git clone https://github.com/acme-pricewatch/price-insight.git
cd price-insight
pnpm install
```

### Environment Setup

```bash
cp apps/backend/.env.example apps/backend/.env
cp apps/frontend/.env.example apps/frontend/.env
# fill in MySQL, OpenAI, Shopify, DataForSEO credentials in apps/backend/.env
```

### Start Development

```bash
pnpm dev   # starts backend (Fastify, :4000) and frontend (Nuxt, :3000) via Turborepo
```

---

## Features

- **Competitor discovery** — DataForSEO shopping/SERP search, processed via an async task + pingback webhook
- **AI pricing reports** — structured, schema-validated OpenAI calls over product/competitor/sales-history data, with input-hash caching and pending/success/failed status tracking
- **Price comparison** — percentile/average/median market position and margin analysis in `packages/core/` (pure JS, also exposed as a standalone CLI/tool-call schema)
- **Shopify integration** — OAuth client-credentials product/order sync, plus real-time order sync via HMAC-verified Shopify webhooks
- **Async processing on Cloud Run** — order sync and scheduled discovery run through Cloud Tasks + Cloud Scheduler against a private, IAM-gated `order-worker` service (no in-process queue, no Redis — see [Migration History](#migration-history))
- **Simple, deliberate auth** — single shared password issuing a JWT in an httpOnly cookie (no OAuth) — documented as an intentional MVP choice, not an oversight
- **IaC + CI/CD** — Terraform-managed Cloud Run/Cloud SQL/Secret Manager/load balancer; GitHub Actions with Workload Identity Federation (no long-lived service-account keys)

---

## Architecture

### Frontend — `apps/frontend/`

| Layer | Technology |
|-------|-----------|
| Framework | [Nuxt 4](https://nuxt.com) + Vue 3 |
| UI | [@nuxt/ui](https://ui.nuxt.com) (Tailwind v4) |
| Auth | None of its own — proxies `/api/**` and `/auth/**` to the backend, which owns the session cookie |

### Backend — `apps/backend`

| Layer | Technology |
|-------|-----------|
| Framework | [Fastify 5](https://fastify.dev) |
| ORM | [Drizzle ORM](https://orm.drizzle.team) over MySQL 8 |
| Validation | [Zod](https://zod.dev) |
| AI | OpenAI Responses API |
| Queue | [@google-cloud/tasks](https://cloud.google.com/tasks) (Cloud Tasks client) |

One backend image, three Cloud Run services:
- **`backend`** — public API + auth + Shopify/DataForSEO/OpenAI integrations
- **`order-worker`** — same image, different entrypoint (`order-worker-server.ts`); only registers `/internal/*` routes; private, invoked only via OIDC by Cloud Tasks/Cloud Scheduler
- **`frontend`** — the Nuxt app

### Infrastructure (`infra/terraform/`)

| Component | Detail |
|-----------|--------|
| Compute | 3 Cloud Run v2 services (frontend, backend, order-worker), scale-to-zero |
| Database | Cloud SQL (MySQL), Unix-socket volume mount — no public IP |
| Async | Cloud Tasks queue (`order-sync`, 3 retries/exponential backoff) + Cloud Scheduler (daily discovery job) |
| Secrets | Google Secret Manager, least-privilege per-service grants (backend/frontend/order-worker each only see the secrets they need) |
| Networking | Single global HTTPS load balancer, path-based routing (`/api`, `/auth`, `/webhooks` → backend; everything else → frontend), managed SSL cert, apex→www redirect |
| CI auth | Workload Identity Federation — no JSON service-account keys stored in GitHub |

### Security design decisions worth calling out

- **Defense in depth on internal endpoints**: `order-worker`'s `/internal/*` routes are gated by Cloud Run IAM (`run.invoker` scoped to one dedicated service account) *and* independently verify the Google-signed OIDC token's audience + signing service account at the application layer (`lib/verify-oidc.ts`) — in case ingress/IAM is ever loosened.
- **Shopify webhook verification**: HMAC-SHA256 over the raw request body, compared with `crypto.timingSafeEqual` to avoid timing side-channels (`lib/shopify-hmac.ts`).
- **Per-service least privilege**: `order-worker` has its own runtime service account scoped only to Cloud SQL + the DB/Shopify secrets it needs — it never has access to OpenAI/DataForSEO/session secrets that `backend` holds.
- **Split deploy ownership**: Terraform owns infrastructure *shape* (services, IAM, secrets); a separate CI service account owns routine image rollouts (`roles/run.developer`, scoped per-service) — so a compromised CI credential can't rewrite IAM policy.

---

## External Services

| Service | Purpose |
|---------|---------|
| [DataForSEO](https://dataforseo.com/) | Shopping/SERP competitor discovery, async task + pingback webhook |
| [OpenAI](https://openai.com) | Structured pricing report generation |
| Shopify Admin API | Merchant product/order sync (REST + GraphQL) + real-time order webhooks |

`packages/core/` additionally ships a standalone `price-insight-extract` CLI that uses [Jina Reader](https://jina.ai/reader/) to pull a product page and extract structured pricing data — independent of the backend's live AI-report pipeline above.

---

## Repository Structure

```
price-insight/
├── apps/
│   ├── backend/    # Fastify 5 API — routes, services, Drizzle schema, order-worker entrypoint
│   └── frontend/   # Nuxt 4 dashboard, proxies API/auth to the backend
├── packages/
│   └── core/       # Pure-JS price analysis library (analyzePrice), CLI bins, tool_call.json
├── infra/terraform/  # Cloud Run, Cloud SQL, Cloud Tasks/Scheduler, Secret Manager, IAM, load balancer
├── k8s/            # Historical reference only — pre-Cloud-Run GKE manifests, see below
└── .github/workflows/  # build, deploy, and Terraform plan/apply CI
```

---

## Local Development

```bash
pnpm install        # install all workspace dependencies
pnpm dev            # start backend + frontend via Turborepo
pnpm build           # build all packages
pnpm test            # run all tests
pnpm lint            # lint all packages
```

Backend-specific (run from `apps/backend/`):

```bash
pnpm db:generate     # generate a Drizzle migration from schema changes
pnpm db:studio       # open Drizzle Studio
```

`db:push` exists for local-only schema sync — never run it against a shared database; it doesn't record history in the migrations table.

---

## Deployment

Deploys are manually triggered via GitHub Actions `workflow_dispatch` (`deploy.yml`), targeting `all`, `backend`, `frontend`, or `order-worker` independently:

1. Authenticate via Workload Identity Federation (no stored keys)
2. Resolve the commit SHA and verify the image digest
3. `gcloud run deploy` the target service(s) with that digest
4. Terraform (`infra-terraform.yml`) separately owns the underlying service/IAM/secret *shape* — it `ignore_changes`s the image/command/args/traffic fields that CI manages, so routine deploys and infra applies don't fight each other
5. PRs touching `infra/terraform/` get a Terraform plan preview via `infra-terraform-plan.yml` before merge

---

## Migration History

This project originally ran on GKE with an in-cluster Redis-backed BullMQ queue and node-cron scheduling (manifests preserved under `k8s/` for reference). It was migrated to Cloud Run to get scale-to-zero and remove the operational overhead of running a Kubernetes cluster for a low-traffic internal tool. The in-process queue was replaced with Cloud Tasks + Cloud Scheduler (mirroring the old retry/concurrency settings), and Redis was dropped entirely — Cloud Tasks' own retry/backoff made it unnecessary.

---

## License

MIT — see [LICENSE](LICENSE).
