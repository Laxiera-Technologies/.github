# CONTRIBUTIONS.md — Laxiera Technologies LLC

**Last updated:** 2025‑11‑07  
**Scope:** Laxiera monorepo (Backend, Android POS, Dashboard Web, Standalone Terminal, Customer App, Infra)  
**Contact:** `@platform-architecture` • `@android` • `@web` • `@backend` • `@security`

This document is the **canonical contributor guide** for all Laxiera codebases. It explains how to set up your environment, coding standards, branching, testing, CI/CD, security, documentation, and the pull request process. It is **beginner‑friendly** and includes exact commands and file paths where possible.

---

## 0) Quick Start (TL;DR)

1) **Clone & tools**
```bash
git clone git@github.com:laxiera/laxiera.git
cd laxiera
# Recommended: Dev Containers (VS Code) or JetBrains with Docker
```

2) **Start core services (Postgres, Redis, RabbitMQ)**
```bash
docker compose -f infra/compose.dev.yml up -d
# postgres: localhost:5432 | redis: 6379 | rabbitmq: 5672 (mgmt UI: 15672)
# default creds live in infra/.env.example — copy to infra/.env and adjust
```

3) **Bootstrap Backend (Node.js/Express/Sequelize)**
```bash
cd apps/backend
cp .env.example .env     # fill Firebase Admin, DB URL, Stripe/Adyen/Moov test keys
pnpm install
pnpm db:migrate          # runs Sequelize migrations
pnpm db:seed             # demo data (safe to re-run in dev)
pnpm dev                 # http://localhost:8080
```

4) **Run Android POS (Kotlin/Compose)**
- Open **Android Studio** → *Open* → `apps/android-pos/`
- Create a `local.properties` if needed and set `sdk.dir`.
- In `apps/android-pos/app/src/main/assets/`, copy sample `config.dev.json` and update base URLs.
- Select **Pixel 7 Pro (API 34)** emulator → **Run** ▶️

5) **Run Dashboard (Next.js/React)**
```bash
cd apps/dashboard
cp .env.example .env.local
pnpm install
pnpm dev                 # http://localhost:3000
```

6) **Verify health**
- Backend: `GET http://localhost:8080/health`
- Dashboard: open browser to `http://localhost:3000`
- Android POS: login with seeded demo account (see `/apps/backend/seed/USERS.md`).

---

## 1) Monorepo Layout

```
laxiera/
├─ apps/
│  ├─ backend/               # Node.js, Express, Sequelize, OpenAPI, webhooks
│  ├─ dashboard/             # Next.js, React, shadcn/ui, Tailwind
│  ├─ android-pos/           # Kotlin, Jetpack Compose, Room, Retrofit
│  ├─ android-terminal/      # Kotlin terminal (payments-only)
│  └─ customer-app/          # React Native (Expo) or Next.js (channel)
├─ packages/
│  ├─ contracts/             # OpenAPI/JSON Schemas/Protobufs (single source)
│  ├─ ui/                    # Shared tokens, icons, CSS, theming
│  ├─ testing/               # Test helpers, fixtures, mock servers
│  └─ toolkits/              # SDKs/clients (TS/Kotlin)
├─ infra/
│  ├─ compose.dev.yml        # Postgres, Redis, RabbitMQ, Mailhog, MinIO (optional)
│  ├─ terraform/             # IaC for cloud
│  └─ k8s/                   # manifests/helm
├─ docs/
│  ├─ ADR/                   # Architecture Decision Records
│  └─ playbooks/             # On-call, incidents, DR
├─ .github/
│  ├─ workflows/             # CI/CD pipelines
│  ├─ ISSUE_TEMPLATE/        # Bug, Feature, RFC templates
│  └─ PULL_REQUEST_TEMPLATE.md
└─ CONTRIBUTIONS.md          # (this file)
```

> **Source of truth:** API models & events live in `packages/contracts`. **All surfaces** consume generated types from here.

---

## 2) Branching, Commits, Reviews

### Branching
- **Trunk‑based** on `main`, short‑lived branches.  
- Naming: `feature/<slug>`, `fix/<slug>`, `chore/<slug>`, `docs/<slug>`, `perf/<slug>`, `infra/<slug>`

### Conventional Commits
Use **Conventional Commits** to drive changelogs and semver:
```
feat(pos): seat assignment per table with stopwatch
fix(backend): handle null variant in item router
perf(android): reduce recompositions on TableNode by 40%
docs: add ADR for pricing rules engine
test(dashboard): add e2e for checkout flow
```
- Include `BREAKING CHANGE:` footer if applicable.

