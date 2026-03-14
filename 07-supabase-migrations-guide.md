# 07 — Supabase Migrations Guide

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** March 14, 2026  
> **Related Docs:** [Architecture Principles](./02-architecture-principles.md) · [Supabase & Auth](./04-supabase-firebase-auth.md) · [Database Schema](./05-database-schema.md)

---

## Purpose

This document defines how VERIS manages all database changes — tables, RLS policies, RPC functions, and triggers — using the **Supabase CLI and SQL migration files**.

Every change to the database schema is a committed, version-controlled `.sql` file. No exceptions. The Supabase dashboard is read-only for production — it is never used to make direct schema changes.

Read [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) first. This document assumes you understand RLS, RPC conventions, and trigger patterns defined there. This guide covers **how to author and deploy** them — not what they do.

---

## 1. CLI Setup

### Install & Initialize

```bash
# Install the Supabase CLI globally
npm install -g supabase

# In the Next.js project root — generates the supabase/ folder
supabase init
```

This creates:

```
your-nextjs-project/
└── supabase/
    ├── config.toml          # Local dev configuration
    ├── seed.sql             # Local dev seed data (test orgs, officers, students)
    └── migrations/          # All schema changes live here
```

Commit the entire `supabase/` folder to version control. The `migrations/` folder is the source of truth for the database schema.

### Link to Your Remote Project

```bash
# Link local CLI to your Supabase project
supabase link --project-ref <your-project-ref>

# Your project ref is in: Supabase Dashboard → Project Settings → General
```

---

## 2. Migration File Conventions

### Naming

Every migration file is auto-timestamped by the CLI. Always use the CLI to create new files — never create them manually.

```bash
# Creates: supabase/migrations/20260315120000_<name>.sql
supabase migration new <name>
```

**Naming rules:**

| Rule | Example |
|------|---------|
| Use `snake_case` | `create_attendance_tables` |
| Be specific — state what changes | `add_academic_period_id_to_fines` |
| Use a verb prefix | `create_`, `add_`, `drop_`, `alter_`, `rename_` |
| One concern per file | Don't create tables AND define RLS in the same file |

### Filename Prefix Reference

| Prefix | What Goes In It |
|--------|----------------|
| `schema_` | `CREATE TABLE`, indexes, constraints |
| `rls_` | `ALTER TABLE ENABLE ROW LEVEL SECURITY` + all `CREATE POLICY` |
| `rpc_` | `CREATE OR REPLACE FUNCTION` for app-callable RPCs |
| `trigger_` | Trigger functions (`fn_*`) + `CREATE TRIGGER` attachments |
| `seed_` | One-time data inserts (e.g., `system_settings` row) |
| `add_` | Additive changes — new column, new index on existing table |
| `alter_` | Modifications to existing tables or functions |
| `drop_` | Removals |

**Examples of good migration names:**

```
20260101000000_create_core_schema.sql
20260101000001_create_rls_policies.sql
20260101000002_create_auth_helper_functions.sql
20260101000003_create_audit_log_trigger.sql
20260101000004_create_payment_status_sync_trigger.sql
20260101000005_create_absence_fine_trigger.sql
20260101000006_create_financial_rpc_functions.sql
20260101000007_create_clearance_rpc_functions.sql
```

### File Ownership Map

Each migration file owns one domain. This is the standard split for VERIS:

| File Pattern | Owns |
|-------------|------|
| `*_create_core_schema.sql` | `CREATE TABLE` statements for all tables in a domain |
| `*_create_rls_policies.sql` | `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` + all `CREATE POLICY` statements |
| `*_create_auth_helper_functions.sql` | `auth.organization_id()`, `auth.user_role()`, `auth.has_tier()`, `auth.is_system_admin()` |
| `*_create_*_trigger.sql` | One trigger function + its `CREATE TRIGGER` attachment |
| `*_create_*_rpc_functions.sql` | One or more related RPC functions in the same domain |
| `*_add_*.sql` | Additive schema changes (new column, new index) |
| `*_alter_*.sql` | Modifications to existing tables or functions |

