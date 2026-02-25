# 02 — Architecture Principles

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** 2026  
> **Related Docs:** [System Overview](./01-system-overview.md) · [Next.js Guidelines](./03-nextjs-guidelines.md) · [Database Schema](./05-database-schema.md)

---

## Purpose

This document defines the architectural principles that govern **every line of code** in the VERIS codebase. These are not suggestions — they are constraints. Deviations require a PR with explicit justification and team consensus.

If you haven't read [01-system-overview.md](./01-system-overview.md) yet, stop and read it first.

---

## 1. Feature-Driven Architecture

VERIS uses a **feature-driven folder structure**. Code is organized by **business domain**, not by technical layer.

### Why Not Horizontal Layers?

A traditional structure like this:

```
src/
├── hooks/
├── components/
├── services/
├── utils/
```

Fails at scale because:

- **Coupling increases** — a single `hooks/` folder becomes a dumping ground with no clear ownership.
- **Discoverability drops** — finding all code related to "attendance" requires searching across 5+ top-level folders.
- **Deletion is dangerous** — removing a feature means hunting across the entire tree.

### The Feature-Driven Rule

Every domain gets its own folder under `src/features/`. Each feature folder is **self-contained** — it owns its components, hooks, actions, types, and utilities.

```
src/
├── app/                          # Next.js App Router (routes only)
│   ├── (auth)/                   # Auth route group (login, signup, etc.)
│   ├── (dashboard)/              # Officer dashboard route group
│   │   ├── attendance/
│   │   ├── members/
│   │   ├── financials/
│   │   ├── clearance/
│   │   └── settings/
│   ├── (portal)/                 # Student portal route group (Premium)
│   └── (system-admin)/           # System Admin route group (platform management)
│       ├── organizations/
│       ├── invites/
│       └── audit-log/
│
├── components/                   # Shared application-level components
│   ├── ui/                       # ShadCN UI primitives (Button, Dialog, etc.)
│   ├── layout/                   # Shell, Sidebar, Header, Footer
│   └── shared/                   # Cross-feature composed components (e.g., DataTable, StatusBadge)
│
├── features/                     # Feature modules (the core of the app)
│   ├── attendance/
│   │   ├── components/           # Feature-specific UI (AttendanceTable, CheckInForm)
│   │   ├── hooks/                # Feature-specific React Query hooks
│   │   ├── actions/              # Server Actions for this feature
│   │   ├── types/                # TypeScript types/interfaces
│   │   └── utils/                # Feature-specific helpers
│   ├── members/
│   ├── financials/
│   ├── clearance/
│   ├── organizations/
│   ├── auth/
│   └── system-admin/             # Platform management (System Admin only)
│       ├── components/           # System Admin UI (OrgTable, InviteForm, AuditViewer)
│       ├── hooks/                # React Query hooks for platform data
│       ├── actions/              # Server Actions (generateInviteUrl, suspendOrg, etc.)
│       └── types/                # System Admin types
│
├── lib/                          # Shared infrastructure utilities
│   ├── supabase/                 # Supabase client initialization (server + client)
│   ├── firebase/                 # Firebase Storage client initialization
│   └── utils.ts                  # Truly generic utilities (cn(), formatDate, etc.)
│
└── types/                        # Global shared types (database types, enums)
```

### Folder Rules

| Rule | Enforcement |
|------|-------------|
| **Route files are thin.** `app/` pages import from `features/` — they do not contain business logic, hooks, or complex UI. | Code review |
| **No global `src/hooks/` or `src/services/`.** If a hook serves one feature, it lives in that feature's `hooks/` folder. | Linting / code review |
| **Shared code must earn its place.** A utility only moves to `src/lib/` or `src/components/shared/` when it is used by **3+ features**. Until then, co-locate. | The Rule of Three |
| **Feature folders are deletable.** Removing `src/features/clearance/` should not break attendance or members. Cross-feature imports are allowed but must be explicit and minimal. | Architectural discipline |

### Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Feature folders | `kebab-case` | `src/features/financials/` |
| Component files | `PascalCase.tsx` | `AttendanceTable.tsx` |
| Hook files | `camelCase.ts` prefixed with `use` | `useAttendanceList.ts` |
| Server Action files | `camelCase.ts` | `checkInStudent.ts` |
| Type files | `camelCase.ts` | `attendance.types.ts` |
| Utility files | `camelCase.ts` | `formatCurrency.ts` |