### Pull Requests
- Keep PRs **≤ 400 lines** net diff; split otherwise.  
- Link **issue** or **ADR**.  
- Include **Before/After** screenshots or short Loom for UI/UX.  
- Add **bench numbers** for perf changes (method, device, sample size).  
- **Checklist** (auto in PR template):
  - [ ] Code builds locally
  - [ ] Unit tests added/updated
  - [ ] E2E/regression updated (if affected)
  - [ ] Migrations safe + reversible
  - [ ] Contracts updated + `changeset` added
  - [ ] Feature flagged or backward compatible
  - [ ] Security/PII review (if applicable)
  - [ ] Docs/Help Center updated

### Code Review
- Two approvals required for security‑sensitive paths (& payments).  
- Review focuses: **correctness**, **readability**, **fail‑safety**, **observability**, **performance**, **accessibility**.

---

## 3) Coding Standards (Per Surface)

### Backend (Node.js/Express/Sequelize)
- **Runtime:** Node 20 LTS, PNPM.  
- **Style:** ESLint + Prettier; `src/` with domain modules; `routes/`, `controllers/`, `services/`, `models/`, `lib/`, `middleware/`, `webhooks/`.  
- **DB:** Postgres; migrations in `db/migrations`, seeds in `db/seeders`; use **transactional** migrations, **down** scripts required.  
- **Contracts:** Generate TS types & validators from `packages/contracts` on build.  
- **Security:** No secrets in code; use `envsafe`/Zod validation; parameterized SQL; rate limits; idempotency keys for payments.  
- **Logging:** pino (JSON), request IDs, redact PII.  
- **Testing:** Vitest/Jest; supertest for HTTP; 80%+ coverage target on core.  
- **Performance budget:** P95 API < 200ms on common endpoints (health, menu fetch, submit order).

### Android POS / Terminal (Kotlin, Compose)
- **Style:** Kotlin style, `ktlint` & `detekt` enforced.  
- **Architecture:** MVVM + Repository; **Room** for offline; `Retrofit` + `OkHttp` (retry, backoff).  
- **Compose:** Stability annotations; **remember** hoists; avoid allocations in draw scopes; use `Immutable` models.  
- **Performance:** Avoid `drawIntoCanvas` unless necessary; cache brushes/paths; list diffing with `Lazy*` keys; prefer `SnapshotStateList` for small reactive sets.  
- **Testing:** Robolectric unit, Instrumented UI tests for critical flows (login, open table, add item, pay).  
- **Access:** A11y labels, minimum 4.5:1 contrast for text on glass backgrounds.

### Dashboard (Next.js/React)
- **Style:** ESLint/Prettier, TypeScript strict.  
- **UI:** shadcn/ui, Tailwind tokens; responsive; semantic HTML; keyboard nav.  
- **Data:** React Query; optimistic updates behind feature flags; SSR only when needed.  
- **Testing:** Vitest + Playwright e2e; Lighthouse: Performance ≥ 90, A11y ≥ 95.

### Shared
- **Feature flags:** `packages/toolkits/flags` provider; never dead‑code delete without deprecation plan.  
- **I18n:** keys in `packages/ui/i18n`; no hard‑coded strings.  
- **Telemetry:** OpenTelemetry traces, metrics, logs; correlation IDs end‑to‑end.

---

## 4) Environments & Secrets

- **Local:** `.env` files (never commit); see `.env.example`.  
- **Dev/Stage/Prod:** OIDC/GitHub OIDC to cloud; secrets in **Vault/Secret Manager**; env injection via CI.  
- **Firebase Admin:** service account via workload identity if possible (no raw keys).  
- **PCI:** SAQ A – card data handled by processor SDKs/components only.  
- **PII:** encryption-at-rest; redaction in logs; data minimization.

---

## 5) Testing Strategy

### Pyramid
- **Unit** (fast, deterministic) → **Integration** (DB, queues) → **E2E** (Playwright, Android UI).  
- Contract tests ensure requests/responses match `packages/contracts`.  
- Payments: simulator flows for Stripe/Adyen/Moov; webhook replay tests.

### Commands
```bash
# backend
pnpm test
pnpm test:watch
pnpm coverage

# dashboard
pnpm test && pnpm e2e

# android (from Android Studio or CLI)
./gradlew testDebugUnitTest connectedDebugAndroidTest
```

### Quality Gates
- Lints pass; coverage >= thresholds; no high‑severity SAST/Dependency alerts.  
- Database migrations validated (apply & rollback) in CI ephemeral DB.

---

## 6) CI/CD

- **CI**: GitHub Actions in `.github/workflows/` – build, lint, test, type‑check, contracts validation.  
- **Artifacts**: Docker images pushed to registry on `main`; tags produce releases.  
- **CD**: Staging auto‑deploy on `main` green; Production requires manual approval + change ticket.  
- **Changelog**: auto‑generated from Conventional Commits; semantic‑release style.

---

## 7) Data Models & Contracts

