# 04 — Supabase, Firebase & Authentication

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** 02/25/2026  
> **Related Docs:** [Architecture Principles](./02-architecture-principles.md) · [Next.js Guidelines](./03-nextjs-guidelines.md) · [Database Schema](./05-database-schema.md)

---

## Purpose

This document defines how VERIS uses **Supabase** (PostgreSQL, Auth, Realtime) and **Firebase Storage** — the rules for authentication flows, Row Level Security (RLS) policies, RPC functions, RBAC enforcement, and the GCash receipt upload workflow.

Read [02-architecture-principles.md](./02-architecture-principles.md) first. This document assumes you understand the Thick DB / Thin API principle. Everything here reinforces it: **the database is the security layer, not the application.**

---

## 1. Authentication Architecture

VERIS has **three distinct auth flows** — one for System Admins (platform), one for officers (org-level), and one for students. All use Supabase Auth but with different providers and metadata structures.

### Auth Flow Summary

| User Type | Auth Provider | Scope | Registration | Session Storage |
|-----------|--------------|-------|-------------|----------------|
| **System Admin** | Email / Password | Platform-wide | Seeded directly in the database — no self-registration | Supabase session cookie (httpOnly) |
| **Officer** | Email / Password | Organization | Org Admin registers via System Admin–generated invite URL; additional officers invited by Admin | Supabase session cookie (httpOnly) |
| **Student** | Google OAuth | Organization (Premium only) | Self-registration via Google Sign-In → org verification | Supabase session cookie (httpOnly) |

### System Admin Authentication

System Admins are **VERIS platform operators**. They are not part of any organization. Their accounts are **seeded directly in the database** (e.g., via a migration script or manual insert) — there is no signup form.

#### System Admin `app_metadata` Structure

```jsonc
{
  "role": "system_admin"
  // No organization_id — System Admins operate outside the tenant model
}
```

#### System Admin Provisioning

```
1. A developer/DBA creates the System Admin user via Supabase Auth Admin API
   or direct SQL insert into auth.users
2. app_metadata is set to: { "role": "system_admin" }
3. System Admin logs in at /login with email/password
4. Dashboard layout detects role = "system_admin" and redirects to /system-admin
5. System Admin has access to the (system-admin) route group only
```

> **No self-registration.** System Admin accounts are provisioned by the Atlas Dev Team. The number of System Admin accounts should be minimal (1–3 at most). Every System Admin action is audit-logged.

### Organization Onboarding via Invite URL

Organizations do **not** self-register. A System Admin generates an authenticated, time-limited invite URL and sends it to the organization’s designated contact (the future Org Admin).

#### Onboarding Flow

```
1. System Admin creates an org onboarding invite via the System Admin dashboard
   - Specifies: org name, contact email, assigned tier
   - Server Action generates a signed, time-limited invite token
   - Token is stored in an `org_invites` table with expiry and status
2. System Admin sends the invite URL to the organization contact
   (email delivery is manual or via an automated email service)
3. Org contact opens the invite URL → lands on a registration page
   - URL contains the invite token as a query parameter
4. Contact submits the signup form (email, password)
   - Server Action validates the invite token (not expired, not used)
   - Creates the auth user via supabase.auth.signUp()
   - Creates the `organizations` row with the pre-configured tier
   - Creates the `officers` row linking the user as admin
   - Sets app_metadata: { organization_id, role: "admin" }
   - Marks the invite token as consumed
5. Org Admin is redirected to the dashboard
```

> **Why not self-signup?** VERIS is a negotiated-contract SaaS. Organizations are onboarded after a sales/setup process. Self-registration would bypass the setup fee, tier negotiation, and data migration steps.

### Officer Authentication

Officers authenticate with **email and password** via Supabase Auth. The first officer (Org Admin) registers via a System Admin–generated invite URL. Additional officers are invited by the Org Admin.

#### Officer `app_metadata` Structure

```jsonc
// Stored in auth.users.raw_app_meta_data by Supabase
{
  "organization_id": "uuid",       // Tenant isolation key — used in every RLS policy
  "role": "admin" | "manager" | "staff"  // RBAC role — Premium only; Basic/Plus default to "admin"
}
```

> **Critical:** `app_metadata` is **server-writable only**. It cannot be modified by the client. This is what makes it safe to use in RLS policies via `auth.jwt()`.

#### Officer Signup Flow (Via System Admin Invite)

```
1. Org contact opens the System Admin-generated invite URL
2. Submits signup form (email, password)
3. Server Action validates the invite token, then calls supabase.auth.signUp()
4. A new row is created in `organizations` (tier set by invite)
5. A new row is created in `officers` linking the user to the org
6. app_metadata is set: { organization_id, role: "admin" }
7. Invite token is marked as consumed
8. Officer is redirected to the dashboard
```

> See "Organization Onboarding via Invite URL" above for the full System Admin → Org Admin flow.

#### Officer Invite Flow (Premium RBAC)

```
1. Admin invokes "Invite Officer" Server Action with email + role
2. Server Action calls supabase.auth.admin.inviteUserByEmail()
   with app_metadata: { organization_id, role }
3. Invited officer receives email, sets password
4. On first login, app_metadata is already set — officer is scoped to the org
```

### Student Authentication (Premium Only)

Students authenticate via **Google OAuth**. After OAuth, they land on a self-registration onboarding page where they submit their personal data (for data privacy compliance — the organization does not pre-populate student details).

#### Student `app_metadata` Structure

```jsonc
{
  "organization_id": "uuid",
  "role": "student",
  "is_verified": false | true   // Set to true by an officer after org verification
}
```

#### Student Registration Flow