---

## 2. ShadCN UI Component Strategy

VERIS uses **ShadCN UI** as its foundational component library. ShadCN provides unstyled, accessible, composable primitives that are copied into the project (not installed as a package dependency).

### Component Placement Rules

```
src/
├── components/
│   ├── ui/                     # ShadCN primitives — DO NOT put custom logic here
│   │   ├── button.tsx          # <Button />
│   │   ├── dialog.tsx          # <Dialog />
│   │   ├── input.tsx           # <Input />
│   │   ├── table.tsx           # <Table />
│   │   ├── select.tsx          # <Select />
│   │   ├── badge.tsx           # <Badge />
│   │   ├── card.tsx            # <Card />
│   │   └── ...                 # Other ShadCN components as needed
│   │
│   └── shared/                 # Composed components used across 3+ features
│       ├── DataTable.tsx        # Composes ui/table + sorting/filtering logic
│       ├── StatusBadge.tsx      # Composes ui/badge with domain-aware status colors
│       ├── ConfirmDialog.tsx    # Composes ui/dialog with standard confirm/cancel pattern
│       └── ...
│
├── features/
│   └── attendance/
│       └── components/
│           ├── AttendanceTable.tsx    # Composes shared/DataTable with attendance columns
│           ├── CheckInForm.tsx        # Composes ui/input, ui/button for check-in flow
│           └── EventCard.tsx          # Composes ui/card with event-specific layout
```

### ShadCN Rules

| Rule | Detail |
|------|--------|
| **`src/components/ui/` is ShadCN-only.** | These files are generated by the ShadCN CLI (`npx shadcn@latest add <component>`). Do not add custom business logic, domain types, or feature-specific behavior here. |
| **Customization happens via composition, not modification.** | If you need a domain-specific button variant, create a wrapper in your feature's `components/` folder or in `src/components/shared/` — do not edit the ShadCN source files directly unless changing the base design token. |
| **Import path is `@/components/ui/*`.** | All ShadCN imports use the `@/` alias. Example: `import { Button } from "@/components/ui/button"`. |
| **Shared compositions follow the Rule of Three.** | A composed component (e.g., `DataTable`) moves to `src/components/shared/` only when used by 3+ features. Before that, keep it in the feature folder. |
| **Theming via CSS variables.** | ShadCN theming is controlled through Tailwind CSS variables defined in `globals.css`. Premium organizations with custom branding override these variables at runtime. |

### Import Hierarchy

```
Feature Component (src/features/*/components/)
    └── composes → Shared Component (src/components/shared/)
                        └── composes → ShadCN Primitive (src/components/ui/)
                                            └── styled by → Tailwind CSS + CSS Variables
```

> **Anti-pattern:** A ShadCN primitive in `ui/` should never import from `features/` or `shared/`. Dependency flows **downward only**.

---

## 3. Thick DB, Thin API

This is the most important architectural principle in VERIS.

### Definition

> **Business logic and security enforcement live in PostgreSQL — not in the application layer.**

The Next.js Server Actions are **thin wrappers**. They handle request parsing, call the database, and return the result. They do not contain authorization checks that could be bypassed, business rule validation that could drift from the schema, or complex orchestration logic.

### What Lives Where

| Concern | Where | Mechanism |
|---------|-------|-----------|
| **Authorization** | PostgreSQL | Row Level Security (RLS) policies check the user's role and `organization_id` on every query. |
| **Feature gating** | PostgreSQL + Server Action | RLS policies check `organizations.tier`. Server Actions perform an early-exit tier check for UX (clear error messages). |
| **Data validation** | PostgreSQL | `CHECK` constraints, `NOT NULL`, foreign keys, unique indexes. |
| **Automated fines** | PostgreSQL | Triggers fire on attendance records to generate fines based on configurable rules. |
| **Audit logging** | PostgreSQL | Triggers on sensitive tables write to an `audit_log` table automatically. |
| **Computed status** | PostgreSQL | Database functions compute clearance status, balance calculations, etc. |
| **Request parsing** | Next.js Server Action | Zod schema validation on the incoming form data / arguments. |
| **Error formatting** | Next.js Server Action | Transform database errors into user-friendly messages. |
| **Cache invalidation** | Next.js + React Query | Server Actions call `revalidatePath()` / `revalidateTag()`; React Query `invalidateQueries()`. |

