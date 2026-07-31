# System-Design Case Study — A Production AI Service

> A walkthrough of how I architected, deployed, and operate **ATSAlign**
> ([atsalign.com](https://www.atsalign.com)) — an AI-powered ATS resume checker
> and optimizer that tailors resumes to any job description — running as a
> production AI service handling real traffic on Google Cloud. This repository
> documents **system design and
> engineering decisions**. Application source, AI prompts, and ranking logic are
> proprietary and intentionally excluded.

![status](https://img.shields.io/badge/status-in%20production-brightgreen)
![backend](https://img.shields.io/badge/backend-FastAPI-009688)
![frontend](https://img.shields.io/badge/frontend-Next.js-black)
![cloud](https://img.shields.io/badge/cloud-Google%20Cloud%20Run-4285F4)
![license](https://img.shields.io/badge/license-MIT-blue)

---

## What this is (and isn't)

This is a **design case study**, written the way I'd whiteboard the system in an
interview. It covers the architecture, the trade-offs behind each choice, the
data model, how it's deployed, and how I keep it secure and reliable.

It deliberately **does not** include the product's core IP — the AI prompts, the
scoring/ranking logic, or the optimization pipeline. Where those touch the
design, I describe the *engineering pattern* (e.g. "an async job with reserve →
process → settle semantics") rather than the domain logic inside it.

---

## Table of contents

- [System at a glance](#system-at-a-glance)
- [Architecture](#architecture)
- [Request lifecycle](#request-lifecycle)
- [Tech stack & why](#tech-stack--why)
- [Data model](#data-model-sanitized)
- [Security](#security)
- [Scalability & reliability](#scalability--reliability)
- [Deployment & CI/CD](#deployment--cicd)
- [Engineering challenges & lessons](#engineering-challenges--lessons)
- [Roadmap](#roadmap)

---

## System at a glance

A user uploads a document and a target description; the service runs an
LLM-backed analysis and returns structured results. Paid actions draw down a
metered balance. The system is a **stateless FastAPI backend** on Cloud Run, a
**Next.js frontend** on Vercel, and **PostgreSQL** as the system of record, with
object storage for files and a managed LLM provider for inference.

**Design goals that drove the architecture**

- **Stateless services** so the compute tier scales to zero and back under spiky traffic.
- **Correct billing under failure** — a user is never charged for an inference that didn't complete.
- **Fast, cheap reads** — expensive LLM work is cached per owner so repeat views don't re-pay for inference.
- **Security by default** — row-level security on the database, secrets out of the codebase, PII minimized.

---

## Architecture

```mermaid
flowchart TB
    subgraph Client
        U[Browser / Next.js app]
    end

    subgraph Edge["Vercel (Edge/CDN)"]
        FE[Next.js frontend]
    end

    subgraph GCP["Google Cloud"]
        API[FastAPI service on Cloud Run<br/>stateless, autoscaled]
        subgraph Async[Background work]
            W[Async task workers]
        end
    end

    subgraph Data["State & external services"]
        PG[(PostgreSQL<br/>row-level security)]
        OBJ[(Object storage<br/>uploaded files)]
        LLM[[Managed LLM provider]]
        PAY[[Payments provider]]
        OBS[[Sentry — errors & tracing]]
    end

    U --> FE
    FE -->|HTTPS / JSON| API
    API --> PG
    API --> OBJ
    API -->|inference| LLM
    API -->|webhooks| PAY
    API --> W
    W --> LLM
    W --> PG
    API -.-> OBS
    W  -.-> OBS
```

<!-- fallback image; GitHub renders the Mermaid above natively -->
<details><summary>Architecture diagram (PNG)</summary>

![Architecture diagram](docs/architecture.png)
</details>

The backend is the only writer to the database. The frontend never talks to the
database, the LLM provider, or object storage directly — every privileged action
goes through the API, which owns authentication, authorization, validation, and
billing.

---

## Request lifecycle

A representative **metered AI action**, shown as the generic engineering
pattern — *reserve → process → settle* — without the domain logic inside the
"process" step.

```mermaid
sequenceDiagram
    autonumber
    participant FE as Frontend
    participant API as FastAPI
    participant DB as PostgreSQL
    participant LLM as LLM provider

    FE->>API: POST /action (auth token, input ref)
    API->>API: Authenticate + validate input
    API->>DB: Atomically reserve balance (row lock)
    alt insufficient balance
        API-->>FE: 402 Payment Required
    else reserved
        API->>LLM: Run inference
        alt inference succeeds
            LLM-->>API: Result
            API->>DB: Persist result + settle reservation (commit)
            API-->>FE: 200 result
        else inference fails / times out
            API->>DB: Release reservation (refund)
            API-->>FE: 5xx, no charge
        end
    end
```

<!-- fallback image; GitHub renders the Mermaid above natively -->
<details><summary>Request lifecycle diagram (PNG)</summary>

![Request lifecycle](docs/sequence.png)
</details>

The key property: **the balance is reserved before the expensive call and
settled or refunded atomically after it.** A crash mid-inference releases the
reservation — the user is never charged for work they didn't get. Repeat reads
of an already-computed result are served from a per-owner cache and cost nothing.

---

## Tech stack & why

The *why* matters more than the *what* — here's the reasoning behind each choice.

| Layer | Choice | Why this, over the alternatives |
|---|---|---|
| Backend | **FastAPI (Python)** | First-class async for I/O-bound LLM calls; Pydantic gives validation + typed contracts + OpenAPI for free. |
| Compute | **Google Cloud Run** | Stateless containers that scale to zero — I pay for requests, not idle VMs. No cluster to babysit vs. self-managed Kubernetes. |
| Frontend | **Next.js on Vercel** | SSR/SSG for SEO-critical pages; zero-config previews and CDN; clean split from the API. |
| Database | **PostgreSQL** | Relational integrity for billing/accounts; row-level security enforces tenant isolation at the engine, not just in app code. |
| File storage | **Object storage (S3-compatible)** | Keeps large binaries out of the DB and off the app tier; signed, time-limited access. |
| Inference | **Managed LLM provider** | Buy capability, not GPUs. Provider abstracted behind an internal interface so the model is swappable. |
| Errors/tracing | **Sentry** | Exceptions + performance traces across API and workers, tied to releases. |
| CI/CD | **Cloud Build** | Push to `main` → build → deploy, no manual steps. |

---

## Data model (sanitized)

A high-level view of the **plumbing** entities. Tables and columns that encode
proprietary logic (scoring, weighting, ranking) are intentionally omitted.

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ ACTION_RUN : initiates
    USER ||--|| WALLET : owns
    USER ||--o{ PURCHASE : makes
    ACTION_RUN ||--o| RESULT_ARTIFACT : produces
    PURCHASE ||--|{ LEDGER_ENTRY : records

    USER {
        uuid id PK
        string email
        bool is_verified
        timestamptz created_at
    }
    SESSION {
        uuid id PK
        uuid user_id FK
        timestamptz expires_at
    }
    ACTION_RUN {
        uuid id PK
        uuid user_id FK
        string status
        timestamptz created_at
    }
    WALLET {
        uuid user_id FK
        int balance
    }
    PURCHASE {
        uuid id PK
        uuid user_id FK
        string sku
        string provider_ref
    }
    LEDGER_ENTRY {
        uuid id PK
        uuid wallet_ref FK
        int delta
        string reason
    }
```

<!-- fallback image; GitHub renders the Mermaid above natively -->
<details><summary>ER diagram (PNG)</summary>

![ER diagram](docs/er-diagram.png)
</details>

**Design notes.** Billing is modeled as an append-only **ledger** rather than a
mutable counter, so every balance change is auditable and reservations/refunds
are just ledger entries. Isolation between users is enforced by **row-level
security** in PostgreSQL — the backend connects as an owner role and every policy
is scoped by user.

---

## Security

- **Row-level security** on user-facing tables — tenant isolation at the database engine, defense in depth beyond app-layer checks.
- **Secrets never in the repo** — all credentials injected as runtime environment on Cloud Run / Vercel.
- **Verified-email gating** on privileged actions, with third-party-login accounts treated correctly (a subtle bug here once silently blocked verified users — see lessons).
- **Rate limiting** on public and metered endpoints to blunt abuse and cost-amplification.
- **Input validation everywhere** via Pydantic schemas at the API boundary; file uploads validated by type and size before storage.
- **Least-privilege access** to object storage via signed, expiring URLs.
- **PII minimization** — store what's needed, documented in a data-processing inventory.

---

## Scalability & reliability

- **Stateless compute** → Cloud Run scales horizontally on concurrency and to zero when idle; no sticky state to coordinate.
- **Caching of expensive work** → completed LLM results are cached per owner, so repeat reads never re-run inference (latency and cost both drop). *(Strategy only — cache keys and policies are internal.)*
- **Atomic billing under failure** → reserve-before-work + settle-or-refund guarantees no charge without delivery.
- **Async background work** for long-running tasks so request handlers stay responsive.
- **Observability** → Sentry captures exceptions and performance traces across both the API and workers, correlated to releases so a regression points at a deploy.
- **Cost-aware engineering** → per-call inference cost is instrumented so provider or model changes are caught quickly. *(I track this via telemetry; figures are internal.)*

---

## Deployment & CI/CD

```mermaid
flowchart LR
    Dev[git push to main] --> CB[Cloud Build: build container]
    CB --> Test[Run checks]
    Test --> Reg[Push image to registry]
    Reg --> Run[Deploy revision to Cloud Run]
    Run --> Live[(Live traffic, gradual rollout)]
    Dev -.-> V[Vercel auto-build] -.-> FELive[(Frontend live)]
```

<!-- fallback image; GitHub renders the Mermaid above natively -->
<details><summary>Deployment pipeline diagram (PNG)</summary>

![Deployment pipeline](docs/deployment.png)
</details>

- **Backend:** push to `main` triggers Cloud Build, which builds the container and deploys a new Cloud Run revision. Rollbacks are one click to a prior revision.
- **Frontend:** Vercel builds and deploys on push, with unique preview URLs per branch.
- **Config & secrets:** environment-specific, stored in the platform, never committed.

---

## Engineering challenges & lessons

Short, honest write-ups of things that broke and what I learned — the *lesson and
method*, not the proprietary internals.

- **A verification bug that error tracking couldn't see.** Third-party-login users on a legacy path weren't flagged as verified, so a `require_verified` gate silently rejected them — with no exception, so nothing surfaced in Sentry. *Lesson:* silent business-logic failures need their own signals; I added instrumentation on the gate, hardened the check to treat third-party identities as verified, and back-filled affected accounts. Errors aren't the only thing worth alerting on.

- **Provider cost swings with no code change.** Inference cost moved several-fold over weeks because the provider repointed a model alias underneath me. *Lesson:* treat third-party cost as a monitored metric — per-call cost telemetry turned an invisible drift into a dashboard line, and pinned vs. rolling model aliases became a deliberate choice.

- **Billing correctness under partial failure.** Early on, a charge could land for an inference that then failed. *Lesson:* model money as a ledger and make the expensive path *reserve → settle/refund* atomically, so failure can only ever cost the user nothing.

---

## Roadmap

- [ ] Extract the LLM-provider interface into a documented internal SDK boundary
- [ ] Formalize output-quality evaluation (golden sets + structured-output checks)
- [ ] Add read-replica routing for heavy read paths
- [ ] Expand load/latency benchmarking in CI

---

## License

Documentation and diagrams in this repository are released under the
[MIT License](./LICENSE). The ATSAlign product, its source code, AI prompts, and
ranking logic are proprietary and are **not** covered by this license.