> **One concern per file.** A migration that creates tables AND defines RLS AND adds triggers is hard to debug, hard to roll back, and hard to review. Split them.

---

## 3. Local Dev Workflow

### First-Time Setup

```bash
# Start the local Supabase stack (requires Docker)
supabase start

# Output includes your local credentials:
# API URL: http://localhost:54321
# DB URL:  postgresql://postgres:postgres@localhost:54322/postgres
# Studio:  http://localhost:54323
```

Update your `.env.local` with the local credentials output by `supabase start`:

```bash
# .env.local (local dev only — never commit)
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=<local-anon-key>
SUPABASE_SERVICE_ROLE_KEY=<local-service-role-key>
```

### Core Commands

```bash
# Apply all pending migrations to the local DB
supabase migration up

# Wipe local DB and re-run ALL migrations from scratch + seed.sql
# Use this after pulling changes from teammates
supabase db reset

# Create a new empty migration file
supabase migration new <name>

# Check which migrations are applied vs pending
supabase migration list

# Stop the local Supabase stack
supabase stop
```

### Typical Dev Loop

```
1. supabase migration new <name>
2. Write SQL in the generated file
3. supabase migration up          ← applies just the new file locally
4. Test in the app / Supabase Studio (http://localhost:54323)
5. Iterate — if you need to fix the SQL:
   supabase db reset               ← wipes + replays all migrations cleanly
6. git add supabase/migrations/<file>.sql
7. Commit and push
```

> **Never edit a migration file after it has been applied to the remote database.** If you need to fix something, create a new migration that corrects it. Editing applied migrations breaks the migration history for everyone on the team.

---

## 4. Production Push

```bash
# Push all pending migrations to the remote (staging/production) project
supabase db push

# Pull schema changes from remote into a new local migration file
# Use this if a change was made via the dashboard (avoid this pattern)
supabase db pull
```

### Push Rules

| Rule | Detail |
|------|--------|
| **Never push directly from a feature branch.** | Only push from `main` (or your designated release branch) after PR review. |
| **Never edit via the Supabase dashboard in production.** | Dashboard edits are not tracked in version control. They will be overwritten or conflict on the next `db push`. |
| **If a dashboard change was made, pull it immediately.** | Run `supabase db pull` to capture it as a migration file, commit it, and never do it again. |
| **Verify locally before pushing.** | Run `supabase db reset` locally after writing a migration to confirm it applies cleanly from scratch. |

---

## 5. Writing Tables in Migration Files

Standard `CREATE TABLE` inside a migration. All VERIS conventions apply: `uuid` PKs, `timestamptz` timestamps, `numeric(10,2)` for money.

```sql
-- supabase/migrations/20260101000000_create_core_schema.sql

CREATE TABLE organizations (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name                TEXT NOT NULL,
  slug                TEXT UNIQUE NOT NULL,
  tier                VARCHAR NOT NULL CHECK (tier IN ('basic', 'plus', 'premium')),
  status              VARCHAR NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'suspended', 'inactive')),
  student_count       INT4 NOT NULL DEFAULT 0,
  subscription_start  TIMESTAMPTZ,
  subscription_end    TIMESTAMPTZ,
  invite_id           UUID,
  metadata            JSONB,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE academic_periods (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  academic_year    TEXT NOT NULL,
  semester         TEXT NOT NULL,
  start_date       DATE NOT NULL,
  end_date         DATE NOT NULL,
  is_current       BOOL NOT NULL DEFAULT FALSE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (organization_id, academic_year, semester)
);

-- Partial unique index: only one current period per org
CREATE UNIQUE INDEX idx_one_current_period_per_org
  ON academic_periods (organization_id)
  WHERE is_current = TRUE;
```

---

## 6. Writing RLS Policies in Migration Files

RLS is always a separate migration file from table creation. Pattern: enable RLS first, then write policies.

