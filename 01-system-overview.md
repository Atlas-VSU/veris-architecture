# 01 — System Overview

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** 2026  
> **Related Docs:** [Architecture Principles](./02-architecture-principles.md) · [Database Schema](./05-database-schema.md)

---

## Welcome to the Atlas Dev Team

Before you touch any code, read this document. Then read all the others.

VERIS is not a simple CRUD app — it is a multi-tenant, role-gated, subscription-aware platform where **security, data integrity, and feature access are enforced at the database layer**, not the application layer. Understanding the system holistically before contributing is non-negotiable.

The architecture decisions in this repository are deliberate and load-bearing. If something feels over-engineered, it's probably protecting against a real failure mode we've already thought through. If you believe a decision is wrong, raise it formally — open a PR, cite your reasoning, and we'll discuss it. Don't silently deviate.

---

## 1. Product Context

**VERIS** *(Latin: truth, record)* is a SaaS platform that enables student organizations in Philippine universities to manage:

- **Attendance** — event creation, check-in logging, absence tracking
- **Financials** — membership fees, fine assignment, GCash payment verification, clearance management
- **Student Self-Service** — student portal for viewing balances, submitting payments, and downloading clearance certificates

**Primary users:**

| Role | Scope | Description |
|------|-------|-------------|
| **System Admin** | Platform | VERIS platform administrator. Manages organizations, generates onboarding invite URLs, oversees platform-wide data with strict data privacy controls. Operates outside the org tenant model. |
| **Admin** | Organization | Organization owner. Full system access including billing, officer management, and subscription. |
| **Manager** | Organization | Operational lead. Manages events, financials, and clearance. Cannot change roles or subscription. |
| **Staff** | Organization | Execution-level. Handles attendance check-in and member directory lookup only. |
| **Student** | Organization | End-user (Premium only). Registers via Google OAuth, self-submits personal data (for data privacy compliance), and is verified by the organization before gaining full portal access. |

> **Note:** The multi-role system (Admin / Manager / Staff) is a **Premium-exclusive feature**. Basic and Plus plans operate with a single officer account that has full access.

> **Note:** The **System Admin** role is platform-level — it is not part of any organization and is not tied to subscription tiers. System Admins are managed directly in the database (seeded, not self-registered). See [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) for the full auth flow.

---

## 2. Subscription Tiers

VERIS is sold on an **annual per-student pricing model** with tier-based feature gating. All access control is enforced server-side via Supabase RLS.

### Tier Comparison

| Feature Domain | Basic | Plus | Premium |
|----------------|:-----:|:----:|:-------:|
| Price (₱ / student / year) | ₱2 | ₱3 | ₱4 |
| Minimum students | 100 | 75 | 60 |
| Annual floor price | ₱200 | ₱225 | ₱240 |
| Attendance management | ✓ | ✓ | ✓ |
| Member directory & bulk import | ✓ | ✓ | ✓ |
| Unlimited events & members | ✓ | ✓ | ✓ |
| Basic dashboard & trends | ✓ | ✓ | ✓ |
| Membership fees & manual fines | — | ✓ | ✓ |
| GCash payment verification (officer-side) | — | ✓ | ✓ |
| Officer clearance management | — | ✓ | ✓ |
| Basic financial summaries | — | ✓ | ✓ |
| Automated fines from attendance | — | — | ✓ |
| Student portal (Google OAuth + self-registration) | — | — | ✓ |
| Student self-service payment upload | — | — | ✓ |
| Clearance self-check & certificate download | — | — | ✓ |
| Fine appeal workflow | — | — | ✓ |
| Advanced financial analytics | — | — | ✓ |
| Multi-role RBAC (Admin / Manager / Staff) | — | — | ✓ |
| Custom branding & themes | — | — | ✓ |
| Priority support | — | — | ✓ |

### Billing Rules

- Pricing is assessed at **signup and at each annual renewal**.
- If an organization's student count falls below the tier minimum, billing defaults to the **floor price** (minimum students × per-student rate).
- Mid-year drops below minimum are **billed at the floor with no refunds**.
- The **initial setup fee** (system configuration, data migration, onboarding, training) is a **one-time negotiated contract fee** separate from annual subscription pricing.

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Frontend Framework** | Next.js 16+ (App Router) | RSC-first data fetching, Server Actions for mutations, file-based routing |
| **Language** | TypeScript | Type safety across the full stack |
| **Styling** | Tailwind CSS | Utility-first, consistent with design system |
| **UI Component Library** | ShadCN UI | Unstyled, composable primitives in `src/components/ui/`; feature components compose these |
| **Database** | Supabase (PostgreSQL) | Managed Postgres with built-in Auth, RLS, and real-time |
| **Authentication** | Supabase Auth | System Admin auth (email/password, seeded); Officer auth (email/password, invited via System Admin); Student auth (Google OAuth with self-registration + org verification, Premium only) |
| **Client State / Fetching** | React Query (TanStack Query) | Live data, cache invalidation, optimistic updates |
| **File Storage** | Firebase Storage | **Exclusively** for GCash payment receipt images |
| **Hosting** | Vercel (inferred) | Optimal for Next.js App Router deployments |