### Why This Matters

1. **Security cannot be bypassed.** Even if a Server Action is miscoded or a new endpoint is accidentally exposed, RLS blocks unauthorized data access at the database layer.
2. **Logic is centralized.** One source of truth for "can this user do this?" — the RLS policy — not scattered across 15 Server Actions.
3. **Testing is simpler.** Business rules can be tested with pure SQL. No need to spin up a Next.js server to verify authorization logic.

### Server Action Anatomy

A correctly structured Server Action follows this pattern:

```typescript
// src/features/attendance/actions/checkInStudent.ts
"use server";

import { z } from "zod";
import { createServerClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";

const CheckInSchema = z.object({
  eventId: z.string().uuid(),
  studentId: z.string().uuid(),
});

export async function checkInStudent(formData: FormData) {
  // 1. Parse & validate input
  const parsed = CheckInSchema.safeParse({
    eventId: formData.get("eventId"),
    studentId: formData.get("studentId"),
  });

  if (!parsed.success) {
    return { error: "Invalid input.", details: parsed.error.flatten() };
  }

  // 2. Call database — RLS handles auth + org scoping automatically
  const supabase = await createServerClient();
  const { data, error } = await supabase
    .from("attendance_records")
    .insert({
      event_id: parsed.data.eventId,
      student_id: parsed.data.studentId,
      checked_in_at: new Date().toISOString(),
    })
    .select()
    .single();

  if (error) {
    return { error: "Check-in failed. Student may already be checked in." };
  }

  // 3. Invalidate cache
  revalidatePath("/dashboard/attendance");

  return { success: true, data };
}
```

**What this action does NOT do:**

- ❌ Check if the user is authorized (RLS does it)
- ❌ Check if the user belongs to the correct organization (RLS does it)
- ❌ Check if the student is already checked in (unique constraint does it)
- ❌ Generate a fine for late check-in (database trigger does it)

---

## 4. Strict Data Flow Rules

Data flow in VERIS follows an explicit, non-negotiable pattern. Each method has exactly one job.

### Data Flow Matrix

| Operation | Method | Where It Runs | Example |
|-----------|--------|--------------|---------|
| **Initial page data** | React Server Component (`async` component) | Server | Fetching event list on page load |
| **Live / interactive data** | React Query (`useQuery`) | Client | Polling attendance count in real-time |
| **Real-time subscriptions** | Supabase Realtime | Client | Live attendance feed updates |
| **All writes (mutations)** | Next.js Server Actions | Server | Creating an event, checking in a student |
| **File uploads** | Server Action → Firebase signed URL | Server + Client | Uploading GCash receipt image |

### What Is Explicitly Banned

| Banned Pattern | Why |
|---------------|-----|
| `app/api/` routes for internal mutations | Server Actions replace this. API routes are only for external webhook receivers. |
| Client-side direct Supabase writes | All mutations must go through Server Actions for validation, logging, and cache invalidation. |
| `getServerSideProps` / `getStaticProps` | These are Pages Router patterns. VERIS uses App Router exclusively. |
| Global state managers (Redux, Zustand) for server data | React Query handles all server state. Local UI state uses `useState` / `useReducer`. |
| Fetching in `useEffect` | Use React Query's `useQuery` instead. Manages loading, error, caching, and refetch automatically. |

### RSC vs Client Component Decision Tree

```
Does this component need:
├── Browser APIs, event handlers, or useState/useEffect?
│   └── YES → Client Component ("use client")
│       └── Does it fetch server data?
│           ├── YES → Use React Query (useQuery)
│           └── NO  → Pure client component
│
└── NO → React Server Component (default)
    └── Does it fetch data?
        ├── YES → Fetch with Supabase server client directly in the component
        └── NO  → Static RSC (no data fetching)
```

---

## 5. Storage Split

Data storage in VERIS is strictly partitioned. No exceptions.

