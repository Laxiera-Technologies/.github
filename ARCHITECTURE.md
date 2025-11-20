# Laxiera Core – MVP Architecture

_Last updated: 2025-11-20_

This document describes the **MVP cloud architecture** for Laxiera Technologies (“Laxi”) backend services.  
It is intentionally opinionated and should be kept in sync with reality as we evolve.

---

## 1. Scope

This architecture covers the **centralized backend** used by:

- Android POS (Kotlin)
- Android Standalone Terminal (Kotlin)
- Cyvian / Laxi Invoicing App (React Native + TypeScript)
- Cyvian / Laxi Dashboard (React Native + TypeScript)
- Any future internal tools that talk to the same API

**Out of scope** (for now):

- Marketing / landing pages (Next.js – can be hosted separately)
- Mobile app builds / deployments

---

## 2. Cloud provider & regions

**Cloud provider:** Google Cloud Platform (GCP)  
**Primary project:** `laxi-core-prod`  
**Primary region:** `us-east1` (low latency to East Coast + good general default)

We adopt a **multi-env strategy** with separate projects:

- `laxi-core-dev`
- `laxi-core-stage`
- `laxi-core-prod`

All infra definitions should be environment-agnostic where possible (only project ID & DB names differ).

---

## 3. High-level components

### 3.1. Backend API (Node.js on Cloud Run)

- Service name: `laxi-core-api`
- Runtime: Node.js (JavaScript/TypeScript)
- Containerized and deployed to **Cloud Run (fully managed)**.
- Docker images stored in **Artifact Registry**:
  - `us-east1-docker.pkg.dev/<PROJECT_ID>/laxi-backend/core-api:<TAG>`
- Stateless by design; all state is in Postgres / Redis / message queues.

Responsibilities:

- REST/JSON API for all apps
- Authentication & authorization using Firebase ID tokens
- Business logic for POS, invoicing, loyalty, etc.
- Integration with RabbitMQ for async workflows

---

### 3.2. Database (PostgreSQL on Cloud SQL)

- Cloud SQL instance: `laxi-core-sql`
- Engine: PostgreSQL 15
- Region: `us-east1`
- Network: private IP attached to `laxi-core-vpc`
- Primary database: `laxi_core`
- Application user: `laxi_app`

Guidelines:

- All schema changes go through migrations (e.g., Sequelize / Knex / Prisma).
- No direct writes outside the backend service.
- Sensitive credentials are **never** committed to git; use Secret Manager.

---

### 3.3. Cache (Redis)

MVP setup:

- Redis runs in Docker on the same VM as RabbitMQ (see below).
- Intended usage:
  - Session-like cache (if needed)
  - Hot data caching (e.g., menu data, config, feature flags)
  - Rate limiting / throttling keys

Future:

- When cache usage becomes critical, move to **Memorystore (Redis)**:
  - Instance: `laxi-core-redis`
  - Region: `us-east1`
  - Access-restricted to `laxi-core-vpc`.

---

### 3.4. Messaging (RabbitMQ on Compute Engine)

- VM name: `laxi-core-mq-01`
- Machine type: `e2-micro` or `e2-small` (MVP)
- Zone: `us-east1-b`
- Network: `laxi-core-vpc` (no public IP; access via IAP/SSH)
- Containers:
  - `rabbitmq:3-management`
  - `redis:7.2` (for MVP)

Queues are used for:

- Sending receipts / emails
- Invoice and payment processing workflows
- Non-critical background jobs (sync, analytics events, etc.)

---

## 4. Networking

- VPC: `laxi-core-vpc`
- Subnet: `laxi-core-subnet-main` (region `us-east1`)
- Cloud SQL and MQ VM use **private IP** only.
- Cloud Run reaches internal resources via **Serverless VPC Connector**:
  - `laxi-core-svpc-us-east1`
- External access to MQ / Redis is **not allowed**; access only from:
  - Cloud Run service account
  - Admin bastion / IAP

---

## 5. Authentication & Security

### 5.1. Firebase

- Auth provider: **Firebase Authentication**.
- Clients obtain Firebase ID tokens; backend verifies them via Firebase Admin SDK.
- Backend roles (owner, admin, manager, staff, etc.) are stored in Postgres and mapped from the Firebase user.

### 5.2. Secret management

- All sensitive values live in **Secret Manager**:
  - `DB_PASSWORD`
  - `RABBITMQ_PASSWORD`
  - `REDIS_PASSWORD` (if used)
  - Any third-party API keys (Moov, Adyen, etc.)
- Cloud Run service account: `laxi-core-api-sa@<PROJECT_ID>.iam.gserviceaccount.com` is granted:
  - `roles/secretmanager.secretAccessor`
  - `roles/cloudsql.client`

---

## 6. Environments & config

We use **12-factor style configuration**:

- Configurable via environment variables.
- Secrets injected via Secret Manager.
- Each environment has its own values:

Examples:

- `NODE_ENV=production`
- `PORT=8080`
- `DB_HOST=<cloudsql private IP>`
- `DB_NAME=laxi_core`
- `DB_USER=laxi_app`
- `DB_PASSWORD=<from Secret Manager>`
- `REDIS_HOST=<internal host>`
- `REDIS_PORT=6379`
- `RABBITMQ_HOST=<internal host>`
- `RABBITMQ_USER=<from Secret Manager>`
- `RABBITMQ_PASSWORD=<from Secret Manager>`
- `FIREBASE_PROJECT_ID=<firebase project id>`

---

## 7. Repository layout (backend)

Recommended layout:

```text
backend/
  src/
    api/
      routes/
      controllers/
      middlewares/
    core/
      config/
      logger/
      errors/
    domain/
      models/         # Sequelize/Prisma models
      services/       # business logic
      repositories/
    infra/
      db/
      mq/
      cache/
    index.ts          # app entry point
  test/
  Dockerfile
  package.json