```
1. Student clicks "Sign in with Google" on the student login page
2. Supabase Auth handles Google OAuth redirect
3. On callback, a database trigger checks:
   - If this Google email is linked to a student in any Premium org → set app_metadata
   - If not → redirect to an error page ("Your organization has not enabled the student portal")
4. Student lands on /portal/onboarding (self-registration form)
5. Student submits personal data (name, ID number, course, etc.)
6. Server Action writes to `students` table with is_verified = false
7. Officer reviews and verifies the student record
8. On verification, a Server Action updates app_metadata: is_verified = true
   (via supabase.auth.admin.updateUserById())
9. Student now has full portal access
```

> **Why self-registration?** Philippine data privacy law (RA 10173) requires informed consent. Students must submit their own data rather than having it pre-populated by the organization.

---

## 2. Supabase Client Usage Rules

VERIS maintains two Supabase client factories (defined in [03-nextjs-guidelines.md](./03-nextjs-guidelines.md#9-supabase-client-initialization)). The rules for when to use each are strict.

### Client Selection Matrix

| Context | Client | Function | Auth Handling |
|---------|--------|----------|---------------|
| React Server Component | `createServerClient()` | `@/lib/supabase/server` | Reads session from cookies; RLS applies using the user's JWT |
| Server Action | `createServerClient()` | `@/lib/supabase/server` | Same — reads cookies, RLS scopes writes |
| Client Component (React Query) | `createBrowserClient()` | `@/lib/supabase/client` | Uses browser cookie; RLS applies on all operations |
| Server-side admin operations | `createAdminClient()` | `@/lib/supabase/admin` | Uses `service_role` key; **bypasses RLS entirely** |

### Admin Client (Service Role)

The admin client is used **exclusively** for operations that require bypassing RLS — user management, metadata updates, and cross-org queries in internal tooling.

```typescript
// src/lib/supabase/admin.ts
import { createClient } from "@supabase/supabase-js";

export function createAdminClient() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!, // Server-only — never exposed to client
    { auth: { autoRefreshToken: false, persistSession: false } }
  );
}
```

### Admin Client Rules

| Rule | Detail |
|------|--------|
| **Never import in Client Components.** | `createAdminClient` must only be called in Server Actions or server-side scripts. The `SUPABASE_SERVICE_ROLE_KEY` is not prefixed with `NEXT_PUBLIC_` — Next.js will not bundle it client-side. |
| **Use sparingly.** | The admin client bypasses RLS. Every usage must be justified. Common cases: `auth.admin.inviteUserByEmail()`, `auth.admin.updateUserById()`, cross-org reporting. |
| **Never use for standard CRUD.** | If an officer is reading their own org's events, use `createServerClient()` — RLS handles scoping. |

---

## 3. Row Level Security (RLS)

RLS is the **primary security mechanism** in VERIS. Every table that contains org-scoped or user-scoped data must have RLS enabled with explicit policies. A table with RLS enabled and **no policies** grants **zero access** — this is the safe default.

### How RLS Works in VERIS

1. Every authenticated request carries a JWT with the user's `app_metadata` claims (`organization_id`, `role`).
2. Supabase exposes this via `auth.jwt()` in SQL.
3. RLS policies use `auth.jwt() ->> 'organization_id'` and `auth.jwt() ->> 'role'` to filter rows.

### JWT Claim Extraction (Helper Function)

To keep RLS policies readable, define reusable SQL functions:

```sql
-- Get the authenticated user's organization_id from JWT
CREATE OR REPLACE FUNCTION auth.organization_id()
RETURNS uuid
LANGUAGE sql
STABLE
AS $$
  SELECT (auth.jwt() -> 'app_metadata' ->> 'organization_id')::uuid;
$$;

-- Get the authenticated user's role from JWT
CREATE OR REPLACE FUNCTION auth.user_role()
RETURNS text
LANGUAGE sql
STABLE
AS $$
  SELECT auth.jwt() -> 'app_metadata' ->> 'role';
$$;

-- Check if the authenticated user's org is on a specific tier (or higher)
CREATE OR REPLACE FUNCTION auth.has_tier(required_tier text)
RETURNS boolean
LANGUAGE sql
STABLE
AS $$
  SELECT EXISTS (
    SELECT 1 FROM organizations
    WHERE id = auth.organization_id()
      AND tier >= required_tier  -- Relies on tier enum ordering: basic < plus < premium
  );
$$;

-- Check if the authenticated user is a System Admin
CREATE OR REPLACE FUNCTION auth.is_system_admin()
RETURNS boolean
LANGUAGE sql
STABLE
AS $$
  SELECT auth.jwt() -> 'app_metadata' ->> 'role' = 'system_admin';
$$;
```

### RLS Policy Patterns

Below are the **pattern templates** used across VERIS tables. Specific table policies are defined in [05-database-schema.md](./05-database-schema.md).

#### Pattern 1: Org-Scoped Read/Write (Most Tables)

The most common pattern. Officers can only access rows belonging to their organization.

```sql
-- Example: events table
ALTER TABLE events ENABLE ROW LEVEL SECURITY;

-- SELECT: officers can read their org's events
CREATE POLICY "Officers can read own org events"
  ON events FOR SELECT
  USING (organization_id = auth.organization_id());

-- INSERT: officers can create events in their org
CREATE POLICY "Officers can create events in own org"
  ON events FOR INSERT
  WITH CHECK (organization_id = auth.organization_id());

-- UPDATE: officers can update their org's events
CREATE POLICY "Officers can update own org events"
  ON events FOR UPDATE
  USING (organization_id = auth.organization_id())
  WITH CHECK (organization_id = auth.organization_id());

-- DELETE: officers can delete their org's events
CREATE POLICY "Officers can delete own org events"
  ON events FOR DELETE
  USING (organization_id = auth.organization_id());
```

#### Pattern 2: Org-Scoped + Role-Gated (Premium RBAC)

For operations restricted to specific roles. On Basic/Plus (single-account model), the officer's role is `admin` by default, so these policies pass transparently.

```sql
-- Example: only Admin can manage officers
CREATE POLICY "Only admins can manage officers"
  ON officers FOR ALL
  USING (
    organization_id = auth.organization_id()
    AND auth.user_role() = 'admin'
  )
  WITH CHECK (
    organization_id = auth.organization_id()
    AND auth.user_role() = 'admin'
  );

-- Example: Staff can only read events and attendance, not create
CREATE POLICY "Staff can read events"
  ON events FOR SELECT
  USING (organization_id = auth.organization_id());
  -- No INSERT/UPDATE/DELETE policy for staff — they simply can't write

CREATE POLICY "Managers and admins can write events"
  ON events FOR INSERT
  WITH CHECK (
    organization_id = auth.organization_id()
    AND auth.user_role() IN ('admin', 'manager')
  );
```

#### Pattern 3: Tier-Gated Access

For tables that only exist in higher tiers (e.g., `fines` requires Plus+, `fine_appeals` requires Premium).

```sql
-- Example: fines table — requires Plus or Premium tier
CREATE POLICY "Officers can read fines (Plus+)"
  ON fines FOR SELECT
  USING (
    organization_id = auth.organization_id()
    AND auth.has_tier('plus')
  );

-- Example: fine_appeals — requires Premium tier
CREATE POLICY "Students can create appeals (Premium)"
  ON fine_appeals FOR INSERT
  WITH CHECK (
    student_id = auth.uid()
    AND auth.has_tier('premium')
  );
```

#### Pattern 4: Student Self-Access (Premium Portal)

Students can only read their own data and submit their own records.

```sql
-- Students can read only their own fines
CREATE POLICY "Students read own fines"
  ON fines FOR SELECT
  USING (
    student_id = auth.uid()
    AND auth.user_role() = 'student'
  );

-- Students can insert payment records for themselves only
CREATE POLICY "Students submit own payments"
  ON payments FOR INSERT
  WITH CHECK (
    student_id = auth.uid()
    AND auth.user_role() = 'student'
  );
```

#### Pattern 5: System Admin Cross-Org Access

System Admins can read organization-level data across all tenants for platform management. However, access to **student PII** (personally identifiable information) is restricted and audit-logged.

```sql
-- System Admin can read all organizations
CREATE POLICY "System admin can read all organizations"
  ON organizations FOR SELECT
  USING (auth.is_system_admin());

-- System Admin can update organizations (tier changes, suspension, etc.)
CREATE POLICY "System admin can update organizations"
  ON organizations FOR UPDATE
  USING (auth.is_system_admin())
  WITH CHECK (auth.is_system_admin());

-- System Admin can read org_invites (platform-level table, no org scoping)
CREATE POLICY "System admin can manage org invites"
  ON org_invites FOR ALL
  USING (auth.is_system_admin())
  WITH CHECK (auth.is_system_admin());

-- System Admin can read the audit log across all orgs
CREATE POLICY "System admin can read audit log"
  ON audit_log FOR SELECT
  USING (auth.is_system_admin());
```

> **Data Privacy Constraint:** System Admin policies on tables containing student PII (e.g., `students`, `payments`, `fines`) should **not** grant blanket `SELECT` access. If a System Admin needs to access student data for a support case, it must go through a `SECURITY DEFINER` RPC function that logs the access reason to the audit table. See [Section 5: RBAC Enforcement](#5-rbac-enforcement) for the full data privacy rules.

### RLS Non-Negotiables

| Rule | Rationale |
|------|-----------|
| **Every org-scoped table has RLS enabled.** | No exceptions. A table without RLS is a cross-tenant data leak waiting to happen. |
| **Never disable RLS "temporarily."** | Use the `createAdminClient()` (service role) locally if you need to bypass RLS for debugging or seeding. |
| **Never pass `organization_id` from the client.** | It is always extracted from the JWT via `auth.organization_id()`. Client-supplied org IDs are ignored by RLS. |
| **Test with multiple orgs.** | Every developer should seed at least two organizations and verify that Org A's officer cannot read Org B's data. |
| **RLS policies are declarative documentation.** | A policy named `"Staff can read events"` is both enforcement and documentation of the access rule. Use clear, descriptive policy names. |

---

## 4. Supabase RPC Functions

VERIS uses **Supabase RPC** (Remote Procedure Call) to invoke PostgreSQL functions for operations that go beyond simple CRUD — complex aggregations, multi-table transactions, computed results, and business logic that must be atomic.

This aligns with the **Thick DB, Thin API** principle: the database owns the logic, and the app layer invokes it via `.rpc()`.

### When to Use RPC vs Direct Queries

| Use Direct Query (`.from().select()`) | Use RPC (`.rpc()`) |
|----------------------------------------|--------------------|
| Simple CRUD on a single table | Multi-table operations that must be atomic |
| Filtered reads with straightforward `WHERE` clauses | Complex aggregations (e.g., financial summaries, clearance status) |
| Paginated list queries | Operations requiring `SECURITY DEFINER` (elevated privilege within controlled scope) |
| Inserting a single row | Business logic with conditional branching (e.g., "create fine only if attendance rule is violated") |

### RPC Function Conventions

| Convention | Detail |
|-----------|--------|
| **Naming** | `snake_case`, verb-first: `get_student_balance`, `compute_clearance_status`, `process_payment_verification` |
| **Schema** | All RPC functions live in the `public` schema (Supabase default) unless they are auth helpers (those go in the `auth` schema). |
| **Security** | Default to `SECURITY INVOKER` — function executes with the calling user's permissions (RLS applies). Use `SECURITY DEFINER` only when the function needs to read data the calling user cannot (e.g., cross-referencing org settings during a student action). |
| **Return Types** | Return explicit types (`RETURNS TABLE`, `RETURNS json`, `RETURNS boolean`). Avoid `RETURNS void` unless the function is purely a side-effect trigger. |
| **Input Validation** | Validate inputs with `IF` / `RAISE EXCEPTION` at the top of the function body. Don't rely solely on the caller for validation. |
| **RLS Awareness** | `SECURITY INVOKER` functions respect RLS automatically. `SECURITY DEFINER` functions bypass RLS — these must manually enforce `organization_id` scoping internally. |

### RPC Function Structure Template

```sql
-- Function: get_student_balance
-- Purpose: Computes a student's total financial balance (fees + fines - payments)
-- Security: INVOKER (RLS applies — student can only compute their own balance)

CREATE OR REPLACE FUNCTION get_student_balance(p_student_id uuid)
RETURNS json
LANGUAGE plpgsql
STABLE                      -- Does not modify data; safe for read replicas
SECURITY INVOKER            -- RLS applies; caller must have access to the rows
AS $$
DECLARE
  v_total_fees    numeric := 0;
  v_total_fines   numeric := 0;
  v_total_paid    numeric := 0;
  v_balance       numeric := 0;
BEGIN
  -- 1. Sum all fees assigned to this student
  SELECT COALESCE(SUM(amount), 0) INTO v_total_fees
  FROM fee_assignments
  WHERE student_id = p_student_id;

  -- 2. Sum all fines for this student
  SELECT COALESCE(SUM(amount), 0) INTO v_total_fines
  FROM fines
  WHERE student_id = p_student_id
    AND status != 'waived';

  -- 3. Sum all verified payments
  SELECT COALESCE(SUM(amount), 0) INTO v_total_paid
  FROM payments
  WHERE student_id = p_student_id
    AND status = 'verified';

  -- 4. Compute balance
  v_balance := (v_total_fees + v_total_fines) - v_total_paid;

  RETURN json_build_object(
    'total_fees', v_total_fees,
    'total_fines', v_total_fines,
    'total_paid', v_total_paid,
    'balance', v_balance
  );
END;
$$;
```

### Calling RPC from the Application Layer

#### From a React Server Component (Initial Load)

```typescript
// src/app/(portal)/page.tsx
import { createServerClient } from "@/lib/supabase/server";
import { StudentDashboard } from "@/features/portal/components/StudentDashboard";

export default async function PortalHomePage() {
  const supabase = await createServerClient();
  const { data: { user } } = await supabase.auth.getUser();

  // Call RPC function — RLS scopes to the student's data automatically
  const { data: balance } = await supabase.rpc("get_student_balance", {
    p_student_id: user!.id,
  });

  return <StudentDashboard balance={balance} />;
}
```

#### From a React Query Hook (Live/Client Data)

```typescript
// src/features/portal/hooks/useStudentBalance.ts
"use client";

import { useQuery } from "@tanstack/react-query";
import { createBrowserClient } from "@/lib/supabase/client";

export function useStudentBalance(studentId: string) {
  const supabase = createBrowserClient();

  return useQuery({
    queryKey: ["student-balance", studentId],
    queryFn: async () => {
      const { data, error } = await supabase.rpc("get_student_balance", {
        p_student_id: studentId,
      });
      if (error) throw error;
      return data;
    },
    refetchInterval: 60_000, // Refresh balance every 60 seconds
  });
}
```

#### From a Server Action (Post-Mutation Computation)

```typescript
// src/features/financials/actions/verifyPayment.ts
"use server";

import { z } from "zod";
import { createServerClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";
import type { ActionResponse } from "@/types/action-response";

const VerifyPaymentSchema = z.object({
  paymentId: z.string().uuid(),
});

export async function verifyPayment(
  input: z.infer<typeof VerifyPaymentSchema>
): Promise<ActionResponse> {
  const parsed = VerifyPaymentSchema.safeParse(input);
  if (!parsed.success) {
    return { error: "Invalid input.", details: parsed.error.flatten().fieldErrors };
  }

  const supabase = await createServerClient();

  // 1. Update payment status — RLS ensures officer belongs to the correct org
  const { error } = await supabase
    .from("payments")
    .update({ status: "verified", verified_at: new Date().toISOString() })
    .eq("id", parsed.data.paymentId);

  if (error) {
    return { error: "Failed to verify payment." };
  }

  // 2. Recompute clearance status via RPC (atomic, multi-table logic in DB)
  //    This function checks if the student has cleared all fees/fines after this payment.
  await supabase.rpc("recompute_clearance_status", {
    p_payment_id: parsed.data.paymentId,
  });

  revalidatePath("/financials/payments");
  return { success: true, data: undefined };
}
```

### RPC vs Triggers vs Direct Queries — Decision Table

| Scenario | Mechanism | Why |
|----------|-----------|-----|
| Officer reads a list of events | Direct query (`.from("events").select()`) | Simple, single-table read. RLS scopes by org. |
| Student views their balance | RPC (`get_student_balance`) | Multi-table aggregation (fees + fines - payments). Must be atomic and consistent. |
| Officer verifies a payment | Direct query (`.update()`) + RPC follow-up | Write is simple (single update). Post-write clearance recomputation is complex → RPC. |
| Attendance check-in triggers a late fine | Trigger (`AFTER INSERT ON attendance_records`) | Automated, event-driven. No human invocation needed. |
| Dashboard shows org-wide financial summary | RPC (`get_financial_summary`) | Aggregates across multiple tables with tier-aware logic. Too complex for a `.select()`. |
| Student submits a fine appeal | Direct query (`.insert()`) | Simple single-row insert. Validation via `CHECK` constraints + RLS. |

### `SECURITY DEFINER` Guidelines

Most RPC functions should be `SECURITY INVOKER` (the default behavior when explicitly set). Use `SECURITY DEFINER` only when:

1. The function must read rows the calling user cannot access (e.g., checking org settings during a student action).
2. The function must write to an audit table that users do not have direct INSERT access to.

When using `SECURITY DEFINER`:

```sql
-- SECURITY DEFINER: manually enforce org scoping since RLS is bypassed
CREATE OR REPLACE FUNCTION process_fine_appeal(p_appeal_id uuid)
RETURNS json
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER            -- Runs as the function owner (superuser-level)
SET search_path = public    -- Prevent search_path hijacking
AS $$
DECLARE
  v_appeal   record;
  v_caller_org uuid;
BEGIN
  -- 1. Manually extract caller's org (RLS is bypassed in DEFINER context)
  v_caller_org := (current_setting('request.jwt.claims', true)::json -> 'app_metadata' ->> 'organization_id')::uuid;

  -- 2. Fetch the appeal and verify org match
  SELECT * INTO v_appeal FROM fine_appeals WHERE id = p_appeal_id;
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Appeal not found';
  END IF;

  -- 3. Verify the fine belongs to the caller's org
  IF NOT EXISTS (
    SELECT 1 FROM fines f
    WHERE f.id = v_appeal.fine_id
      AND f.organization_id = v_caller_org
  ) THEN
    RAISE EXCEPTION 'Access denied';
  END IF;

  -- 4. Process the appeal (business logic)
  -- ... (update statuses, write audit log, etc.)

  RETURN json_build_object('status', 'processed');
END;
$$;
```

---

## 5. RBAC Enforcement

VERIS has five roles across two scopes. RBAC is enforced at the **database layer** via RLS — not in the application.

### Role Hierarchy

```
System Admin (platform-level — cross-org, no org membership)
  └── Admin (org-level — full org access)
       └── Manager (operational access — no role/subscription management)
            └── Staff (attendance + member directory — read-heavy, minimal writes)
                 └── Student (self-service — own data only, Premium tier)
```

> **Important distinction:** System Admin and the four org-level roles (Admin/Manager/Staff/Student) operate in **completely separate scopes**. A System Admin does not belong to any organization. An Org Admin has full access within their org but zero cross-org visibility.

### Role-Permission Matrix

This matrix governs RLS policy design. For the full feature-level breakdown of org roles, see [features-tiers-summary.md](./features-tiers-summary.md#3-role-based-access-control-rbac--premium-feature).

#### Organization-Level Roles

| Table / Operation | Admin | Manager | Staff | Student |
|-------------------|:-----:|:-------:|:-----:|:-------:|
| `events` SELECT | ✓ | ✓ | ✓ | — |
| `events` INSERT/UPDATE | ✓ | ✓ | — | — |
| `events` DELETE | ✓ | ✓ | — | — |
| `attendance_records` SELECT | ✓ | ✓ | ✓ | Own records |
| `attendance_records` INSERT | ✓ | ✓ | ✓ | — |
| `students` SELECT | ✓ | ✓ | ✓ | Own row |
| `students` INSERT/UPDATE | ✓ | ✓ | — | Own row (onboarding) |
| `students` DELETE/ARCHIVE | ✓ | — | — | — |
| `fines` SELECT | ✓ | ✓ | — | Own fines |
| `fines` INSERT/UPDATE | ✓ | ✓ | — | — |
| `payments` SELECT | ✓ | ✓ | — | Own payments |
| `payments` INSERT | ✓ | ✓ | — | Own payments |
| `payments` UPDATE (verify) | ✓ | ✓ | — | — |
| `fine_appeals` SELECT | ✓ | ✓ | — | Own appeals |
| `fine_appeals` INSERT | — | — | — | Own appeals |
| `officers` SELECT/MANAGE | ✓ | — | — | — |
| `organizations` UPDATE | ✓ | — | — | — |

#### System Admin (Platform-Level)

| Table / Operation | System Admin | Notes |
|-------------------|:------------:|-------|
| `organizations` SELECT | ✓ | All orgs — for listing, filtering, status overview |
| `organizations` UPDATE | ✓ | Tier changes, suspension, metadata updates |
| `organizations` DELETE | — | Orgs are soft-deleted (status change), never hard-deleted |
| `org_invites` ALL | ✓ | Full CRUD — generate, revoke, list invite URLs |
| `officers` SELECT | ✓ | Cross-org — for support (e.g., “which officer is the admin?”) |
| `audit_log` SELECT | ✓ | Platform-wide audit trail |
| `students` SELECT | **Restricted** | No blanket access. Must use audited RPC function with access reason. |
| `fines` / `payments` SELECT | **Restricted** | Same — no blanket access to student financial data. |
| `events` / `attendance_records` | — | System Admin does not need operational event data. |

### System Admin Data Privacy Constraints

VERIS operates under the Philippine Data Privacy Act (RA 10173). System Admins have elevated platform access, but this access is **not carte blanche** over student PII.

| Rule | Detail |
|------|--------|
| **No blanket student data access.** | System Admin RLS policies do **not** grant `SELECT` on `students`, `fines`, `payments`, or `fine_appeals`. |
| **Audited access via RPC only.** | If a System Admin needs student data for a legitimate support case, they must call a `SECURITY DEFINER` RPC function (e.g., `sysadmin_lookup_student`) that requires an `access_reason` parameter and writes to the audit log before returning data. |
| **Aggregate metrics are permitted.** | System Admin can access aggregate/anonymized data (e.g., "Org X has 150 active students", "Total payments this month: ₱15,000") via RPC functions that return counts/sums without exposing individual records. |
| **Audit log is immutable.** | The `audit_log` table has no `UPDATE` or `DELETE` policies for any role. System Admin can read it but never modify it. |

#### Audited Student Data Access (RPC Example)

```sql
-- System Admin: look up a student record with mandatory audit trail
CREATE OR REPLACE FUNCTION sysadmin_lookup_student(
  p_student_id uuid,
  p_access_reason text   -- Required: e.g., "Support ticket #1234"
)
RETURNS json
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_student record;
BEGIN
  -- 1. Verify caller is System Admin
  IF NOT auth.is_system_admin() THEN
    RAISE EXCEPTION 'Access denied: system_admin role required';
  END IF;

  -- 2. Validate access reason is provided
  IF p_access_reason IS NULL OR TRIM(p_access_reason) = '' THEN
    RAISE EXCEPTION 'Access reason is required for student data lookup';
  END IF;

  -- 3. Log the access BEFORE returning data
  INSERT INTO audit_log (
    table_name, record_id, action, old_data, new_data,
    performed_by, organization_id, created_at
  ) VALUES (
    'students', p_student_id, 'SYSADMIN_READ',
    NULL, json_build_object('access_reason', p_access_reason),
    auth.uid(), NULL, NOW()
  );

  -- 4. Fetch and return the student record
  SELECT * INTO v_student FROM students WHERE id = p_student_id;
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Student not found';
  END IF;

  RETURN row_to_json(v_student);
END;
$$;
```

### RLS Policy Naming Convention

Policies are named in lowercase with a readable format to serve as documentation:

```
{role(s)} can {operation} {scope}
```

Examples:
- `"officers can read own org events"`
- `"admins and managers can create fines"`
- `"students can read own fines"`
- `"staff can insert attendance records"`
- `"system admin can read all organizations"`
- `"system admin can manage org invites"`

### Role-Aware RPC Example

When an RPC function needs to branch by role (rare — prefer separate policies on tables):

```sql
-- Returns different data depending on the caller's role
CREATE OR REPLACE FUNCTION get_financial_summary()
RETURNS json
LANGUAGE plpgsql
STABLE
SECURITY INVOKER
AS $$
DECLARE
  v_role text;
  v_org  uuid;
  v_result json;
BEGIN
  v_role := auth.user_role();
  v_org  := auth.organization_id();

  -- Staff should not be calling this, but defense-in-depth:
  IF v_role = 'staff' THEN
    RAISE EXCEPTION 'Insufficient permissions';
  END IF;

  SELECT json_build_object(
    'total_collected', COALESCE(SUM(amount) FILTER (WHERE status = 'verified'), 0),
    'total_pending',   COALESCE(SUM(amount) FILTER (WHERE status = 'pending'), 0),
    'total_fines',     (SELECT COALESCE(SUM(amount), 0) FROM fines WHERE organization_id = v_org AND status != 'waived')
  ) INTO v_result
  FROM payments
  WHERE organization_id = v_org;

  RETURN v_result;
END;
$$;
```

---

## 6. Database Triggers

Triggers handle **automated, event-driven logic** that must run reliably regardless of which application endpoint initiated the change. They are the backbone of the Thick DB philosophy.

### Trigger Conventions

| Convention | Detail |
|-----------|--------|
| **Naming** | `trg_{table}_{timing}_{event}` — e.g., `trg_attendance_records_after_insert` |
| **Function Naming** | `fn_{description}` — e.g., `fn_generate_absence_fine`, `fn_audit_log` |
| **One trigger, one concern.** | Don't bundle unrelated logic into a single trigger function. Separate fine generation from audit logging. |
| **Idempotency** | Trigger functions must be safe to re-run. Use `INSERT ... ON CONFLICT DO NOTHING` where appropriate. |
| **Performance** | Triggers run synchronously within the transaction. Keep them fast. Offload heavy work (emails, notifications) to a queue/edge function. |

### Example: Audit Log Trigger

Every sensitive table writes to an `audit_log` on INSERT, UPDATE, and DELETE. This trigger is reusable across tables.

```sql
-- Reusable audit log trigger function
CREATE OR REPLACE FUNCTION fn_audit_log()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER            -- Writes to audit_log even if the user lacks direct INSERT
SET search_path = public
AS $$
BEGIN
  INSERT INTO audit_log (
    table_name,
    record_id,
    action,
    old_data,
    new_data,
    performed_by,
    organization_id,
    created_at
  ) VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,                    -- 'INSERT', 'UPDATE', or 'DELETE'
    CASE WHEN TG_OP != 'INSERT' THEN row_to_json(OLD) ELSE NULL END,
    CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW) ELSE NULL END,
    auth.uid(),
    COALESCE(NEW.organization_id, OLD.organization_id),
    NOW()
  );

  RETURN COALESCE(NEW, OLD);
END;
$$;

-- Attach to any sensitive table:
CREATE TRIGGER trg_payments_audit
  AFTER INSERT OR UPDATE OR DELETE ON payments
  FOR EACH ROW EXECUTE FUNCTION fn_audit_log();

CREATE TRIGGER trg_fines_audit
  AFTER INSERT OR UPDATE OR DELETE ON fines
  FOR EACH ROW EXECUTE FUNCTION fn_audit_log();
```

### Example: Auto-Generate Meeting Fine (Premium Automation)

When an event is archived (closed), a trigger evaluates attendance records and generates fines for absent students. This is a Premium-only feature — the trigger checks the org's tier before executing.

```sql
-- Trigger function: generate fines for absent students when an event is archived
CREATE OR REPLACE FUNCTION fn_generate_absence_fines()
RETURNS trigger
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_tier    text;
  v_fine_amount numeric;
  v_fine_type_id uuid;
BEGIN
  -- Only fire when status changes to 'archived'
  IF NEW.status != 'archived' OR OLD.status = 'archived' THEN
    RETURN NEW;
  END IF;

  -- Check if org is Premium (automated fines are Premium-only)
  SELECT tier INTO v_org_tier FROM organizations WHERE id = NEW.organization_id;
  IF v_org_tier != 'premium' THEN
    RETURN NEW;
  END IF;

  -- Look up the configured absence fine type for this org
  SELECT id, default_amount INTO v_fine_type_id, v_fine_amount
  FROM fine_types
  WHERE organization_id = NEW.organization_id
    AND trigger_source = 'absence'
    AND is_active = true
  LIMIT 1;

  IF v_fine_type_id IS NULL THEN
    RETURN NEW; -- No absence fine type configured; skip
  END IF;

  -- Insert fines for students who are members but have no attendance record
  INSERT INTO fines (student_id, organization_id, fine_type_id, event_id, amount, status, created_at)
  SELECT
    s.id,
    NEW.organization_id,
    v_fine_type_id,
    NEW.id,
    v_fine_amount,
    'pending',
    NOW()
  FROM students s
  WHERE s.organization_id = NEW.organization_id
    AND s.is_active = true
    AND NOT EXISTS (
      SELECT 1 FROM attendance_records ar
      WHERE ar.event_id = NEW.id AND ar.student_id = s.id
    )
  ON CONFLICT DO NOTHING;  -- Idempotent: won't duplicate if trigger re-fires

  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_events_after_update_archive
  AFTER UPDATE ON events
  FOR EACH ROW EXECUTE FUNCTION fn_generate_absence_fines();
```

> **Note:** This is an example of the automated fine mechanism. The full fine-type configuration schema and all trigger variants are defined in [05-database-schema.md](./05-database-schema.md).

---

## 7. Firebase Storage — GCash Receipt Upload Workflow

Firebase Storage is used for **one thing only**: GCash payment receipt images. All other data lives in Supabase.

### Why Firebase Storage?

| Concern | Why Not Supabase Storage? |
|---------|--------------------------|
| **Signed URL generation** | Firebase provides straightforward signed URL generation with time-limited access via the Admin SDK. |
| **Independent scaling** | Receipt image storage scales independently from the relational database. |
| **CDN delivery** | Firebase Storage integrates with Google's CDN for fast image serving. |

> If Supabase Storage capabilities evolve to match these requirements, this decision may be revisited via an architecture PR.

### Firebase Client Initialization

```typescript
// src/lib/firebase/config.ts
import { initializeApp, getApps, cert } from "firebase-admin/app";
import { getStorage } from "firebase-admin/storage";

// Server-side only — Firebase Admin SDK
function getFirebaseAdmin() {
  if (getApps().length === 0) {
    initializeApp({
      credential: cert({
        projectId: process.env.FIREBASE_PROJECT_ID!,
        clientEmail: process.env.FIREBASE_CLIENT_EMAIL!,
        privateKey: process.env.FIREBASE_PRIVATE_KEY!.replace(/\\n/g, "\n"),
      }),
      storageBucket: process.env.FIREBASE_STORAGE_BUCKET!,
    });
  }
  return getStorage().bucket();
}

export { getFirebaseAdmin };
```

### Upload Flow: Server-Generated Signed URL

Clients never talk to Firebase directly. The Server Action generates a signed upload URL, the client uploads to it, and the Server Action records the reference in Supabase.

```
┌───────────────┐       1. Request upload URL          ┌──────────────────┐
│  Client        │ ──────────────────────────────────▶│  Server Action    │
│  (Student      │                                     │  (uploadReceipt)  │
│   Portal)      │       2. Return signed URL          │                   │
│                │ ◀──────────────────────────────────│  • Validates user │
│                │                                     │  • Generates URL  │
│                │       3. PUT image to signed URL    │                   │
│                │ ─────────────────────────────────▶ │  Firebase Storage │
│                │                                     │                   │
│                │       4. Confirm upload             │                   │
│                │ ──────────────────────────────────▶│  Server Action    │
│                │                                     │  (confirmUpload)  │
│                │                                     │  • Writes to DB   │
│                │       5. Success response           │  • Links receipt  │
│                │ ◀──────────────────────────────────│     to payment    │
└───────────────┘                                      └──────────────────┘
```


### Firebase Storage Rules

Firebase Security Rules restrict all direct access. Only signed URLs (generated by the server) can read or write.

```javascript
// Firebase Storage Security Rules (firebase.storage.rules)
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /receipts/{orgId}/{fileName} {
      // Deny all direct client access
      // All reads/writes happen via server-generated signed URLs
      allow read, write: if false;
    }
  }
}
```

> **Why deny all?** The Firebase Admin SDK (used in Server Actions) can generate signed URLs regardless of security rules. By denying all direct access, we ensure that only server-generated, time-limited URLs can access receipt images. No client can read or write directly.

### Firebase Usage Rules

| Rule | Detail |
|------|--------|
| **Receipts only.** | Firebase Storage stores GCash payment receipt images. Nothing else. Profile photos, exports, documents — all of these stay in Supabase or are not stored at all. |
| **No client-side Firebase SDK.** | The client never imports `firebase/storage`. All Firebase interaction happens through Server Actions that use the Firebase Admin SDK. |
| **Signed URLs are ephemeral.** | Upload URLs expire in 15 minutes. Read URLs expire in 30 minutes. Do not store signed URLs in the database — store the `filePath` and regenerate signed URLs on demand. |
| **File paths are deterministic.** | Format: `receipts/{organization_id}/{payment_id}.{ext}`. This makes lookups easy and prevents orphaned files. |

---

## 8. Environment Variables

All Supabase and Firebase credentials are managed via environment variables. Next.js scoping rules apply.

### Variable Reference

| Variable | Scope | Used By |
|----------|-------|---------|
| `NEXT_PUBLIC_SUPABASE_URL` | Public (client + server) | Browser client, server client |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Public (client + server) | Browser client, server client (with RLS) |
| `SUPABASE_SERVICE_ROLE_KEY` | **Server only** | Admin client (bypasses RLS) |
| `FIREBASE_PROJECT_ID` | **Server only** | Firebase Admin SDK |
| `FIREBASE_CLIENT_EMAIL` | **Server only** | Firebase Admin SDK |
| `FIREBASE_PRIVATE_KEY` | **Server only** | Firebase Admin SDK |
| `FIREBASE_STORAGE_BUCKET` | **Server only** | Firebase Admin SDK |

### Rules

- **`NEXT_PUBLIC_` prefix** = safe to expose to the browser. Only the Supabase URL and anon key have this prefix.
- **Everything else is server-only.** If a variable does not start with `NEXT_PUBLIC_`, Next.js will not bundle it into client code. Do not circumvent this.
- **Never commit `.env` files.** Use `.env.local` for development and your hosting platform's environment variable management (e.g., Vercel Environment Variables) for production.
- **Rotate keys immediately** if any server-only variable is accidentally exposed in client code, commit history, or logs.

---

## 9. Supabase Realtime — Subscription Rules

Supabase Realtime enables live updates via PostgreSQL change events. VERIS uses it selectively — not every table needs real-time subscriptions.

### When to Use Realtime

| Use Realtime For | Don't Use Realtime For |
|-----------------|----------------------|
| Attendance check-in feed (live display during an event) | Member directory (changes infrequently) |
| Payment verification queue (officers see new submissions live) | Organization settings (changed rarely) |
| Student portal balance updates (after payment verification) | Historical reports (static data) |

### Realtime + RLS

Supabase Realtime respects RLS policies. A subscriber only receives change events for rows they have `SELECT` access to. This means:

- An officer subscribed to `attendance_records` changes will only see events for their org.
- A student subscribed to `payments` changes will only see their own payments.

No additional filtering logic is needed in the subscription setup — RLS handles it.

### Enabling Realtime on a Table

Realtime must be explicitly enabled per table in the Supabase dashboard or via SQL:

```sql
-- Enable realtime for specific tables
ALTER PUBLICATION supabase_realtime ADD TABLE attendance_records;
ALTER PUBLICATION supabase_realtime ADD TABLE payments;
ALTER PUBLICATION supabase_realtime ADD TABLE fines;
```

> **Do not enable realtime on all tables.** Only add tables that have active subscribers. Unnecessary realtime publications add overhead to every write.

### Subscription Pattern

The standard subscription pattern (within a React Query hook) is documented in [03-nextjs-guidelines.md](./03-nextjs-guidelines.md#5-react-query--detailed-patterns). The pattern is:

1. `useQuery` fetches the initial data.
2. `useEffect` sets up a Supabase Realtime channel subscription.
3. On receiving a change event, call `queryClient.invalidateQueries()` to trigger a refetch.
4. Cleanup the channel on unmount.

---

## 10. Summary of Non-Negotiables

| Principle | Violation | Correct Approach |
|-----------|-----------|-----------------|
| RLS on every org-scoped table | Creating a table without `ENABLE ROW LEVEL SECURITY` | Enable RLS and write explicit policies before any data is inserted |
| JWT-based org scoping | Passing `organization_id` from a form field or query param | Use `auth.organization_id()` in RLS and Server Actions |
| Server Actions for all writes | Calling `supabase.from("x").insert()` in a Client Component | Create a Server Action in the feature's `actions/` folder |
| Firebase for receipts only | Storing member profile photos in Firebase | Use Supabase or do not store binary files |
| No client-side Firebase SDK | Importing `firebase/storage` in a Client Component | Use Server Actions that call Firebase Admin SDK |
| RPC for complex logic | Writing a 50-line aggregation query inline in a Server Action | Move it to a PostgreSQL function, call via `.rpc()` |
| `SECURITY INVOKER` by default | Using `SECURITY DEFINER` without manual org-scoping | Only use DEFINER when required, and always validate `organization_id` internally |
| Admin client used sparingly | Using `createAdminClient()` for regular CRUD reads | Use `createServerClient()` — RLS handles scoping |
| Descriptive policy names | `policy_1`, `allow_all`, `temp_policy` | `"officers can read own org events"` |
| System Admin has no blanket PII access | Granting `SELECT` on `students` table to `system_admin` via RLS | Use audited `SECURITY DEFINER` RPC with mandatory `access_reason` |
| Org onboarding via invite only | Adding a public org self-registration form | System Admin generates invite URL; org admin registers via token |
| System Admin accounts are seeded | Creating a signup form for System Admin | Provision via migration script or Supabase Auth Admin API |

---

## 11. Related Documents

| Document | Contents |
|----------|----------|
| [01-system-overview.md](./01-system-overview.md) | Product context, tiers, tech stack, multi-tenancy model |
| [02-architecture-principles.md](./02-architecture-principles.md) | Feature-driven structure, Thick DB / Thin API, data flow rules |
| [03-nextjs-guidelines.md](./03-nextjs-guidelines.md) | RSC vs Client Components, Server Action patterns, Supabase client initialization |
| [05-database-schema.md](./05-database-schema.md) | SQL table definitions, complete RLS policies per table, trigger definitions |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | Build order: Basic → Plus → Premium |