```sql
-- supabase/migrations/20260101000001_create_rls_policies.sql

-- ── Auth helper functions (defined first — policies depend on them) ──────────

CREATE OR REPLACE FUNCTION auth.organization_id()
RETURNS uuid LANGUAGE sql STABLE AS $$
  SELECT (auth.jwt() -> 'app_metadata' ->> 'organization_id')::uuid;
$$;

CREATE OR REPLACE FUNCTION auth.user_role()
RETURNS text LANGUAGE sql STABLE AS $$
  SELECT auth.jwt() -> 'app_metadata' ->> 'role';
$$;

CREATE OR REPLACE FUNCTION auth.has_tier(required_tier text)
RETURNS boolean LANGUAGE sql STABLE AS $$
  SELECT EXISTS (
    SELECT 1 FROM organizations
    WHERE id = auth.organization_id()
      AND tier = required_tier
         OR (required_tier = 'basic')                   -- basic = all tiers
         OR (required_tier = 'plus'  AND tier IN ('plus', 'premium'))
         OR (required_tier = 'premium' AND tier = 'premium')
  );
$$;

CREATE OR REPLACE FUNCTION auth.is_system_admin()
RETURNS boolean LANGUAGE sql STABLE AS $$
  SELECT auth.jwt() -> 'app_metadata' ->> 'role' = 'system_admin';
$$;

-- ── organizations ────────────────────────────────────────────────────────────

ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "officers can read own org"
  ON organizations FOR SELECT
  USING (id = auth.organization_id());

CREATE POLICY "system admin can read all organizations"
  ON organizations FOR SELECT
  USING (auth.is_system_admin());

CREATE POLICY "system admin can update organizations"
  ON organizations FOR UPDATE
  USING (auth.is_system_admin())
  WITH CHECK (auth.is_system_admin());

-- ── fines (Plus+ tier gate) ──────────────────────────────────────────────────

ALTER TABLE fines ENABLE ROW LEVEL SECURITY;

CREATE POLICY "officers can read fines plus tier"
  ON fines FOR SELECT
  USING (
    organization_id = auth.organization_id()
    AND auth.has_tier('plus')
    AND deleted_at IS NULL
  );

CREATE POLICY "admins and managers can write fines"
  ON fines FOR INSERT
  WITH CHECK (
    organization_id = auth.organization_id()
    AND auth.user_role() IN ('admin', 'manager')
    AND auth.has_tier('plus')
  );

CREATE POLICY "students can read own fines"
  ON fines FOR SELECT
  USING (
    student_id = auth.uid()
    AND auth.user_role() = 'student'
    AND deleted_at IS NULL
  );

-- ── fine_appeals (Premium tier gate) ─────────────────────────────────────────

ALTER TABLE fine_appeals ENABLE ROW LEVEL SECURITY;

CREATE POLICY "students can submit own appeals premium"
  ON fine_appeals FOR INSERT
  WITH CHECK (
    student_id = auth.uid()
    AND auth.user_role() = 'student'
    AND auth.has_tier('premium')
  );

CREATE POLICY "officers can read appeals premium"
  ON fine_appeals FOR SELECT
  USING (
    organization_id = auth.organization_id()
    AND auth.has_tier('premium')
  );
```

---

## 7. Writing Triggers in Migration Files

Every trigger is its own migration file: the function first, then the `CREATE TRIGGER` attachment.

### Pattern

```sql
-- 1. Create the trigger function
CREATE OR REPLACE FUNCTION fn_<description>()
RETURNS trigger
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER          -- or INVOKER — see Section 8
SET search_path = public  -- required for SECURITY DEFINER
AS $$
BEGIN
  -- logic
  RETURN NEW; -- or OLD for DELETE triggers
END;
$$;

-- 2. Attach the trigger to the table
CREATE TRIGGER trg_<table>_<timing>_<event>
  AFTER INSERT OR UPDATE ON <table>
  FOR EACH ROW EXECUTE FUNCTION fn_<description>();
```

### Real Example: Audit Log Trigger