| Data Type | Storage | Access Pattern |
|-----------|---------|---------------|
| All structured/relational data | **Supabase PostgreSQL** | Supabase JS client (server-side for RSC/Actions, client-side for React Query) |
| User authentication state | **Supabase Auth** | `supabase.auth.getUser()` server-side; `supabase.auth.onAuthStateChange()` client-side |
| GCash payment receipt images | **Firebase Storage** | Upload via signed URL generated in a Server Action; display via signed read URL |
| Everything else | **Supabase** | Default to Supabase unless it's a binary file upload for payment receipts |

> **Hard rule:** Firebase Storage is used for **GCash receipt images only**. Do not store profile pictures, documents, exports, or any other file type in Firebase. If a future feature requires file storage beyond payment receipts, that decision requires an architecture review.

---

## 6. Multi-Tenancy Enforcement

Every org-scoped query **must** be automatically filtered by `organization_id`. This is not optional — it is enforced at the database layer.

### How It Works

1. When an officer logs in, their `organization_id` is stored in Supabase Auth user metadata (`app_metadata.organization_id`).
2. Every RLS policy on org-scoped tables includes:
   ```sql
   auth.jwt() ->> 'organization_id' = organization_id
   ```
3. This means no Server Action or client query ever needs to manually filter by org — **it's physically impossible to read another org's data** as long as RLS is enabled.

### Developer Responsibility

- **Never disable RLS** on a table that contains org-scoped data — not even "temporarily" for debugging. Use the Supabase service role key in a local script if you need to bypass RLS.
- **Never pass `organization_id` from the client** as a query parameter. Extract it from the authenticated session on the server.
- **Always test with multiple orgs** in development to verify tenant isolation.

---

## 7. Error Handling Strategy

Errors are handled at defined boundaries, not scattered throughout the code.

| Layer | Error Handling |
|-------|---------------|
| **Server Action** | `try/catch` around Supabase calls. Return `{ error: string }` or `{ success: true, data }`. Never throw from a Server Action. |
| **React Query** | `onError` callbacks for mutations. `error` state from `useQuery` for display. |
| **RSC** | `error.tsx` boundaries at the route segment level catch rendering failures. |
| **Database** | Constraint violations (unique, FK, check) return specific Postgres error codes. Server Actions map these to user-friendly messages. |

### Error Return Convention

All Server Actions return a discriminated union:

```typescript
// src/types/action-response.ts
type ActionResponse<T = void> =
  | { success: true; data: T }
  | { error: string; details?: Record<string, string[]> };
```

---

## 8. Summary of Non-Negotiables

| Principle | Violation Example | Correct Approach |
|-----------|------------------|-----------------|
| Feature-driven structure | Creating `src/hooks/useAttendance.ts` | `src/features/attendance/hooks/useAttendance.ts` |
| ShadCN in `ui/` only | Adding a `FineStatusBadge` to `src/components/ui/` | `src/features/financials/components/FineStatusBadge.tsx` composing `ui/badge` |
| Thick DB, Thin API | Checking user role in a Server Action with `if (role !== 'admin')` | RLS policy: `USING (auth.jwt() ->> 'role' = 'admin')` |
| Server Actions for writes | Creating `app/api/attendance/route.ts` for a POST endpoint | `src/features/attendance/actions/checkInStudent.ts` |
| No client-side Supabase writes | `supabase.from('events').insert(...)` in a Client Component | Call a Server Action that does the insert |
| Firebase for receipts only | Storing member profile photos in Firebase Storage | Store in Supabase Storage or reconsider the requirement |
| Rule of Three for shared code | Moving a utility to `src/lib/` after one feature uses it | Keep it co-located until 3+ features need it |

---

## 9. Related Documents

| Document | Contents |
|----------|----------|
| [01-system-overview.md](./01-system-overview.md) | Product context, tiers, tech stack, high-level diagrams |
| [03-nextjs-guidelines.md](./03-nextjs-guidelines.md) | Detailed RSC vs Client Component rules, Server Action patterns, ShadCN composition examples |
| [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) | Full RLS policy specs, RBAC definitions, Firebase upload workflow |
| [05-database-schema.md](./05-database-schema.md) | SQL table definitions, triggers, and stored functions |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | Build order: Basic → Plus → Premium |