- **Source of truth:** `packages/contracts` (OpenAPI/JSON Schema/Protobuf).  
- **Versioning:** additive changes preferred; breaking changes behind new version; deprecation period ≥ 90 days.  
- **IDs:** ULID/UUIDv7 with prefixes; time is RFC 3339 on wire; money is integer minor units.  
- **Events:** outbox pattern with reliable delivery; retried with backoff; DLQ.

---

## 8) Database & Migrations

- Create migration:
```bash
pnpm db:migrate:make add-table-discounts
```
- Write `up` + `down` with **safe DDL** (no locks on hot tables; use `CONCURRENTLY` where supported).  
- Include **seed fixture** updates if needed.  
- Provide **ERD diff** screenshot or text diagram in PR when schema changes.

---

## 9) Security & Privacy

- **Threat model** before introducing new data flows.  
- **Never** log secrets, tokens, or PAN/PII.  
- Use **idempotency keys** for POST/PUT payment endpoints.  
- **Permissions** checked server‑side and mirrored on client for UX.  
- **Dependency policy:** Renovate bot with weekly updates; pin ranges; no abandoned libs.  
- **Report vulnerabilities**: security@laxiera.com (see `SECURITY.md`).

---

## 10) Observability

- Pino/OTel structured logs with `traceId`, `spanId`.  
- Metrics: p95 latency, error rate, queue depth, DB slow queries.  
- Traces around **order open**, **submit**, **payment**.  
- Dashboards & alerts defined as code in `infra/monitoring/`.

---

## 11) Documentation & ADRs

- Every feature that alters architecture needs an **ADR** in `docs/ADR/` using `000X-title.md`.  
- Update **Help Center** (`/apps/dashboard/app/help/`) when user‑visible workflows change.  
- Code examples should be copy/paste‑able, with exact file paths.

---

## 12) Releases & Hotfixes

- **Release cadence:** weekly (Mon).  
- **Versioning:** semver at service boundaries.  
- **Hotfix:** branch from tag → PR to `main` + cherry‑pick to `release/*` if maintained.  
- Post‑release checklist: migrations applied, smoke tests, rollback plan validated.

---

## 13) Issue Triage & Labels

- Labels: `bug`, `feat`, `perf`, `security`, `payments`, `android`, `dashboard`, `backend`, `good first issue`, `help wanted`.  
- SLA (working days): `bug/critical` 1d, `payments` 1d, `security` immediate, `feat` scoped in sprint.  
- Use **RFC** issue template for large proposals.

---

## 14) Accessibility & UX

- A11y: keyboard navigable, landmarks, aria labels, focus trap in modals.  
- Contrast targets: text ≥ 4.5:1; large text ≥ 3:1.  
- Motion: reduce‑motion safe variants.

---

## 15) Performance Budgets (Examples)

- **Backend API**: p95 < 200ms for read, < 400ms for write.  
- **Dashboard**: First Interaction < 2.5s on mid‑tier laptop.  
- **Android POS**: Open Table → Ticket screen < 300ms; Compose recompositions minimized; no GC churn on draw.

---

## 16) Local Dev Recipes

- **Reset DB (dev only):**
```bash
docker compose -f infra/compose.dev.yml down -v
docker compose -f infra/compose.dev.yml up -d
pnpm -C apps/backend db:migrate && pnpm -C apps/backend db:seed
```

- **Payment Webhooks (Stripe example):**
```bash
stripe login
stripe listen --forward-to localhost:8080/webhooks/stripe
```

- **Generate contracts → clients:**
```bash
pnpm -C packages/contracts build
pnpm -C packages/toolkits generate
```

---

## 17) Contributor Covenant & DCO

- We adopt the **Contributor Covenant**. Be respectful, constructive, and inclusive.  
- **DCO** (Developer Certificate of Origin) is enforced; sign commits (`-s`) or configure globally:
```bash
git config --global user.signingkey <your-key>
git config --global commit.gpgsign true
```

---

## 18) Contacts & Escalation

- **Incidents / Payments:** `#oncall-payments` (PagerDuty)  
- **Security:** `#security` + security@laxiera.com  
- **Release management:** `#release-train`  
- **Architecture Qs:** `#platform-architecture`

---

### Appendix A — PR Template (Summary)
See `.github/PULL_REQUEST_TEMPLATE.md`. Ensure:
- Title uses Conventional Commit style (scope optional).
- Linked issues, screenshots, tests, rollout plan, and metrics are present.

### Appendix B — ADR Template
```
# ADR 00XX: <Title>
Date: YYYY-MM-DD
Status: Proposed | Accepted | Superseded by ADR-XXXX

## Context
(Background and forces)

## Decision
(What we decided and why)

## Consequences
(Positive, negative, tradeoffs)

## Alternatives
(Other options considered)

## References
(Links to PRs, issues, benchmarks)
```