```sql
-- supabase/migrations/20260101000003_create_audit_log_trigger.sql

CREATE OR REPLACE FUNCTION fn_audit_log()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
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
    TG_OP,
    CASE WHEN TG_OP != 'INSERT' THEN row_to_json(OLD) ELSE NULL END,
    CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW) ELSE NULL END,
    auth.uid(),
    COALESCE(
      CASE WHEN TG_OP != 'DELETE' THEN NEW.organization_id ELSE NULL END,
      OLD.organization_id
    ),
    NOW()
  );
  RETURN COALESCE(NEW, OLD);
END;
$$;

-- Attach to all sensitive tables
CREATE TRIGGER trg_payments_after_audit
  AFTER INSERT OR UPDATE OR DELETE ON payments
  FOR EACH ROW EXECUTE FUNCTION fn_audit_log();

CREATE TRIGGER trg_fines_after_audit
  AFTER INSERT OR UPDATE OR DELETE ON fines
  FOR EACH ROW EXECUTE FUNCTION fn_audit_log();

CREATE TRIGGER trg_students_after_audit
  AFTER INSERT OR UPDATE OR DELETE ON students
  FOR EACH ROW EXECUTE FUNCTION fn_audit_log();
```

### Real Example: Payment Status Sync Trigger

Fires after any `payment_allocations` change and recalculates the `status` of the linked `fee_assignment` or `fine`.

```sql
-- supabase/migrations/20260101000004_create_payment_status_sync_trigger.sql

CREATE OR REPLACE FUNCTION fn_sync_obligation_status()
RETURNS trigger
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_target_fee_id   uuid;
  v_target_fine_id  uuid;
  v_total_paid      numeric;
  v_obligation_amt  numeric;
  v_current_status  varchar;
  v_new_status      varchar;
BEGIN
  -- Determine which obligation was affected
  v_target_fee_id  := COALESCE(NEW.fee_assignment_id, OLD.fee_assignment_id);
  v_target_fine_id := COALESCE(NEW.fine_id, OLD.fine_id);

  -- ── Fee Assignment ──────────────────────────────────────────────────────────
  IF v_target_fee_id IS NOT NULL THEN
    SELECT COALESCE(SUM(pa.amount_allocated), 0)
    INTO v_total_paid
    FROM payment_allocations pa
    JOIN payments p ON pa.payment_id = p.id
    WHERE pa.fee_assignment_id = v_target_fee_id
      AND p.status = 'verified';

    SELECT amount, status INTO v_obligation_amt, v_current_status
    FROM fee_assignments WHERE id = v_target_fee_id;

    -- Waivers take precedence — do not overwrite
    IF v_current_status IN ('waived') THEN
      RETURN COALESCE(NEW, OLD);
    END IF;

    IF v_total_paid = 0 THEN
      v_new_status := 'pending';
    ELSIF v_total_paid < v_obligation_amt THEN
      v_new_status := 'partially_paid';
    ELSE
      v_new_status := 'paid';
    END IF;

    UPDATE fee_assignments
    SET status = v_new_status, updated_at = NOW()
    WHERE id = v_target_fee_id AND status != v_new_status;
  END IF;

  -- ── Fine ────────────────────────────────────────────────────────────────────
  IF v_target_fine_id IS NOT NULL THEN
    SELECT COALESCE(SUM(pa.amount_allocated), 0)
    INTO v_total_paid
    FROM payment_allocations pa
    JOIN payments p ON pa.payment_id = p.id
    WHERE pa.fine_id = v_target_fine_id
      AND p.status = 'verified';

    SELECT amount, status INTO v_obligation_amt, v_current_status
    FROM fines WHERE id = v_target_fine_id;

    -- Waivers and active appeals take precedence
    IF v_current_status IN ('waived', 'appealed') THEN
      RETURN COALESCE(NEW, OLD);
    END IF;

    IF v_total_paid = 0 THEN
      v_new_status := 'pending';
    ELSIF v_total_paid < v_obligation_amt THEN
      v_new_status := 'partially_paid';
    ELSE
      v_new_status := 'paid';
    END IF;

    UPDATE fines
    SET status = v_new_status, updated_at = NOW()
    WHERE id = v_target_fine_id AND status != v_new_status;
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_payment_allocations_after_sync
  AFTER INSERT OR UPDATE OR DELETE ON payment_allocations
  FOR EACH ROW EXECUTE FUNCTION fn_sync_obligation_status();
```

