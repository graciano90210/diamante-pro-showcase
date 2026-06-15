# System Architecture

## Overview

Diamante Pro follows a **multi-tenant SaaS architecture** where a single deployment serves multiple independent companies (tenants), each with complete data isolation enforced at the query layer.

```mermaid
graph TB
    subgraph Clients
        WEB[Web Browser<br/>Dashboard]
        MOB[Flutter App<br/>Android]
    end

    subgraph Backend["Backend — Heroku (Python 3.12 / Flask 3.0)"]
        AUTH[Auth Blueprint<br/>JWT Login/Logout]
        API1[API v1<br/>Mobile REST]
        API2[API v2<br/>OpenAPI 3.0]
        BPS[Business Blueprints<br/>clientes · prestamos · cobros<br/>rutas · reportes · finanzas]
        SVC[Services Layer<br/>PrestamoService · CreditIntelligence<br/>EmailService · BillingService]
        MW[Middleware<br/>Subscription Check<br/>Role Guard]
    end

    subgraph Data["Data Layer"]
        PG[(PostgreSQL 15<br/>Multi-Tenant)]
        CACHE[SQLite<br/>Mobile Offline Cache]
    end

    subgraph Integrations
        MP[Mercado Pago<br/>Billing & Webhooks]
        SG[SendGrid<br/>Transactional Email]
        GP[Google Play<br/>Mobile Distribution]
    end

    WEB -->|HTTPS| AUTH
    WEB -->|HTTPS| BPS
    MOB -->|REST/JWT| API1
    MOB -->|Offline| CACHE
    CACHE -->|Sync on reconnect| API1

    AUTH --> MW
    BPS --> MW
    MW --> SVC
    SVC --> PG

    SVC --> MP
    SVC --> SG
```

---

## Multi-Tenancy Design

Every database query is scoped by `empresa_id`, which is embedded directly in the JWT payload at login time. This means:

- No middleware step needed to "switch tenant context"
- A misconfigured query returns empty results, not another tenant's data
- SUPER_ADMIN role bypasses this filter for cross-tenant analytics

```
JWT Payload
└── empresa_id: 42          ← injected at login
    role: "ADMIN"
    sub: "user_id"

Every SQLAlchemy query:
    .where(Model.empresa_id == current_user.empresa_id)
```

Security audit **v3.02** verified no cross-tenant data exposure across all 47 endpoints.

---

## Blueprint Structure

The app uses Flask's **factory pattern** (`create_app()`). Each blueprint owns one business domain:

| Blueprint | URL Prefix | Responsibility |
|---|---|---|
| `auth.py` | `/` | Login, logout, session management |
| `clientes.py` | `/clientes` | Client CRUD, profile, credit history |
| `prestamos.py` | `/prestamos` | Loan origination, approval workflow |
| `cobros.py` | `/cobro` | Collection management, payment recording |
| `rutas.py` | `/rutas` | Route assignment, collector management |
| `finanzas.py` | `/finanzas` | Cash reconciliation, financial reports |
| `reportes.py` | `/` | PDF/Excel report generation |
| `api.py` | `/api` | AJAX helpers for dashboard |
| `api_mobile.py` | — | v1 Mobile REST (legacy, read-only) |
| `api_openapi.py` | `/api/v2/` | v2 OpenAPI 3.0 (all new endpoints) |

---

## Subscription State Machine

A `before_request` hook enforces access based on the company's `estado_cuenta`:

```mermaid
stateDiagram-v2
    [*] --> TRIAL: New company registered
    TRIAL --> ACTIVO: Payment approved
    TRIAL --> TRIAL_EXPIRED: 15 days elapsed
    TRIAL_EXPIRED --> ACTIVO: Payment approved
    ACTIVO --> MOROSO: Payment overdue
    MOROSO --> ACTIVO: Payment received
    MOROSO --> CANCELADO: Grace period expired
    CANCELADO --> [*]
```

---

## Inline Schema Migrations

Rather than Alembic, `create_app()` runs `ALTER TABLE … ADD COLUMN IF NOT EXISTS` at every startup. Each statement runs in its own savepoint so a single failure doesn't abort the rest. This enables zero-downtime deploys on Heroku without manual migration commands.

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `empresa_id` on every table | Simplest tenant isolation — no schema-per-tenant complexity |
| JWT over session cookies | Stateless auth works identically for web and mobile |
| Services layer separated from blueprints | Business logic is testable without HTTP context |
| Dual API (v1 + v2) | v1 frozen for mobile compatibility; v2 evolves independently |
| SQLAlchemy 2.0 style only | Explicit sessions, no legacy `.query.` interface |