### Why Firebase Storage for Receipts?

GCash payment receipts are **user-submitted images** that require:
- Secure, time-limited access via signed URLs
- Independent storage scaling from structured data
- Simple upload flows from the student portal

Firebase Storage handles this cleanly. All other file/data concerns live in Supabase. **Do not use Firebase for anything else.**

---

## 4. Student Data Model

Based on the organization's member import template, students carry the following canonical fields:

| Column | Type | Notes |
|--------|------|-------|
| `IDNumber` | `text` | Organization-scoped student ID. Primary lookup key for check-in. |
| `LastName` | `text` | |
| `FirstName` | `text` | |
| `MiddleName` | `text` | Nullable |
| `Gender` | `text` | |
| `OtherNationality` | `text` | Nullable |
| `Town` | `text` | |
| `Province` | `text` | |
| `Course` | `text` | Degree program (e.g., BS Computer Science) |
| `Major` | `text` | Nullable specialization within a course |
| `Level` | `text` | Year level (e.g., 1st Year, 2nd Year) |
| `Department` | `text` | Academic department |
| `College` | `text` | College/faculty within the university |

> Note: Other columns will be omitted as they are not needed for the actual features

> For full SQL table definitions, see [05-database-schema.md](./05-database-schema.md).

---

## 5. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT BROWSER                           │
│                                                                 │
│   React Server Components (initial render, SEO, data fetch)     │
│   React Client Components + React Query (live, interactive UI)  │
│   ShadCN UI primitives (src/components/ui/) composed by features│
└───────────────┬─────────────────────────────┬───────────────────┘
                │ RSC / Server Actions        │ React Query (client)
                ▼                             ▼
┌───────────────────────────┐   ┌─────────────────────────────────┐
│     Next.js App Server    │   │         Supabase Realtime        │
│                           │   │   (subscriptions, live updates)  │
│  • React Server Components│   └─────────────────────────────────┘
│  • Server Actions (writes)│
│  • No app/api/ routes*    │
└───────────────┬───────────┘
                │ Supabase JS Client (server-side)
                ▼
┌───────────────────────────────────────────────────────────────┐
│                        SUPABASE                               │
│                                                               │
│  PostgreSQL Database                                          │
│  ├── RLS Policies (RBAC enforcement — Admin/Manager/Staff)   │
│  ├── Postgres Triggers (auto-fine generation, audit logs)    │
│  └── Stored Functions (complex business logic)               │
│                                                               │
│  Supabase Auth                                                │
│  ├── System Admin Auth: email/password (seeded, no signup)   │
│  ├── Officer Auth: email/password (invited by System Admin)  │
│  └── Student Auth: Google OAuth + self-reg (Premium only)    │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                     FIREBASE STORAGE                          │
│  GCash payment receipt images only                            │
│  Access via signed URLs generated in Server Actions           │
└───────────────────────────────────────────────────────────────┘

* Exception: app/api/ routes are permitted ONLY for external webhooks
  (e.g., payment gateway callbacks). All internal mutations use Server Actions.
```

---

## 6. Multi-Tenancy Model

VERIS is a **multi-tenant SaaS** where each **organization** is an isolated tenant.

- Every database table that contains org-specific data has an `organization_id` foreign key column.
- Supabase RLS policies enforce `organization_id` scoping on **every query** — officers cannot read or write data outside their own organization.
- Tier feature access is gated via the `organizations.tier` column, checked in RLS policies and Server Actions.
- Students (Premium) are additionally scoped to their organization via their auth metadata.
- **System Admins** operate outside the tenant model. They have cross-org read access for platform management (organization listing, subscription oversight, aggregate metrics) but are **prohibited from accessing individual student PII** unless explicitly required for a support action — and all such access is audit-logged. See [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md#5-rbac-enforcement) for data privacy constraints.

> For the full RLS policy specifications, see [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md).

---

## 7. Feature Gating Strategy

Feature access is enforced at **two layers** — both must hold:

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| **Database** | RLS `USING` clauses check `organizations.tier` | Hard enforcement — cannot be bypassed by the app layer |
| **Application** | Server Actions validate tier before executing | Early exit with clear error before hitting the DB |

UI-level feature hiding (hiding buttons, navigation items) is a **UX convenience only** — it is not a security control.

---

## 8. Related Documents

| Document | When to Read |
|----------|-------------|
| [02-architecture-principles.md](./02-architecture-principles.md) | Before writing any code — folder structure, core patterns, and ShadCN component strategy |
| [03-nextjs-guidelines.md](./03-nextjs-guidelines.md) | Before building any Next.js component, action, or ShadCN composition |
| [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) | Before touching auth, RLS, or file uploads |
| [05-database-schema.md](./05-database-schema.md) | Before writing any SQL or Supabase query |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | For understanding build order and phase dependencies |