### Real Example: Absence Fine Generation Trigger (Premium)

Fires when an event's `status` is updated to `'archived'`. Generates fines for all active members with no attendance record.

```sql
-- supabase/migrations/20260101000005_create_absence_fine_trigger.sql

CREATE OR REPLACE FUNCTION fn_generate_absence_fines()
RETURNS trigger
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_tier      text;
  v_fine_type_id  uuid;
  v_fine_amount   numeric;
BEGIN
  -- Only fire when transitioning TO 'archived'
  IF NEW.status != 'archived' OR OLD.status = 'archived' THEN
    RETURN NEW;
  END IF;

  -- Automated fines are Premium-only
  SELECT tier INTO v_org_tier
  FROM organizations WHERE id = NEW.organization_id;

  IF v_org_tier != 'premium' THEN
    RETURN NEW;
  END IF;

  -- Look up the active absence fine type configured for this event's type
  SELECT ft.id, ft.default_amount
  INTO v_fine_type_id, v_fine_amount
  FROM fine_types ft
  WHERE ft.organization_id = NEW.organization_id
    AND ft.event_type_id = NEW.event_type_id
    AND ft.trigger_source = 'absence'
    AND ft.is_active = TRUE
  LIMIT 1;

  -- No fine type configured — skip silently
  IF v_fine_type_id IS NULL THEN
    RETURN NEW;
  END IF;

  -- Insert a fine for every active member who has no attendance record
  INSERT INTO fines (
    student_id,
    organization_id,
    fine_type_id,
    event_id,
    academic_period_id,
    amount,
    status,
    created_at,
    updated_at
  )
  SELECT
    som.student_id,
    NEW.organization_id,
    v_fine_type_id,
    NEW.id,
    NEW.academic_period_id,
    v_fine_amount,
    'pending',
    NOW(),
    NOW()
  FROM student_org_memberships som
  WHERE som.organization_id = NEW.organization_id
    AND som.status = 'active'
    AND som.deleted_at IS NULL
    AND NOT EXISTS (
      SELECT 1 FROM attendance_records ar
      WHERE ar.event_id = NEW.id
        AND ar.student_id = som.student_id
    )
  ON CONFLICT DO NOTHING;  -- Idempotent: safe to re-run

  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_events_after_update_absence_fines
  AFTER UPDATE OF status ON events
  FOR EACH ROW EXECUTE FUNCTION fn_generate_absence_fines();
```

---

## 8. Writing RPC Functions in Migration Files

RPC functions are authored in migration files exactly like triggers. The key decision per function is `SECURITY INVOKER` vs `SECURITY DEFINER`.

### INVOKER vs DEFINER — Decision Rule

| Use `SECURITY INVOKER` | Use `SECURITY DEFINER` |
|------------------------|------------------------|
| Function reads/writes data the caller already has RLS access to | Function needs to read data the caller cannot access (e.g., org settings during a student action) |
| Aggregations over the caller's own org data | Writing to `audit_log` (users have no direct INSERT on this table) |
| Most financial summary functions | `sysadmin_lookup_student` — crosses org boundaries |
| Default choice when unsure | When `INVOKER` would fail due to RLS blocking a required read |

> **Rule:** Default to `SECURITY INVOKER`. Justify every `SECURITY DEFINER` usage in a SQL comment above the function.

### Real Example: Student Balance RPC

