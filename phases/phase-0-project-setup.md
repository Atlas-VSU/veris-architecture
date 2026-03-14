# Phase 0 — Project Setup & Foundation

> **Phase Status:** Not Started  
> **Last Updated:** March 15, 2026  
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

## Required Reading Before This Phase

Every developer assigned to Phase 0 tasks must read these documents first:

| Document | Required Before |
|----------|----------------|
| [01-system-overview.md](../01-system-overview.md) | Everything |
| [02-architecture-principles.md](../02-architecture-principles.md) | E0.1, E0.2 |
| [04-supabase-firebase-auth.md](../04-supabase-firebase-auth.md) | E0.3, E0.4 |
| [05-database-schema.md](../05-database-schema.md) | E0.4 |
| [07-supabase-migrations-guide.md](../07-supabase-migrations-guide.md) | E0.3, E0.4 — covers CLI setup, migration naming conventions, and how to author schema/RLS/trigger/RPC files |

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
| T0.2.9 | Create Supabase admin client utility | Backend | `src/lib/supabase/admin.ts`: `createAdminClient()` using the `service_role` key. **Server-only** — `SUPABASE_SERVICE_ROLE_KEY` is not prefixed with `NEXT_PUBLIC_`. Used exclusively for auth admin operations (`inviteUserByEmail`, `updateUserById`) and cross-org queries. Never imported in Client Components. Per [04-supabase-firebase-auth.md](../04-supabase-firebase-auth.md#2-supabase-client-usage-rules). |
| T0.2.10 | Create Firebase client utility | Backend | `src/lib/firebase/storage.ts`: Firebase Admin SDK initialization and Storage bucket export. **Server-only** — used exclusively in Server Actions. No Firebase client SDK. Configured via env vars. |
| T0.2.11 | Create global shared types | Frontend / Backend | `src/types/database.types.ts`: placeholder for Supabase-generated types (generated in E0.4). `src/types/action-response.ts`: `ActionResponse<T>` discriminated union type per [02-architecture-principles.md](../02-architecture-principles.md). |

---

## E0.3: Supabase Provisioning

> Provision the Supabase project and configure the local development environment with the Supabase CLI. Read [07-supabase-migrations-guide.md](../07-supabase-migrations-guide.md) Sections 1–4 before starting this epic.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.3.1 | Create Supabase cloud project | Backend | Project created on `supabase.com`. Project URL and API keys obtained. Region selected (closest to target users — SEA). |
| T0.3.2 | Install and configure Supabase CLI | Backend | `supabase init` run in project root. `supabase/config.toml` present and committed. CLI version pinned in `docs/setup.md`. Developer has read [07-supabase-migrations-guide.md](../07-supabase-migrations-guide.md) Sections 1–4 before proceeding to E0.4. |
| T0.3.3 | Start Supabase local development | Backend | `supabase start` runs successfully. Local Postgres, Auth, and Studio accessible at documented ports. `.env.local` updated with local credentials. |
| T0.3.4 | Link local project to cloud | Backend | `supabase link --project-ref <ref>` connects local CLI to cloud project. `supabase migration list` shows correct project. |

---

## E0.4: Database Schema Migration

> Migrate only the **platform-core tables** needed for Phase 0 (setup) and Phase 1 (System Admin). Remaining tables are added as migrations in their respective phases — this avoids premature schema commitments for features that may evolve during planning.
>
> **Migration file naming:** Follow the prefix convention defined in [07-supabase-migrations-guide.md](../07-supabase-migrations-guide.md#2-migration-file-conventions). Platform-core migrations must be structured as:
> ```
> 20260101000000_schema_platform_core.sql
> 20260101000001_rls_platform_core.sql
> 20260101000002_rpc_auth_helper_functions.sql
> 20260101000003_trigger_audit_log.sql
> ```
> Use `supabase migration new <prefix_name>` to generate each file — never create migration files manually.

### Scope

| Table | Why It's Needed Now |
|-------|-------------------|
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
| T0.4.1 | Migration: platform-core schema | Backend | File: `*_schema_platform_core.sql`. Creates `system_settings`, `organizations`, `organization_settings`, `org_invites`, `officers`, `audit_log` with all columns, check constraints, FKs, and unique indexes per [05-database-schema.md](../05-database-schema.md). Soft delete columns (`deleted_at`, `deleted_by`) present on `officers`. |
| T0.4.2 | Migration: auth helper functions | Backend | File: `*_rpc_auth_helper_functions.sql`. Creates `auth.organization_id()`, `auth.user_role()`, `auth.has_tier()`, `auth.is_system_admin()` per [04-supabase-firebase-auth.md](../04-supabase-firebase-auth.md#3-row-level-security-rls). These must exist before RLS policies are created. |
| T0.4.3 | Migration: platform-core RLS policies | Backend | File: `*_rls_platform_core.sql`. Enables RLS on all 6 platform-core tables. Defines all policies per [04-supabase-firebase-auth.md](../04-supabase-firebase-auth.md#3-row-level-security-rls): org-scoped officer access, System Admin cross-org read, `audit_log` append-only (no UPDATE/DELETE for any role). |
| T0.4.4 | Migration: audit log trigger | Backend | File: `*_trigger_audit_log.sql`. Creates `fn_audit_log()` trigger function and attaches `trg_*_after_audit` triggers to `organizations`, `officers`, `org_invites` per [04-supabase-firebase-auth.md](../04-supabase-firebase-auth.md#6-database-triggers). |
| T0.4.5 | Validate migration sequence | Backend | `supabase db reset` applies all migrations from scratch with zero errors. Schema matches the platform-core subset of [05-database-schema.md](../05-database-schema.md). All triggers and RLS policies are present and correctly attached. |
| T0.4.6 | Generate Supabase TypeScript types | Backend / Frontend | Run `supabase gen types typescript --local > src/types/database.types.ts`. Types match all 6 platform-core tables. Document the `npm run gen:types` script in `package.json`. Re-run after each phase adds new migrations. |

---

## E0.5: Firebase Configuration

> Set up Firebase project exclusively for GCash receipt image storage per [02-architecture-principles.md](../02-architecture-principles.md#5-storage-split).

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.5.1 | Create Firebase project | Backend | Firebase project created. No additional Firebase services enabled (no Firestore, no FCM, no Analytics). Storage only. |
| T0.5.2 | Configure Storage bucket | Backend | Default bucket created. Storage security rules deny all direct client access (`allow read, write: if false`). All access via server-generated signed URLs only. |
| T0.5.3 | Set up Firebase emulator | Backend | `firebase emulators:start` runs Storage emulator locally. Emulator ports documented in `docs/setup.md`. |
| T0.5.4 | Add Firebase config to environment | Backend / Frontend | Firebase Admin SDK credentials (`FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY`, `FIREBASE_STORAGE_BUCKET`) added to `.env.local.example`. `src/lib/firebase/storage.ts` reads from these env vars. |
| T0.5.5 | Verify emulator round-trip | QA | Upload a test file to the emulator Storage bucket via the `createAdminClient` Firebase wrapper in a Server Action. Retrieve it via a signed URL. Confirm the full flow works locally with no client-side Firebase SDK involved. |

---

## E0.6: CI/CD Pipeline

> Automate code quality checks so that no broken code reaches the `main` branch.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.6.1 | Create GitHub Actions lint workflow | DevOps / Backend | `.github/workflows/ci.yml`: runs on push/PR to `main`. Step: `npm run lint`. Fails on lint errors. |
| T0.6.2 | Add type-check step | DevOps / Backend | Same workflow: runs `npx tsc --noEmit`. Fails on type errors. |
| T0.6.3 | Add build step | DevOps / Backend | Same workflow: runs `npm run build`. Fails on build errors. Validates Server Actions, RSC imports, and missing env var references. |
| T0.6.4 | Add migration validation step | DevOps / Backend | CI step runs `supabase db reset` against a local instance (via GitHub Actions service container or Supabase CLI's `--local` flag). Fails if any migration file produces an error. Catches SQL syntax errors and dependency ordering issues before they reach production. |
| T0.6.5 | Configure branch protection rules | DevOps | `main` branch requires: all CI checks passing, at least 1 PR review approval. Direct pushes to `main` are blocked. |

---

## E0.7: Seed Data & Dev Environment

> Provide a reproducible local environment with realistic test data so developers can work immediately after cloning.

### Tasks

| ID | Task | Assignee | Acceptance Criteria |
|----|------|----------|-------------------|
| T0.7.1 | Create System Admin seed script | Backend | Script creates 1–2 System Admin accounts via Supabase Auth Admin API (`createAdminClient()`). Sets `app_metadata: { "role": "system_admin" }`. Idempotent — safe to re-run without creating duplicates. |
| T0.7.2 | Create `system_settings` seed | Backend | `supabase/seed.sql` inserts the single `system_settings` row: institution name (`Visayas State University`), short name (`VSU`), platform name (`VERIS`). |
| T0.7.3 | Create test organization seeds | Backend | `supabase/seed.sql` inserts 3 test organizations — one per tier (Basic, Plus, Premium) — with fixed UUIDs for predictable cross-developer consistency. Corresponding `organization_settings` rows created. |
| T0.7.4 | Create test officer seeds | Backend | Creates test officer auth users and `officers` rows for each test org. Minimum: one `admin` per org. Premium org additionally gets one `manager` and one `staff`. All use predictable test credentials documented in `docs/setup.md`. |
| T0.7.5 | ~~Create test student seeds~~ | — | **Deferred to Phase 2.** `students` table does not exist yet. |
| T0.7.6 | Create seed runner script | Backend | `package.json` has `"seed": "supabase db reset"` script (reset replays all migrations + `seed.sql`). Documented in `docs/setup.md`. |
| T0.7.7 | Write developer setup guide | QA / Backend | `docs/setup.md` covers: prerequisites (Node version, Docker, Supabase CLI version, Firebase CLI version), clone → install → `.env.local` setup → `supabase start` → `npm run seed` → `npm run dev` → verify. Step-by-step with exact commands. A new developer should have the full environment running in under 30 minutes. |

---

## QA Checklist

Before Phase 0 is marked complete, **all** of the following must pass:

| # | Check | Method |
|---|-------|--------|
| 1 | `npm run dev` starts Next.js at `localhost:3000` with zero console errors | Manual |
| 2 | `supabase start` launches local Postgres, Auth, and Studio without errors | Manual |
| 3 | `supabase db reset` applies all platform-core migrations from scratch with zero errors | Manual |
| 4 | `supabase db reset` is **re-run a second time** and still passes — confirms idempotency of seed and trigger functions | Manual |
| 5 | `supabase gen types typescript` generates types matching all 6 platform-core tables | Manual |
| 6 | Firebase emulator starts and a test file upload round-trip succeeds via Server Action | Manual |
| 7 | `npm run lint` passes with zero warnings or errors | CI |
| 8 | `npx tsc --noEmit` passes with zero type errors | CI |
| 9 | `npm run build` completes successfully | CI |
| 10 | CI pipeline (GitHub Actions) runs and all steps pass on a clean PR | CI |
| 11 | All route group stub layouts render without errors: `/login`, `/`, `/portal`, `/system-admin` | Manual |
| 12 | `createServerClient()`, `createBrowserClient()`, and `createAdminClient()` each connect to the local Supabase instance correctly | Manual |
| 13 | `createAdminClient()` is confirmed absent from all Client Component files | Code review |
| 14 | RLS policies are active on all 6 platform-core tables — verified via Supabase Studio (`Authentication → Policies`) | Manual |
| 15 | Audit log trigger fires correctly: insert a test `officers` row, confirm a corresponding row appears in `audit_log` | Manual |
| 16 | Seed data is present after `npm run seed`: 1 `system_settings` row, 3 `organizations`, 5 `officers` | Manual |
| 17 | A new developer can follow `docs/setup.md` and have the full environment running in under 30 minutes | Manual (peer review) |

---

## Deliverables Summary

| Deliverable | Location |
|-------------|----------|
| Next.js project with feature-driven structure | `src/` |
| ShadCN UI primitives | `src/components/ui/` |
| Supabase server, browser, and admin client utilities | `src/lib/supabase/` |
| Firebase Admin Storage utility | `src/lib/firebase/` |
| Shared layout shells (stubs) | `src/app/(auth)/`, `(dashboard)/`, `(portal)/`, `(system-admin)/` |
| React Query provider | `src/components/providers/` |
| Global shared types + `ActionResponse<T>` | `src/types/` |
| Database migrations (platform-core) | `supabase/migrations/` |
| Seed data | `supabase/seed.sql` |
| System Admin seed script | `scripts/seed-auth-users.ts` |
| CI/CD workflows | `.github/workflows/ci.yml` |
| Environment variable template | `.env.local.example` |
| Developer setup guide | `docs/setup.md` |