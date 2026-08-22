# Keystrada — AI-Powered Property & Tenant Management Platform

(https://keystrada.vercel.app)

An end-to-end SaaS platform that lets property management companies and landlords run tenant
tracking, lease contracts, rent collection, maintenance requests, and rent-receipt matching
**from a single panel**.

**Team:** A two-person team — **Furkan Turan** & **Yaşar Samed Erol**. We both worked across
the whole stack — frontend and backend alike; from design to the API, from the database schema
to the AI module, the entire system was built together.

[🇹🇷 Türkçe versiyon](README.md)

---

## ⚠️ About This Repo

This repository is **portfolio-only**. The project's source code is not shared, due to client
business data and records that fall under data protection regulations (Turkish KVKK). What
you'll find here is the system architecture, the technologies used, the problems it solves, and
screenshots of the interface.

---

## Problem & Solution

At real estate offices and among landlords, tenant records, lease contracts, payment tracking,
and maintenance requests are usually scattered across spreadsheets, WhatsApp, and paper. That
leads to lost information, delayed collections, and — in particular — time-consuming manual work
to figure out **which tenant or property a given bank rent receipt actually belongs to**.

Keystrada brings all of these processes into one system, and as the project's most distinctive
piece, it reads uploaded PDF bank receipts through an **OCR + NLP-based AI module** and
automatically matches them to the right tenant and property based on a confidence score —
removing most of the manual reconciliation work an accounting team would otherwise do by hand.

---

## Three Role-Based Experiences

| Role | Access |
|---|---|
| **Admin** | Full system access: staff, reports, all property and finance data |
| **Agent (Emlakçı)** | Portfolio management — properties, tenants, listings, leases, maintenance, appointments, collections (restricted staff/report access) |
| **Tenant (Kiracı)** | A separate tenant portal showing their own lease, payment history, and maintenance requests |

Role control isn't limited to hiding UI elements — it's enforced end-to-end through
middleware-level route protection, page-level guard components, and a dynamic (role-filtered)
sidebar and command palette.

---

## Modules

| Module | What it does |
|---|---|
| **Dashboard** | Role-based overview: active contracts, collection status, maintenance requests, revenue/expense trend |
| **Properties** | Apartment/flat/villa/land/commercial records, multi-unit (apartment/plaza) management, image gallery, notes |
| **Listings** | Create sale/rental listings from a property, JSON feature fields, publish status |
| **Tenants** | Tenant profiles, document archive (ID, residence, contract), tenant-specific notes |
| **Leases** | Contracts, deposits, payment-day tracking; automatic monthly rent debiting every night |
| **Finance** | Accounts (tenant, owner, bank, expense, income), transaction and payment records, automatic balance calculation |
| **Rent Receipts (AI)** | Upload a PDF receipt → OCR + NLP automatically matches it to a tenant/property, with a confidence-scored approval flow |
| **Maintenance** | Category/priority-based maintenance requests, technician assignment, cost pass-through to tenants |
| **Reservations** | Property viewing / appointment calendar |
| **Staff** | Security, cleaning, and technician staff; assignable across multiple properties |
| **Documents** | Property documents such as deeds, insurance, and DASK policies; polymorphic document system |
| **Notifications** | In-app, email, SMS, and push notification preferences |
| **Reports** | Property and staff performance reports, PDF/Excel export |
| **Users & Roles** | RBAC management, role-based user list and authorization |

---

## Screenshots

### Landing Page & Dashboard
The product's public landing page, with a dashboard preview from the panel below it.

![Keystrada Landing & Dashboard](foto1.png)

### Key Features
The landing page section describing the core problems the product solves.

![Key Features](foto2.png)

### Management Panel — Dashboard
Revenue summary, active contract count, pending payments, maintenance status, and the
appointment calendar on a single screen.

![Dashboard Detail](foto3.png)

---

## Tech Stack

**Frontend**
- **Next.js (App Router)** + **React 19** + **TypeScript**
- **Tailwind CSS v4** + **shadcn/ui (Radix primitives)** — design system and accessible component layer
- **Zod** — form and schema validation
- **TanStack Table** — paginated, filterable data tables
- **Recharts** — dashboard charts
- Route-group architecture: `(public)` / `(auth)` / `(app)`, with Next.js Route Handlers as a backend proxy layer

**Backend**
- **Python / Django REST Framework** — RESTful API
- **PostgreSQL 16** — relational data model
- **Celery + Redis** — background jobs (automatic monthly rent debiting, receipt-processing queue)
- **JWT (SimpleJWT)** — authentication with access/refresh rotation and a blacklist mechanism
- **drf-spectacular** — auto-generated OpenAPI/Swagger API docs
- Backend authorization via Django `Permission` combined with a custom `Role` model
- Polymorphic `Document` / `Transaction` models via `GenericForeignKey`

**AI / NLP Module**
- **BERT-based intent classifier** — classifies receipt text intent ("rent payment", "maintenance fee payment", etc.)
- **NER (Named Entity Recognition)** — extracts sender name, amount, date, and unit number
- **OCR** — extracts text from PDF receipts
- **Fuzzy matching (fuzzywuzzy)** — matches tenant name and property address
- Runs directly inside the Django process rather than as a separate microservice

**Infrastructure**
- **Docker + Docker Compose** — container isolation
- **Nginx** — reverse proxy
- **Gunicorn** — WSGI server (multiple workers/threads)
- **Celery Beat** — scheduled jobs (automatic rent debiting at 00:01 every night)

---

## Architecture

```
                         ┌───────────────────────────────┐
   Visitor      ────────▶│                               │
                         │             Nginx             │
   Admin/Agent  ────────▶│      reverse proxy · SSL      │
   Tenant       ────────▶│                               │
                         └───────────────┬───────────────┘
                                         │
                          ┌──────────────▼───────────────┐
                          │           Frontend           │
                          │      Next.js (App Router)    │
                          │  (public) · (auth) · (app)   │
                          │  JWT-based auth · RBAC guards│
                          │  route-group middleware      │
                          └──────────────┬───────────────┘
                                         │ REST API (JSON/JWT)
                          ┌──────────────▼───────────────┐
                          │      Backend (Django + DRF)  │
                          │   Role/Permission · JWT auth │
                          └───┬────────────────────────┬─┘
                              │                        │
                   ┌──────────▼─────────┐   ┌──────────▼──────────┐
                   │    PostgreSQL      │   │  Celery + Redis     │
                   │  property·tenant·  │   │  automatic rent     │
                   │  lease·finance     │   │  debiting           │
                   └────────────────────┘   │  receipt processing │
                                            │  queue              │
                                            └───────────┬─────────┘
                                                        │
                                             ┌──────────▼──────────┐
                                             │   NLP/OCR Module    │
                                             │  BERT · NER · Fuzzy │
                                             │  Matching · OCR     │
                                             └─────────────────────┘
```

**Data model summary:** Properties link to tenants through a `Lease`; each lease is automatically
debited by Celery Beat according to its monthly payment day. Financial movements are tracked
through the `Account` → `Transaction` → `Payment` chain, with the balance recalculated via
Django signals on every transaction change. Rent receipts go through a separate `RentReceipt`
pipeline where OCR/NLP processing attaches them to that chain automatically or via manual
approval, based on the confidence score.

---

## Notable Technical Points

- **AI-powered receipt matching:** Confidence score ≥ 80% auto-matches, 50–80% goes to manual
  review, below 50% is rejected — cutting down the manual reconciliation work for the accounting
  team.
- **Automatic monthly rent debiting:** Celery Beat scans active leases every night at 00:01 and
  creates a debit record for tenants whose payment day has arrived, with 3 automatic retries on
  failure.
- **Real-time balance calculation:** Account balances are automatically recalculated on every
  transaction change via Django `post_save`/`post_delete` signals.
- **End-to-end RBAC:** Route-based role checks in middleware, page-level guard components, and a
  dynamic sidebar and command palette (Cmd+K) — a role-filtered structure instead of a
  hard-coded menu.
- **Tenant self-service portal:** Users with the tenant role can track their own lease, payment
  history, and maintenance requests from a dedicated dashboard view.
- **A team that audits its own project:** Before commercialization, the entire codebase was
  reviewed and a comprehensive self-audit report was produced (security, RBAC, error handling,
  performance); those findings were then broken into sprints and implemented — security headers
  (CSP, X-Frame-Options), global and route-level error boundaries, tightened JWT handling, and
  the full role-based authorization system were all delivered through that process.
- **Multi-unit property support:** Properties with multiple independent units, such as apartment
  buildings or plazas, are managed under a single record, organized by unit number.

---

## Contact

Feel free to reach out for a similar project or collaboration.

**Furkan Turan** & **Yaşar Samed Erol** — *Full-stack Developers*