```sql
-- supabase/migrations/20260101000006_create_financial_rpc_functions.sql

-- SECURITY INVOKER: student can only compute their own balance — RLS enforces this.
CREATE OR REPLACE FUNCTION get_student_balance(p_student_id uuid)
RETURNS json
LANGUAGE plpgsql
STABLE
SECURITY INVOKER
AS $$
DECLARE
  v_total_fees   numeric := 0;
  v_total_fines  numeric := 0;
  v_total_paid   numeric := 0;
BEGIN
  SELECT COALESCE(SUM(amount), 0) INTO v_total_fees
  FROM fee_assignments
  WHERE student_id = p_student_id
    AND deleted_at IS NULL;

  SELECT COALESCE(SUM(amount), 0) INTO v_total_fines
  FROM fines
  WHERE student_id = p_student_id
    AND status NOT IN ('waived', 'appealed')
    AND deleted_at IS NULL;

  SELECT COALESCE(SUM(amount), 0) INTO v_total_paid
  FROM payments
  WHERE student_id = p_student_id
    AND status = 'verified'
    AND deleted_at IS NULL;

  RETURN json_build_object(
    'total_fees',   v_total_fees,
    'total_fines',  v_total_fines,
    'total_paid',   v_total_paid,
    'balance',      (v_total_fees + v_total_fines) - v_total_paid
  );
END;
$$;

GRANT EXECUTE ON FUNCTION get_student_balance(uuid) TO authenticated;
```

### Real Example: Org Financial Summary RPC

```sql
-- SECURITY INVOKER: officer's RLS already scopes to their org.
-- Staff role check is a defense-in-depth guard — primary enforcement is via RLS.
CREATE OR REPLACE FUNCTION get_org_financial_summary()
RETURNS json
LANGUAGE plpgsql
STABLE
SECURITY INVOKER
AS $$
DECLARE
  v_org    uuid;
  v_result json;
BEGIN
  v_org := auth.organization_id();

  IF auth.user_role() = 'staff' THEN
    RAISE EXCEPTION 'Insufficient permissions';
  END IF;

  SELECT json_build_object(
    'total_collected',   COALESCE(SUM(amount) FILTER (WHERE status = 'verified'), 0),
    'total_pending',     COALESCE(SUM(amount) FILTER (WHERE status = 'pending'),  0),
    'total_fines_unpaid',(
      SELECT COALESCE(SUM(amount), 0) FROM fines
      WHERE organization_id = v_org
        AND status NOT IN ('paid', 'waived')
        AND deleted_at IS NULL
    )
  ) INTO v_result
  FROM payments
  WHERE organization_id = v_org
    AND deleted_at IS NULL;

  RETURN v_result;
END;
$$;

GRANT EXECUTE ON FUNCTION get_org_financial_summary() TO authenticated;
```

### Real Example: System Admin Audited Student Lookup

```sql
-- supabase/migrations/20260101000007_create_sysadmin_rpc_functions.sql

-- SECURITY DEFINER: System Admin must bypass RLS on `students` (no blanket SELECT policy).
-- Mandatory audit trail before data is returned.
CREATE OR REPLACE FUNCTION sysadmin_lookup_student(
  p_student_id    uuid,
  p_access_reason text
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
  IF NOT auth.is_system_admin() THEN
    RAISE EXCEPTION 'Access denied: system_admin role required';
  END IF;

  IF p_access_reason IS NULL OR TRIM(p_access_reason) = '' THEN
    RAISE EXCEPTION 'Access reason is required for student data lookup';
  END IF;

  -- Log BEFORE returning data
  INSERT INTO audit_log (
    table_name, record_id, action, old_data, new_data,
    performed_by, organization_id, created_at
  ) VALUES (
    'students', p_student_id, 'SYSADMIN_READ',
    NULL, json_build_object('access_reason', p_access_reason),
    auth.uid(), NULL, NOW()
  );

  SELECT * INTO v_student FROM students WHERE id = p_student_id;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Student not found';
  END IF;

  RETURN row_to_json(v_student);
END;
$$;

GRANT EXECUTE ON FUNCTION sysadmin_lookup_student(uuid, text) TO authenticated;
```

---

## 9. Calling Migrations from the App Layer

