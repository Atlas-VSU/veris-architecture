# 06 — Implementation Roadmap

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** March 1, 2026  
> **Related Docs:** [System Overview](./01-system-overview.md) · [Architecture Principles](./02-architecture-principles.md) · [Database Schema](./05-database-schema.md)

---

## Purpose

This document defines the **high-level phased rollout** for building VERIS. Each phase builds on the previous one and maps directly to the product's subscription tiers.

Phases are sequential — do not start a phase until the previous one is complete and tested. Each phase has a dedicated detail file in `phases/` for granular task breakdowns, assignments, and acceptance criteria.

---

## Phase Overview

| Phase | Name | Objective | Detail File |
|-------|------|-----------|-------------|
| **0** | Project Setup | Repository, tooling, infrastructure, and CI/CD | [`phases/phase-0-project-setup.md`](./phases/phase-0-project-setup.md) |
| **1** | System Admin | Platform-level admin dashboard for org onboarding and management | [`phases/phase-1-system-admin.md`](./phases/phase-1-system-admin.md) |
| **2** | Basic Tier | Core attendance management for organizations | [`phases/phase-2-basic-tier.md`](./phases/phase-2-basic-tier.md) |
| **3** | Plus Tier | Manual financial management (fees, fines, payments, clearance) | [`phases/phase-3-plus-tier.md`](./phases/phase-3-plus-tier.md) |
| **4** | Premium Tier | Automation, student portal, RBAC, branding, and analytics | [`phases/phase-4-premium-tier.md`](./phases/phase-4-premium-tier.md) |

---

## Phase 0: Project Setup

> **Goal:** Establish the development environment, repository structure, CI/CD pipeline, and shared infrastructure so that all subsequent phases start from a stable, consistent foundation.

No user-facing features are built in this phase. This is pure engineering scaffolding.

### Backend

