# Phase 0 — Project Setup & Foundation

> **Phase Status:** Not Started  
> **Last Updated:** March 1, 2026  
> **Parent Doc:** [06-implementation-roadmap.md](../06-implementation-roadmap.md)  
> **Depends On:** None (first phase)

---

## Objective

Establish the development environment, repository structure, CI/CD pipeline, and shared infrastructure. No user-facing features are built in this phase — this is pure engineering scaffolding.

**Exit Criteria:** A developer can clone the repo, run one setup command, and have:
- Next.js running locally with all route groups stubbed
- Supabase local instance with the platform-core schema applied
- Firebase emulator configured for Storage
- CI pipeline passing lint, type-check, and build

---

## Epics

| # | Epic | Description | Sprint |
|---|------|-------------|--------|
| E0.1 | [Repository & Tooling](#e01-repository--tooling) | Next.js project init, linting, formatting, TypeScript config | Sprint 1 |
| E0.2 | [ShadCN & Frontend Foundation](#e02-shadcn--frontend-foundation) | ShadCN setup, shared layout shells, providers, Supabase client utils | Sprint 1 |
| E0.3 | [Supabase Provisioning](#e03-supabase-provisioning) | Supabase project, CLI setup, base migration structure | Sprint 1 |
| E0.4 | [Database Schema Migration](#e04-database-schema-migration) | Platform-core tables needed for Phase 0–1 (remaining tables migrate in their respective phases) | Sprint 1 |
| E0.5 | [Firebase Configuration](#e05-firebase-configuration) | Firebase project, Storage bucket, emulator setup | Sprint 2 |
| E0.6 | [CI/CD Pipeline](#e06-cicd-pipeline) | GitHub Actions for lint, type-check, build, migration validation | Sprint 2 |
| E0.7 | [Seed Data & Dev Environment](#e07-seed-data--dev-environment) | System Admin seeding, test org data, developer setup guide | Sprint 2 |

---

## Sprint Breakdown

| Sprint | Duration | Focus | Epics |
|--------|----------|-------|-------|
| **Sprint 1** | 2 weeks | Project init, frontend scaffold, Supabase setup, platform-core migrations | E0.1, E0.2, E0.3, E0.4 |
| **Sprint 2** | 2 weeks | Firebase, CI/CD, seed data, dev environment docs | E0.5, E0.6, E0.7 |

---

## E0.1: Repository & Tooling

> Initialize the Next.js project with the exact tech stack and enforce code quality from day one.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.1.1 | Initialize Next.js project with App Router | Backend / Frontend | `npx create-next-app@latest` with TypeScript, Tailwind CSS, App Router, `src/` directory enabled. Runs on `localhost:3000`. |
| T0.1.2 | Configure TypeScript strict mode | Frontend | `tsconfig.json` has `"strict": true`, `"noUncheckedIndexedAccess": true`. Zero type errors on clean project. |
| T0.1.3 | Configure ESLint with Next.js rules | Frontend | `.eslintrc.json` extends `next/core-web-vitals` and `next/typescript`. Lint passes on clean project. |
| T0.1.4 | Configure Prettier | Frontend | `.prettierrc` defined with project conventions. Format-on-save works. No conflicts with ESLint. |
| T0.1.5 | Set up path aliases | Frontend | `@/` alias resolves to `src/` in `tsconfig.json`. Imports like `@/components/ui/button` work. |
| T0.1.6 | Create `.env.local.example` | Backend / Frontend | Template with all required env vars: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, Firebase config keys. No real secrets committed. |
| T0.1.7 | Configure `.gitignore` | Backend / Frontend | Covers `node_modules/`, `.env.local`, `.next/`, Supabase local data, Firebase emulator data. |

---

## E0.2: ShadCN & Frontend Foundation

> Scaffold the feature-driven folder structure, install ShadCN primitives, and set up shared providers per [02-architecture-principles.md](../02-architecture-principles.md) and [03-nextjs-guidelines.md](../03-nextjs-guidelines.md).

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.2.1 | Scaffold feature-driven folder structure | Frontend | Create the full directory tree per [02-architecture-principles.md](../02-architecture-principles.md): `src/app/`, `src/components/{ui,layout,shared}/`, `src/features/{auth,attendance,members,financials,clearance,organizations,system-admin}/`, `src/lib/{supabase,firebase}/`, `src/types/`. Each feature folder has `components/`, `hooks/`, `actions/`, `types/`, `utils/` subfolders. |
| T0.2.2 | Initialize ShadCN UI | Frontend | Run `npx shadcn@latest init`. Configure `components.json` with `@/components/ui` path, `tailwind.config.ts` CSS variables enabled. |
| T0.2.3 | Install base ShadCN primitives | Frontend | Install: `Button`, `Input`, `Card`, `Dialog`, `Table`, `Select`, `Badge`, `Dropdown Menu`, `Tabs`, `Separator`, `Skeleton`, `Toast` (Sonner), `Form`. All importable from `@/components/ui/*`. |
| T0.2.4 | Set up CSS variables and theming | Frontend | `globals.css` configured with ShadCN CSS variable system (light/dark). Tailwind config references these variables for colors. |
| T0.2.5 | Create root layout | Frontend | `src/app/layout.tsx`: `<QueryProvider>` wrapper, `<html>` with `suppressHydrationWarning`. |
| T0.2.6 | Create React Query provider | Frontend | `src/components/providers/QueryProvider.tsx`: `"use client"` wrapper with `QueryClientProvider` and default client config (stale time, retry). |
| T0.2.7 | Stub route group layouts | Frontend | Create placeholder `layout.tsx` for each route group: `(auth)`, `(dashboard)`, `(portal)`, `(system-admin)`. Each exports a basic shell component. Route protection logic is **not** implemented yet (deferred to Phase 1 & 2). |
| T0.2.8 | Create Supabase client utilities | Frontend / Backend | `src/lib/supabase/server.ts`: `createServerClient()` using `@supabase/ssr` with cookie handling. `src/lib/supabase/client.ts`: `createBrowserClient()` for React Query usage. Both read from env vars. |
| T0.2.9 | Create Firebase client utility | Frontend / Backend | `src/lib/firebase/storage.ts`: Firebase app initialization and Storage instance export. Configured via env vars. |
| T0.2.10 | Create global shared types | Frontend / Backend | `src/types/database.types.ts`: placeholder for Supabase-generated types (generated in E0.4). `src/types/action-response.ts`: `ActionResponse<T>` discriminated union type per [02-architecture-principles.md](../02-architecture-principles.md). |

---

## E0.3: Supabase Provisioning

> Provision the Supabase project and configure the local development environment with the Supabase CLI.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.3.1 | Create Supabase cloud project | Backend | Project created on `supabase.com`. Project URL and API keys obtained. Region selected (closest to target users). |
| T0.3.2 | Install and configure Supabase CLI | Backend | `supabase init` in project root. `supabase/config.toml` configured with project ref. CLI version pinned in docs. |
| T0.3.3 | Start Supabase local development | Backend | `supabase start` runs successfully. Local Postgres, Auth, and Studio accessible. Connection string documented. |
| T0.3.4 | Link local project to cloud | Backend | `supabase link --project-ref <ref>` connects local CLI to cloud project. `supabase db pull` works. |
| T0.3.5 | Configure connection pooling | Backend | Verify Supabase connection pooler (Supavisor) is enabled. Document the pooled connection string for server-side usage. |

---

## E0.4: Database Schema Migration

> Migrate only the **platform-core tables** needed for Phase 0 (setup) and Phase 1 (System Admin). Remaining tables (attendance, financials, clearance, etc.) are added as migrations in their respective phases — this avoids premature schema commitments for features that may evolve during planning.

### Scope

| Table | Why It's Needed Now |
|-------|--------------------|
| `system_settings` | Platform identity. Seeded during setup. |
| `organizations` | Tenant root. System Admin creates orgs in Phase 1. |
| `organization_settings` | 1:1 with orgs. Auto-created on org onboarding. |
| `org_invites` | System Admin generates invite tokens in Phase 1. |
| `officers` | Officer accounts created during org onboarding in Phase 1. |
| `audit_log` | System Admin actions are audit-logged from Phase 1. |

### Deferred to Later Phases

| Phase | Tables |
|-------|--------|
| Phase 2 (Basic) | `students`, `student_org_memberships`, `bulk_import_requests`, `org_registration_links`, `academic_periods`, `event_types`, `events`, `attendance_records` |
| Phase 3 (Plus) | `fee_types`, `fee_assignments`, `fine_types`, `fines`, `payments`, `waivers`, `payment_allocations`, `clearances`, `subscription_payments`, `notification_queue` |
| Phase 4 (Premium) | `fine_appeals` |

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.4.1 | Migration: `system_settings` table | Backend | Creates `system_settings` with all columns per schema doc. Single-row constraint enforced. |
| T0.4.2 | Migration: `organizations` table | Backend | Creates `organizations` with all columns, `tier` and `status` check constraints, `slug` unique index. |
| T0.4.3 | Migration: `organization_settings` table | Backend | Creates `organization_settings` with FK to `organizations`, unique constraint on `organization_id`. |
| T0.4.4 | Migration: `org_invites` table | Backend | Creates `org_invites` with token unique index, status check constraint, `created_by` FK to `auth.users`. |
| T0.4.5 | Migration: `officers` table | Backend | Creates `officers` with FK to `auth.users` (unique) and `organizations`. Role check constraint. Soft delete columns. |
| T0.4.6 | Migration: `audit_log` table | Backend | Creates `audit_log` as append-only (no UPDATE/DELETE RLS). FKs to `auth.users`. |
| T0.4.7 | Generate Supabase TypeScript types | Backend / Frontend | Run `supabase gen types typescript` → output to `src/types/database.types.ts`. Types match all 6 platform-core tables. Re-run after each phase adds new migrations. |
| T0.4.8 | Validate migration sequence | Backend | `supabase db reset` applies all migrations from scratch with zero errors. Schema matches the platform-core subset of [05-database-schema.md](../05-database-schema.md). |

---

## E0.5: Firebase Configuration

> Set up Firebase project exclusively for GCash receipt image storage per [02-architecture-principles.md](../02-architecture-principles.md).

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.5.1 | Create Firebase project | Backend | Firebase project created. No additional Firebase services enabled (no Firestore, no FCM). Storage only. |
| T0.5.2 | Configure Storage bucket | Backend | Default bucket created. Storage rules restrict access to authenticated uploads only (write rules will be refined in Phase 3). |
| T0.5.3 | Set up Firebase emulator | Backend | `firebase emulators:start` runs Storage emulator locally. Emulator ports documented in dev setup guide. |
| T0.5.4 | Add Firebase config to environment | Backend / Frontend | Firebase config keys (`apiKey`, `storageBucket`, etc.) added to `.env.local.example`. `src/lib/firebase/storage.ts` reads from these env vars. |
| T0.5.5 | Verify emulator round-trip | QA | Upload a test file to the emulator Storage bucket via the Firebase client. Retrieve it via signed URL. Confirm the flow works locally. |

---

## E0.6: CI/CD Pipeline

> Automate code quality checks so that no broken code reaches the `main` branch.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.6.1 | Create GitHub Actions lint workflow | DevOps / Backend | `.github/workflows/lint.yml`: runs on push/PR to `main`. Steps: checkout, install deps, `npm run lint`. Fails on lint errors. |
| T0.6.2 | Create GitHub Actions type-check workflow | DevOps / Backend | Same workflow (or combined): runs `npx tsc --noEmit`. Fails on type errors. |
| T0.6.3 | Create GitHub Actions build workflow | DevOps / Backend | Same workflow (or combined): runs `npm run build`. Fails on build errors. Validates Server Actions, RSC imports. |
| T0.6.4 | Create Supabase migration validation step | DevOps / Backend | CI step that validates migration SQL files parse correctly. Option: use `supabase db lint` or a dry-run against a disposable local instance. |
| T0.6.5 | Configure branch protection rules | DevOps | `main` branch requires: CI checks passing, at least 1 PR review. Direct pushes to `main` are blocked. |

---

## E0.7: Seed Data & Dev Environment

> Provide a reproducible local environment with realistic test data so developers can work immediately.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.7.1 | Create System Admin seed script | Backend | Script creates 1–2 System Admin accounts via Supabase Auth Admin API. Sets `app_metadata: { "role": "system_admin" }`. Idempotent — safe to re-run. |
| T0.7.2 | Create `system_settings` seed | Backend | Inserts the single `system_settings` row with institution name, short name, platform name. |
| T0.7.3 | Create test organization seeds | Backend | Inserts 3 test organizations (one per tier: Basic, Plus, Premium) with corresponding `organization_settings` rows. |
| T0.7.4 | Create test officer seeds | Backend | Creates test officer auth users and `officers` rows for each test org. At least one admin per org. Premium org gets admin + manager + staff. |
| T0.7.5 | ~~Create test student seeds~~ | — | Deferred to Phase 2. `students` table does not exist yet. |
| T0.7.6 | Create seed runner script | Backend | `npm run seed` or `supabase db seed` command that runs all seed files in order. Documented in the setup guide. |
| T0.7.7 | Write developer setup guide | QA / Backend | `CONTRIBUTING.md` or `docs/setup.md` covering: prerequisites (Node, Docker, Supabase CLI, Firebase CLI), clone → install → env config → `supabase start` → `supabase db reset` → `npm run dev` → verify. Step-by-step, no assumptions. |

---

## QA Checklist

Before Phase 0 is marked complete, all of the following must pass:

| # | Check | Method |
|---|-------|--------|
| 1 | `npm run dev` starts Next.js at `localhost:3000` with zero errors | Manual |
| 2 | `supabase start` launches local Postgres, Auth, and Studio | Manual |
| 3 | `supabase db reset` applies all platform-core migrations from scratch with zero errors | Manual / CI |
| 4 | `supabase gen types typescript` generates types matching the 6 platform-core tables | Manual |
| 5 | Firebase emulator starts and accepts a test file upload | Manual |
| 6 | `npm run lint` passes with zero warnings | CI |
| 7 | `npx tsc --noEmit` passes with zero type errors | CI |
| 8 | `npm run build` completes successfully | CI |
| 9 | CI pipeline (GitHub Actions) runs and passes on a clean PR | CI |
| 10 | All route group layouts render without errors (`/login`, `/`, `/portal`, `/system-admin`) | Manual |
| 11 | Supabase server + browser clients connect to local instance | Manual |
| 12 | Seed data is present after running the seed script | Manual |
| 13 | A new developer can follow the setup guide and have the full environment running in <30 minutes | Manual (peer review) |

---

## Deliverables Summary

| Deliverable | Location |
|-------------|----------|
| Next.js project with feature-driven structure | `src/` |
| ShadCN UI primitives | `src/components/ui/` |
| Shared layout shells (stubs) | `src/app/(auth)/`, `(dashboard)/`, `(portal)/`, `(system-admin)/` |
| Supabase client utilities | `src/lib/supabase/` |
| Firebase client utility | `src/lib/firebase/` |
| React Query provider | `src/components/providers/` |
| Global types | `src/types/` |
| Database migrations (6 platform-core tables) | `supabase/migrations/` |
| Seed scripts | `supabase/seed.sql` or `scripts/seed.ts` |
| CI/CD workflows | `.github/workflows/` |
| Environment template | `.env.local.example` |
| Developer setup guide | `CONTRIBUTING.md` or `docs/setup.md` |