Migration files define the database objects. The app invokes them like this:

### Calling an RPC from a React Server Component

```typescript
// src/features/portal/components/StudentBalanceSummary.tsx

import { createServerClient } from '@/lib/supabase/server'

export default async function StudentBalanceSummary({ studentId }: { studentId: string }) {
  const supabase = await createServerClient()

  const { data, error } = await supabase.rpc('get_student_balance', {
    p_student_id: studentId,
  })

  if (error) throw error

  return (
    <div>
      <p>Balance: ₱{data.balance.toFixed(2)}</p>
    </div>
  )
}
```

### Calling an RPC from a Server Action

```typescript
// src/features/financials/actions/recomputeClearance.ts
'use server'

import { createServerClient } from '@/lib/supabase/server'
import { revalidatePath } from 'next/cache'

export async function recomputeClearanceStatus(studentId: string, periodId: string) {
  const supabase = await createServerClient()

  const { error } = await supabase.rpc('recompute_clearance_status', {
    p_student_id: studentId,
    p_period_id:  periodId,
  })

  if (error) return { error: 'Failed to recompute clearance.' }

  revalidatePath('/dashboard/clearance')
  return { success: true, data: undefined }
}
```

> **Triggers are never called directly.** They fire automatically from the database on the relevant `INSERT`, `UPDATE`, or `DELETE`. The app does not invoke trigger functions — it just performs the operation that fires them.

---

## 10. seed.sql — Local Dev Data

`supabase/seed.sql` runs automatically after every `supabase db reset`. Use it to populate local dev data so developers can work without manual setup.

```sql
-- supabase/seed.sql

-- Two test organizations (Basic and Premium) for tenant isolation testing
INSERT INTO organizations (id, name, slug, tier, status, subscription_start, subscription_end)
VALUES
  ('00000000-0000-0000-0000-000000000001', 'Test Org Basic',   'test-org-basic',   'basic',   'active', NOW(), NOW() + INTERVAL '1 year'),
  ('00000000-0000-0000-0000-000000000002', 'Test Org Premium', 'test-org-premium', 'premium', 'active', NOW(), NOW() + INTERVAL '1 year');

-- Seed officers via Supabase Auth Admin API in a separate setup script
-- (auth.users rows cannot be inserted directly via seed.sql in most setups)
-- See: scripts/seed-auth-users.ts
```

> **Never commit real student data or production credentials to `seed.sql`.** Test data only.

---

## 11. Non-Negotiables

| Rule | Detail |
|------|--------|
| **All schema changes via migration files.** | No direct dashboard edits in production. No exceptions. |
| **Never edit an applied migration.** | Create a new migration to fix it. Editing applied files breaks migration history. |
| **One concern per migration file.** | Tables in one file, RLS in another, triggers in another. |
| **CLI-generated filenames only.** | `supabase migration new <name>` — never create migration files manually. |
| **`supabase db reset` before every PR.** | Verify all migrations apply cleanly from scratch locally before pushing. |
| **`SECURITY DEFINER` requires a comment.** | Every `DEFINER` function must have a SQL comment explaining why `INVOKER` is insufficient. |
| **`GRANT EXECUTE` on every RPC.** | All public-facing RPC functions must have `GRANT EXECUTE ... TO authenticated`. Without it, the function exists but cannot be called. |
| **Idempotent triggers.** | Use `ON CONFLICT DO NOTHING` or `IF NOT EXISTS` guards in trigger functions. Triggers can re-fire in edge cases. |
| **`SET search_path = public` on all `DEFINER` functions.** | Prevents search_path hijacking attacks. |

---

## 12. Related Documents

| Document | Contents |
|----------|----------|
| [02-architecture-principles.md](./02-architecture-principles.md) | Thick DB / Thin API — why logic lives in the database |
| [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) | RLS policy patterns, RPC conventions, trigger conventions |
| [05-database-schema.md](./05-database-schema.md) | Table definitions, column types, relationships |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | Build order — which migrations to write first |