- Provision Supabase project (database, auth, connection pooling).
- Configure Firebase project (Storage bucket for GCash receipts only).
- Set up environment variable management (`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, Firebase config).
- Create the base database migration structure (Supabase CLI migrations).
- Seed the `organizations` and core lookup tables with test data.
- Seed the initial System Admin account(s) via Supabase Auth Admin API.

### Frontend

- Initialize Next.js project with App Router, TypeScript, Tailwind CSS, and ShadCN UI.
- Scaffold the feature-driven folder structure per [02-architecture-principles.md](./02-architecture-principles.md).
- Configure ShadCN UI in `src/components/ui/` — install base primitives (Button, Input, Card, Dialog, Table, etc.).
- Set up React Query (TanStack Query) provider.
- Set up Supabase client utilities (server-side and client-side helpers).
- Configure ESLint, Prettier, and TypeScript strict mode.
- Build the shared layout shell (auth-gated layout groups for `(system-admin)`, `(dashboard)`, `(student-portal)`).

### QA

- Verify the local dev environment runs end-to-end (Next.js → Supabase local → Firebase emulator).
- Validate CI pipeline: lint, type-check, and build pass on clean commit.
- Confirm Supabase migrations apply cleanly from scratch.
- Document the local setup guide for onboarding new developers.

---

## Phase 1: System Admin

> **Goal:** Build the platform-level System Admin dashboard — the entry point for onboarding organizations onto VERIS. No org-facing features yet.

This phase is a prerequisite for all tier features because organizations cannot exist without a System Admin creating them.

### Backend

- Implement the `org_invites` table and invite token generation logic (signed, time-limited tokens).
- Build the organization onboarding Server Action: validate invite → create auth user → create `organizations` row → create `officers` row → set `app_metadata`.
- Implement RLS policies for System Admin scope: cross-org read access to `organizations`, no access to student PII.
- Create the audit log table and trigger for System Admin actions.
- Build RPC functions for platform-level aggregate metrics (org count, student counts — anonymized).

### Frontend

- Build the System Admin login page (`/login` with role detection → redirect to `/system-admin`).
- Build the System Admin dashboard layout within the `(system-admin)` route group.
- **Organization Management:** List all organizations, view org details (tier, status, officer count, student count).
- **Invite Management:** Generate invite URLs, view pending/consumed invites, revoke unused invites.
- **Org Configuration:** Tier changes, org suspension/reactivation.
- **Audit Log Viewer:** Filterable log of System Admin actions.

### QA

- Verify System Admin cannot self-register — only seeded accounts can log in.
- Verify invite URLs expire correctly and cannot be reused after consumption.
- Verify the full onboarding flow: invite generated → org contact signs up → org + admin officer created → admin lands on dashboard.
- Verify System Admin cannot access student PII tables.
- Verify audit logs capture all System Admin write actions.

---

## Phase 2: Basic Tier — Attendance Management

> **Goal:** Deliver the core product — organizations can manage events, track attendance, and maintain a member directory. This is the minimum shippable tier.

### Backend

- Implement core database tables: `students`, `events`, `attendance_records`.
- Implement RLS policies scoped by `organization_id` for all Basic-tier tables.
- Build Server Actions for event CRUD (create, update, archive, delete).
- Build Server Actions for attendance check-in/check-out (lookup by student ID or name).
- Build Server Actions for member management (add, edit, bulk import via CSV/spreadsheet).
- Implement tier-gating checks: ensure Basic-tier orgs cannot access Plus/Premium endpoints.

### Frontend

- Build the officer login page and auth flow (email/password via Supabase Auth).
- Build the `(dashboard)` layout with sidebar navigation (Dashboard, Events, Attendance, Members).
- **Dashboard:** Real-time attendance trends, recent members, quick-access to upcoming events.
- **Events Module:** Create/edit events with time-in/out configuration, event listing with status.
- **Attendance Module:** Real-time check-in interface (student ID or name search), attendance log per event.
- **Member Management:** Searchable directory, add/edit member forms, bulk import UI with validation feedback.
- Implement tier-aware UI gating: hide Plus/Premium navigation items for Basic-tier orgs.

### QA

- Verify org isolation: Officer A cannot see Org B's data.
- Verify attendance check-in works via both student ID and name search.
- Verify bulk import handles edge cases (duplicates, malformed data, missing fields).
- Verify event time-in/out configuration enforces correct check-in windows.
- Verify tier gating: Basic-tier orgs cannot navigate to or call financial endpoints.
- Load test the attendance check-in flow for realistic concurrency (e.g., 50+ students checking in simultaneously).

---

## Phase 3: Plus Tier — Financial Management

> **Goal:** Add manual financial tools on top of Basic — officers can define fees, assign fines, verify GCash payments, and manage clearance.

### Backend

- Implement financial tables: `fee_types`, `fines`, `payments`, `clearance_records`.
- Implement RLS policies for financial tables (organization-scoped, tier-gated to Plus+).
- Build Server Actions for fee configuration (create semester/event/custom fee types).
- Build Server Actions for fine assignment (manual, officer-initiated).
- Build the GCash payment verification workflow: Firebase Storage signed URL generation (upload via Server Action), payment proof linking to `payments` table, officer approval/rejection flow.
- Build Server Actions for clearance management (status computation, manual override).
- Implement basic financial summary queries (total collected, outstanding balances, clearance rates).

### Frontend

- Extend the `(dashboard)` sidebar with financial navigation items (Fees, Fines, Payments, Clearance, Reports).
- **Fee Configuration:** Define and manage fee types (semester fees, event fees, custom).
- **Fines Module:** Assign fines to individual students, view fine history per student.
- **Payments Module:** GCash proof upload (officer-side for Plus), verification queue with approve/reject actions, payment history.
- **Clearance Module:** Per-student clearance status view, manual override controls, certificate generation trigger.
- **Reports:** Basic financial summaries — total collected, outstanding, per-event breakdowns.

### QA

- Verify Plus-tier feature access: Plus orgs can access financials, Basic orgs cannot.
- Verify GCash upload flow end-to-end: image upload → Firebase Storage → signed URL stored in DB → officer reviews and approves/rejects.
- Verify clearance status computation: student is cleared only when all fines are paid and no outstanding fees.
- Verify payment proof cannot be accessed by officers in a different organization.
- Verify financial summaries match the underlying transaction data.

---

## Phase 4: Premium Tier — Full Suite

> **Goal:** Deliver the complete VERIS experience — automated fine generation, student self-service portal, granular RBAC, org branding, and advanced analytics.

This is the largest phase. It introduces a new user type (Student) with its own auth flow, a separate portal application surface, and automation rules that span the attendance and financial domains.

### Backend

- Implement automated fine generation: Postgres trigger that creates fine records from attendance absences based on configurable org rules.
- Implement RBAC enforcement: update all RLS policies to check `role` from `app_metadata` (Admin / Manager / Staff).
- Build Server Actions for officer invitation and role management (Admin invites officers, assigns roles).
- Implement Student auth flow: Google OAuth via Supabase Auth, self-registration with `app_metadata` scoping, org verification step.
- Build the student-facing data layer: balance view, payment history, fine history, clearance status, appeal submission.
- Build the fine appeal workflow: student submits appeal → officer reviews → approve/reject with reason.
- Implement clearance certificate generation (PDF or downloadable format).
- Implement org branding storage: custom colors, fonts, logo, light/dark mode preference.
- Build advanced analytics queries: financial trends, attendance patterns, per-event ROI, comparative periods.

### Frontend

- **RBAC UI Enforcement:** Conditionally render navigation items, action buttons, and page access based on officer role (Admin / Manager / Staff). Staff see attendance + directory only.
- **Officer Management:** Admin-only UI for inviting officers, assigning/changing roles, removing officers.
- **Automation Config:** Admin UI for configuring fine-from-absence rules (which events trigger fines, fine amounts, grace periods).
- **Student Portal:** Separate `(student-portal)` route group with its own layout.
  - Google Sign-In + self-registration flow with org verification.
  - Student dashboard: balance summary, payment history, fine history.
  - GCash proof upload (student-side).
  - Fine appeal submission form.
  - Clearance self-check and certificate download.
- **Branding & Customization:** Admin settings page for logo upload, color theme, font selection, light/dark mode toggle. Applied dynamically via CSS variables.
- **Advanced Analytics:** Dashboard widgets with financial trends, attendance heatmaps, comparative period charts.

### QA

- Verify RBAC enforcement end-to-end: Staff cannot access financial pages or Server Actions. Manager cannot manage roles or subscription. Admin has full access.
- Verify automated fine generation: create event → mark student absent → fine record auto-created with correct amount and association.
- Verify student self-registration flow: Google OAuth → org verification → portal access.
- Verify students can only see their own data — no cross-student or cross-org data leakage.
- Verify fine appeal lifecycle: submit → pending → approved/rejected → fine adjusted accordingly.
- Verify clearance certificate generation produces a valid, downloadable document.
- Verify branding customization applies correctly across all org-scoped pages.
- Regression test all Basic and Plus features to ensure Premium additions don't break existing functionality.

---

## Dependencies & Constraints

```
Phase 0 ──► Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4
(Setup)    (Platform)   (Basic)     (Plus)      (Premium)
```

- **Strict sequential order.** Each phase depends on the infrastructure and data models of the previous phase.
- **Phase 1 is non-negotiable before Phase 2.** Organizations don't exist without System Admin onboarding.
- **Phase 2 is the MVP.** If timelines compress, Basic attendance is the minimum shippable product.
- **Phase 3 and Phase 4 can be descoped** for an initial launch, but the database schema should accommodate them from Phase 0 (forward-compatible migrations).

---

## Detailed Phase Files

Each phase has a dedicated detail file in `phases/` containing:

- Granular task breakdowns with assignable work items
- Database migration checklists
- API / Server Action specifications
- UI wireframe references (if applicable)
- Acceptance criteria per task
- Estimated effort and priority

| Phase | Detail File |
|-------|-------------|
| Phase 0 | [`phases/phase-0-project-setup.md`](./phases/phase-0-project-setup.md) |
| Phase 1 | [`phases/phase-1-system-admin.md`](./phases/phase-1-system-admin.md) |
| Phase 2 | [`phases/phase-2-basic-tier.md`](./phases/phase-2-basic-tier.md) |
| Phase 3 | [`phases/phase-3-plus-tier.md`](./phases/phase-3-plus-tier.md) |
| Phase 4 | [`phases/phase-4-premium-tier.md`](./phases/phase-4-premium-tier.md) |

> These files will be created as each phase enters active planning. Do not create them prematurely.
