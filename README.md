<div align="center">
  <h1>💎 Diamante Pro</h1>
  <p><strong>Financial Intelligence SaaS Platform for Microcredit Operators</strong></p>

  ![Python](https://img.shields.io/badge/Python-3.12-blue?logo=python)
  ![Flask](https://img.shields.io/badge/Flask-3.0-black?logo=flask)
  ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-blue?logo=postgresql)
  ![Flutter](https://img.shields.io/badge/Flutter-3.x-02569B?logo=flutter)
  ![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-black?logo=github-actions)
  ![Status](https://img.shields.io/badge/Status-Beta_Live-green)
</div>

---

## What is Diamante Pro?

Diamante Pro is a **production-ready, multi-tenant SaaS platform** that digitalizes and optimizes the complete business cycle of microcredit operators: credit origination, portfolio management, field collection, and financial analytics.

Built from scratch as a full-stack system — from database architecture to a Flutter mobile app — it solves real operational problems that Excel spreadsheets and paper notebooks cannot.

---

## Business Problems Solved

| Problem | Solution |
|---|---|
| Manual collections tracked in notebooks/Excel | Real-time financial dashboard with live KPIs |
| No visibility of field collector location | GPS tracking + geo-referenced visit verification |
| Manual end-of-day cash reports | Automatic report generation in PDF/Excel |
| One system per company, no scalability | Multi-tenant architecture with full data isolation |
| Credit decisions based on intuition | Data pipeline ready for ML-based Credit Scoring |

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│              DIAMANTE PRO                   │
├──────────────────┬──────────────────────────┤
│   Web Dashboard  │     Mobile App           │
│   (Flask + HTML) │     (Flutter 3.x)        │
│   Strategic Mgmt │     Offline-First        │
│   KPI Reports    │     GPS Tracking         │
├──────────────────┴──────────────────────────┤
│            REST API (Flask)                 │
│         Authentication · JWT                │
│         Multi-Tenant Logic                  │
├─────────────────────────────────────────────┤
│         PostgreSQL · Multi-Schema           │
│         Full tenant data isolation          │
├─────────────────────────────────────────────┤
│    CI/CD: GitHub Actions → Heroku           │
└─────────────────────────────────────────────┘
```

---

## Core Features

### Financial Management
- Real-time loan portfolio dashboard
- Automated daily cash reconciliation reports (PDF/Excel)
- Credit origination workflow with risk classification
- Overdue tracking and collection management

### Field Operations
- Offline-first mobile app (Flutter) for field collectors
- GPS tracking and geo-referenced visit logging
- Data sync when connectivity is restored
- Photo capture and visit confirmation

### Multi-Tenancy & Security
- Full schema-level tenant isolation in PostgreSQL
- JWT authentication with role-based access control
- Cross-tenant data exposure prevention (security audit completed)
- LGPD/data privacy compliance architecture

### Analytics & ML-Ready
- KPI dashboards: collection rate, default rate, daily cash flow
- Data pipeline structured for Credit Scoring model integration
- Export-ready datasets for ML experimentation

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12, Flask 3.0, REST API |
| Mobile | Flutter 3.x (Android, offline-first) |
| Database | PostgreSQL 15, multi-schema |
| Auth | JWT, role-based permissions |
| DevOps | GitHub Actions, CI/CD pipeline |
| Deploy | Heroku (Web) + Google Play (Mobile Beta) |
| Reports | PDF generation, Excel export |

---

## Project Stats

- **612+ commits** across active development
- **2 branches**: production + development
- **Beta live** on Google Play (active testers)
- **Security audit** completed (Cross-Tenant Data Exposure v3.02)

---

## Documentation

| Doc | Description |
|---|---|
| [Architecture](docs/architecture.md) | System design, multi-tenancy strategy, blueprint structure |
| [Data Model](docs/data-model.md) | Entity hierarchy, ERD, key design decisions |
| [API Reference](docs/api-reference.md) | REST endpoints, auth flow, request/response examples |
| [ML Pipeline](docs/ml-pipeline.md) | Credit scoring architecture, XGBoost model, data pipeline |
| [Deployment](docs/deployment.md) | CI/CD pipeline, Heroku config, migration strategy |
| [Mobile App](mobile/README.md) | Flutter architecture, offline-first sync, GPS tracking |

---

## ML Integration Roadmap

Diamante Pro is architected to integrate with the companion research project:

> **[Credit Risk Modeling via Deep Survival Analysis](https://github.com/graciano90210/tcc-mba-survival-analysis)**
> Dynamic credit scoring engine using Cox Proportional Hazards + DeepSurv,
> trained on 28,386 real microcredit records. TCC — MBA Data Science @ USP/ICMC.

The platform's data pipeline is pre-structured to feed this model in production.

---

## Author

**Juan Fernando Graciano Pinzon**
Data Scientist · MLOps Engineer · MBA Candidate @ USP

[LinkedIn](https://linkedin.com/in/juanfernandograciano) · [GitHub](https://github.com/graciano90210)

> *This repository contains the public showcase. Source code is proprietary.*
