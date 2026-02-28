# 05 — Database Schema

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** 02/26/2026  
> **Related Docs:** [System Overview](./01-system-overview.md) · [Architecture Principles](./02-architecture-principles.md) · [Supabase & Auth](./04-supabase-firebase-auth.md)

---

## Purpose

This document defines every table in the VERIS database. Each table is presented with its columns, types, and developer notes. This file is **schema only** — RLS policies, triggers, stored functions, validation logic, and RPC definitions are documented in their respective files ([04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) for RLS/RPC/triggers).

Read [01-system-overview.md](./01-system-overview.md) and [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) first. This document assumes you understand the tier model, RBAC roles, and the Thick DB / Thin API principle.

---

## 1. Feature Coverage

The schema supports the following feature domains:

- **Platform Configuration** — institution identity, campus settings, platform-wide defaults
- **Organization Management** — multi-tenant orgs, tier-based subscriptions, System Admin onboarding invites
- **Organization Settings** — per-org configuration, custom branding/themes (Premium)
- **Officer Authentication & RBAC** — email/password auth, role assignment (Admin / Manager / Staff)
- **Shared Student Data Model** — campus-wide student identity, cross-org data sharing with consent
- **Student Self-Registration** — org-generated public URLs for student onboarding via Google OAuth (all tiers), with optional officer approval
- **Bulk Student Import** — CSV import with mandatory formal document backing and review workflow
- **Academic Period Management** — org-defined academic years and semesters for filtering and scoping
- **Event Type Configuration** — org-defined event categories (major, minor, etc.) linked to fine rules
- **Event & Attendance Tracking** — event lifecycle, time-in/out recording, QR-based check-in (student QR + event QR), scoped by event type and academic period
- **Fee Configuration & Assignment** — fee types, per-student fee assignment, scoped to academic periods (Plus+)
- **Fine Management** — manual and automated fines linked to event types, scoped to academic periods (Plus+/Premium)
- **Waiver Management** — unified waiver system for fines and fees with approval workflow and audit trail (Plus+)
- **Payment Processing** — GCash and in-person payments, receipt storage (Firebase), officer verification, official receipt tracking, precise payment-to-fee/fine allocation for accumulated balances (Plus+)
- **Fine Appeals** — student-initiated dispute workflow (Premium)
- **Clearance Management** — per-period clearance status, manual override (Plus+)
- **Subscription Billing** — tracks physical cash payments from organizations to the VERIS platform, with confirmation and receipt tracking
- **Transactional Notifications** — email notification queue for system-triggered events (payment verified, fine assigned, clearance status, appeal resolved)
- **Audit Logging** — immutable, platform-wide audit trail for sensitive operations
- **Soft Delete** — `deleted_at` / `deleted_by` columns on critical tables for non-destructive removal with full audit trail

---

## 2. Shared Student Data Model

VERIS is deployed within a single campus. All organizations on the platform share the same student ID numbering system. Instead of duplicating student records per organization, VERIS uses a **campus-wide `students` table** linked to organizations through a **`student_org_memberships` join table**.

### How It Works

```
┌──────────────────┐        ┌──────────────────────────┐        ┌──────────────────┐
│    students       │─────▶│  student_org_memberships  │◀──────│  organizations   │
│  (campus-wide)    │  1:N  │  (consent + status)       │  N:1  │  (tenant)        │
└──────────────────┘        └──────────────────────────┘        └──────────────────┘
```

1. **First import:** Org A bulk-imports students. Each new student gets a row in `students` and a corresponding `student_org_memberships` row with status `active` (consent is covered by the formal import document).
2. **Cross-org sharing:** Org B imports a student that already exists (matched by `id_number`). Instead of creating a duplicate, a new membership row is created with status `pending_consent`.
3. **Consent resolution:** If the student has a portal account, they approve from their dashboard. If not, the organization obtains consent out-of-band (e.g., physical consent form per RA 10173) and an officer updates the status.
4. **Self-registration (public link):** An officer generates a public registration link ([`org_registration_links`](#49-org_registration_links)). Students visit the link, authenticate via **Google OAuth** (all tiers), and either create a new profile or claim an existing one by entering their `id_number`. A `student_org_memberships` row is created with status `pending_approval` (if the link requires approval) or `active` (if auto-approved). Consent is implicit — the student is actively choosing to join.
5. **Portal access:** A student can only access an organization's portal dashboard if that organization is on the **Premium** tier and the membership `is_verified = true`. **Authentication does not imply dashboard access** — Basic/Plus students who register via a public link authenticate only for identity verification and see a confirmation page, not a dashboard.

### Key Rules

| Rule | Detail |
|------|--------|
| **One student record per campus ID.** | The `students.id_number` column is unique across the entire system. No duplicate student profiles. |
| **Org access is via membership only.** | Officers can only query students who have an `active` membership with their organization. RLS enforces this via a join through `student_org_memberships`. |
| **Student profile is authoritative.** | The `students` table holds the canonical data. If a student's course or level changes, it updates once and propagates to all linked organizations. Changes are audit-logged. |
| **Consent is per-organization.** | Each `student_org_memberships` row tracks consent independently. Consenting to Org A does not grant Org B access. |
| **`auth_user_id` is optional.** | Students who authenticate via Google OAuth — either through the Premium portal or through a public registration link (any tier) — have a linked auth account. Import-only students have no login capability. Authentication does **not** grant dashboard access; that is gated by tier (Premium only) and membership verification (`is_verified = true`). |
| **Campus identity is platform-level.** | The [`system_settings`](#41-system_settings) table stores the institution name and campus identifier. All students in this system belong to the same campus — VERIS is deployed per-campus, which is why `id_number` uniqueness works as the cross-org matching key. |

---

## 3. Schema Overview

All tables use `uuid` primary keys generated via `gen_random_uuid()`. Timestamps use `timestamptz` and default to `now()`. All monetary/amount columns use `numeric(10,2)` to enforce two decimal places and prevent rounding errors.

### Core / Platform

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 1 | [`system_settings`](#41-system_settings) | Platform | Single-row platform config. Institution identity, global defaults. |
| 2 | [`organizations`](#42-organizations) | Platform | Tenant root. Org identity, tier, and subscription info. |
| 3 | [`organization_settings`](#43-organization_settings) | Organization | Per-org config. Branding, active period, operational settings. |
| 4 | [`org_invites`](#44-org_invites) | Platform | System Admin–generated invite tokens for org onboarding. |
| 5 | [`officers`](#45-officers) | Organization | Officer accounts linked to a specific org with a role. |

### Student Data

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 6 | [`students`](#46-students) | Campus-wide | Canonical student profiles. Not org-scoped. |
| 7 | [`student_org_memberships`](#47-student_org_memberships) | Organization | Links students to organizations with consent tracking. |
| 8 | [`bulk_import_requests`](#48-bulk_import_requests) | Organization | Tracks bulk CSV imports with formal document review. |
| 9 | [`org_registration_links`](#49-org_registration_links) | Organization | Public registration URLs for student self-onboarding via Google OAuth. |

### Configuration

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 10 | [`academic_periods`](#410-academic_periods) | Organization | Academic year + semester definitions for scoping and filtering. |
| 11 | [`event_types`](#411-event_types) | Organization | Org-defined event categories (major, minor, etc.). |

### Attendance

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 12 | [`events`](#412-events) | Organization | Attendance events with lifecycle, event type, QR token, and academic period. |
| 13 | [`attendance_records`](#413-attendance_records) | Organization | Individual check-in/out records per student per event. |

### Financials (Plus+)

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 14 | [`fee_types`](#414-fee_types) | Organization | Fee category definitions (semester, event, custom). |
| 15 | [`fee_assignments`](#415-fee_assignments) | Organization | Per-student fee instances, scoped to academic period. |
| 16 | [`fine_types`](#416-fine_types) | Organization | Fine category definitions linked to event types for automation. |
| 17 | [`fines`](#417-fines) | Organization | Individual fines, scoped to academic period. |
| 18 | [`payments`](#418-payments) | Organization | GCash and in-person payments with receipt and OR tracking. |
| 19 | [`waivers`](#419-waivers) | Organization | Unified waiver records for fines and fees with approval workflow. |
| 20 | [`payment_allocations`](#420-payment_allocations) | Organization | Maps each payment to specific fees/fines it covers (partial or full). |

### Clearance & Appeals (Plus+ / Premium)

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 21 | [`fine_appeals`](#421-fine_appeals) | Organization | Student-submitted fine disputes (Premium only). |
| 22 | [`clearances`](#422-clearances) | Organization | Per-period clearance status per student. |

### Subscription Billing

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 23 | [`subscription_payments`](#423-subscription_payments) | Platform | Tracks physical cash payments from orgs for the VERIS platform subscription. |

### Notifications

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 24 | [`notification_queue`](#424-notification_queue) | Organization | Transactional email notification queue for system-triggered events. |

### Audit

| # | Table | Scope | Description |
|---|-------|-------|-------------|
| 25 | [`audit_log`](#425-audit_log) | Platform | Immutable log of all sensitive operations across the system. |

---

## 4. Table Definitions

### 4.1 `system_settings`

Single-row platform configuration. Stores institution identity and global platform defaults. Only one row should ever exist — seeded during initial deployment.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. Only one row. |
| `institution_name` | `text` | Full university/campus name (e.g., "Visayas State University"). Displayed in platform UI and reports. |
| `institution_short_name` | `text` | Abbreviation (e.g., "VSU"). Used in compact UI elements. |
| `campus` | `text` | Nullable. Specific campus within the institution (e.g., "Main Campus", "Tolosa Campus"). Null if the institution has only one campus. |
| `platform_name` | `text` | Default `'VERIS'`. Display name for the platform instance. |
| `settings` | `jsonb` | Nullable. Flexible key-value store for future platform-wide settings (e.g., default timezone, support contact). |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |
| `updated_by` | `uuid` | FK → `auth.users`. Nullable. System Admin who last modified settings. |

---

### 4.2 `organizations`

Tenant root table. Every org-scoped table references this via `organization_id`.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `name` | `text` | Organization display name. |
| `description` | `text` | Nullable. Organization details |
| `slug` | `text` | Unique, URL-safe identifier. Used in routing and storage paths. |
| `tier` | `varchar` | `'basic'` · `'plus'` · `'premium'`. Checked by RLS for feature gating. |
| `status` | `varchar` | `'active'` · `'suspended'` · `'inactive'`. System Admin controls this. |
| `student_count` | `int4` | Current active member count. **Maintained by a database trigger** on `student_org_memberships` — incremented/decremented on INSERT, UPDATE (status changes to/from `'active'`), soft DELETE, and hard DELETE. Not updated by application code directly. Ensures race-free accuracy for billing calculations. |
| `subscription_start` | `timestamptz` | Start of current annual billing cycle. |
| `subscription_end` | `timestamptz` | End of current annual billing cycle. |
| `invite_id` | `uuid` | FK → `org_invites`. Nullable. The invite token that led to this organization's creation. Links back to the onboarding invite for audit traceability. Null for orgs created directly by System Admins without an invite. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store for future extensibility (e.g., contact info, notes from System Admin). |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **Note:** Branding/theme configuration has been moved to [`organization_settings`](#43-organization_settings) to keep this table focused on identity and subscription.

> **Invite traceability:** The `invite_id` column links this org back to the [`org_invites`](#44-org_invites) token that triggered its creation, closing the audit loop between onboarding invites and resulting tenants.

---

### 4.3 `organization_settings`

Per-organization configuration. One-to-one with `organizations`. Stores operational settings and branding — separated from `organizations` to keep the tenant root table lightweight.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. Unique. One settings row per org. Created automatically when an org is onboarded. |
| `branding` | `jsonb` | Nullable. Premium custom theme config (colors, fonts, logo URL). Null for Basic/Plus orgs. See [01-system-overview.md](./01-system-overview.md) for tier details. |
| `default_academic_period_id` | `uuid` | FK → `academic_periods`. Nullable. The currently active academic period for this org. Used as default filter in UI and for new record creation. |
| `settings` | `jsonb` | Nullable. Flexible key-value store for future org-specific settings (e.g., attendance cutoff time, receipt numbering prefix). |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

---

### 4.4 `org_invites`

System Admin–generated tokens for organization onboarding. Not org-scoped — managed at the platform level.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `token` | `text` | Unique, signed invite token. Included as query param in the invite URL. |
| `organization_name` | `text` | Pre-configured by System Admin during invite creation. |
| `contact_email` | `text` | Target recipient's email address. |
| `assigned_tier` | `varchar` | `'basic'` · `'plus'` · `'premium'`. Set by System Admin during contract setup. |
| `status` | `varchar` | `'pending'` · `'consumed'` · `'expired'` · `'revoked'`. |
| `expires_at` | `timestamptz` | Token expiry. Enforced during registration validation. |
| `created_by` | `uuid` | FK → `auth.users`. The System Admin who generated the invite. |
| `consumed_at` | `timestamptz` | Nullable. Set when the Org Admin completes registration. |
| `created_at` | `timestamptz` | Auto-set on insert. |

---

### 4.5 `officers`

Officer accounts linked to a specific organization. One auth user maps to one officer row.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `auth_user_id` | `uuid` | FK → `auth.users`. Unique. Links to Supabase Auth. |
| `organization_id` | `uuid` | FK → `organizations`. Tenant isolation key. |
| `role` | `varchar` | `'admin'` · `'manager'` · `'staff'`. Basic/Plus orgs default to `'admin'` (single-account model). Multi-role is Premium only. |
| `email` | `text` | Officer's email address. Matches `auth.users.email`. |
| `full_name` | `text` | Display name. |
| `is_active` | `bool` | Default `true`. Set to `false` on soft delete — deactivated officers cannot log in. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., position title, contact number). |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

---

### 4.6 `students`

Campus-wide canonical student profiles. **Not org-scoped.** Organization access is governed through [`student_org_memberships`](#47-student_org_memberships). See [Section 2](#2-shared-student-data-model) for the full data-sharing model.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `auth_user_id` | `uuid` | FK → `auth.users`. Nullable, unique. Set when a student authenticates via Google OAuth — either through the Premium portal **or** via a public registration link (any tier). Import-only students have no auth account. **Auth does not imply dashboard access** — portal access is gated by tier + `is_verified`. |
| `id_number` | `text` | Unique. Campus-wide student ID. Primary lookup key for check-in, import deduplication, and cross-org matching. |
| `last_name` | `text` | |
| `first_name` | `text` | |
| `middle_name` | `text` | Nullable. |
| `course` | `text` | Degree program (e.g., "BS Computer Science"). |
| `major` | `text` | Nullable. Specialization within a course. |
| `level` | `text` | Year level (e.g., "1st Year", "2nd Year"). |
| `department` | `text` | Department within the college (e.g., "DCST", "Department of Education"). |
| `college` | `text` | College/faculty within the university. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store for future extensibility (e.g., contact info, notes). |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **RLS Note:** Since this table is not org-scoped, RLS policies must join through `student_org_memberships` to verify the requesting officer's organization has an active membership with the student. See [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) for the policy definitions.

> **QR Note:** Student QR codes are **derived deterministically from `id_number`** at the application layer — no QR image or data is stored in the database. The frontend generates a QR encoding the student's `id_number`, which officers scan to look up and record attendance. This avoids storage overhead and ensures the QR always reflects the current ID.

> **Soft Delete:** This table supports soft delete via `deleted_at` / `deleted_by`. RLS policies and application queries must filter `WHERE deleted_at IS NULL` by default.

---

### 4.7 `student_org_memberships`

Join table linking students to organizations. Tracks consent status, portal verification, and membership lifecycle.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `student_id` | `uuid` | FK → `students`. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `status` | `varchar` | `'pending_consent'` · `'pending_approval'` · `'active'` · `'inactive'` · `'removed'`. See status definitions below. |
| `is_verified` | `bool` | Default `false`. Set to `true` by an officer after identity verification. Required for Premium portal access. |
| `data_consent_given_at` | `timestamptz` | Nullable. Timestamp when the student consented to data sharing with this org. Null while `pending_consent`. Auto-set to `now()` for self-registered students. |
| `added_by` | `uuid` | FK → `auth.users`. Nullable. Officer who created this membership (via import or manual add). Null for self-registered students (student added themselves via public link). |
| `metadata` | `jsonb` | Nullable. Flexible key-value store for org-specific membership data (e.g., section, committee). |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

**Unique constraint:** `(student_id, organization_id)` — a student can only have one membership per org.

**Status definitions:**

| Status | Meaning |
|--------|---------|
| `pending_consent` | Student data exists from another org's import. This org requested access but the student has not yet consented. Officers **cannot** interact with this student's data until consent is granted. || `pending_approval` | Student self-registered via a public registration link ([`org_registration_links`](#49-org_registration_links)) and the link has `requires_approval = true`. The student's data is visible to officers in a review queue but is **not** counted as an active member until an officer approves. || `active` | Student is an active member. Full access within the org's tier capabilities. |
| `inactive` | Membership paused (e.g., student on leave, semester ended). Data retained but student excluded from active operations. |
| `removed` | Student was removed from the org. Soft delete — historical records (attendance, fines, payments) are preserved. |

---

### 4.8 `bulk_import_requests`

Tracks bulk student CSV imports. Every bulk import **requires a formal supporting document** (e.g., signed member list, official enrollment record) that must be reviewed and approved before processing.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `requested_by` | `uuid` | FK → `auth.users`. Officer who initiated the import. |
| `import_file_path` | `text` | Supabase Storage path to the uploaded CSV file. |
| `supporting_document_path` | `text` | Supabase Storage path to the formal document (PDF/image). This is the legal basis for importing student data (RA 10173 compliance). |
| `file_name` | `text` | Original filename of the CSV for display purposes. |
| `total_records` | `int4` | Number of student records detected in the CSV. Set after parsing. |
| `status` | `varchar` | `'pending_review'` · `'approved'` · `'rejected'` · `'processing'` · `'completed'` · `'failed'`. |
| `reviewed_by` | `uuid` | FK → `auth.users`. Nullable. Officer (Admin) or System Admin who reviewed this request. |
| `reviewed_at` | `timestamptz` | Nullable. When the review decision was made. |
| `rejection_reason` | `text` | Nullable. Required when `status = 'rejected'`. |
| `processed_at` | `timestamptz` | Nullable. When the import finished processing (success or fail). |
| `error_summary` | `jsonb` | Nullable. Structured error details for failed or partially failed imports (e.g., `{ "row_errors": [{ "row": 12, "reason": "Duplicate id_number" }] }`). |
| `created_at` | `timestamptz` | Auto-set on insert. |

**Import processing rules:**
- For each row in the CSV, the processor checks `students.id_number` for an existing record.
- **New student:** Creates a `students` row + a `student_org_memberships` row with status `active`.
- **Existing student (already in this org):** Skips or updates the existing membership. Logged in `error_summary`.
- **Existing student (in another org):** Creates a `student_org_memberships` row with status `pending_consent`. The student's canonical data is **not** overwritten.

---

### 4.9 `org_registration_links`

Org-generated public registration URLs for student self-onboarding. Officers create these links and distribute them (e.g., social media, group chats, bulletin boards). Students visit the link, authenticate via Google OAuth, and either create a new profile or join the organization with an existing one. **All tiers.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. The org this link registers students into. |
| `token` | `text` | Unique. URL-safe token (e.g., `nanoid`). Included in the public URL: `{app_url}/join/{token}`. |
| `label` | `text` | Nullable. Descriptive label for officer reference (e.g., "AY 2025-2026 Membership Drive", "Freshmen Onboarding"). |
| `academic_period_id` | `uuid` | FK → `academic_periods`. Nullable. If set, new memberships created via this link are automatically scoped to this period. Falls back to `organization_settings.default_academic_period_id` if null. |
| `requires_approval` | `bool` | Default `true`. When `true`, self-registered students get `student_org_memberships.status = 'pending_approval'` and must be approved by an officer. When `false`, students are immediately `active`. |
| `status` | `varchar` | `'active'` · `'expired'` · `'revoked'`. Only `'active'` links accept new registrations. Expiry is checked against `expires_at` at request time. |
| `expires_at` | `timestamptz` | Nullable. Token expiry. Null means no expiry — link remains active until manually revoked. |
| `max_registrations` | `int4` | Nullable. Maximum number of students that can register via this link. Null means unlimited. |
| `current_registrations` | `int4` | Default `0`. Incremented atomically on each successful registration. When `current_registrations >= max_registrations`, the link stops accepting new students. |
| `created_by` | `uuid` | FK → `auth.users`. Officer who created the link. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., target department filter, campaign tracking). |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **Self-Registration Flow:**
> 1. Officer creates a link via the org dashboard → `org_registration_links` row with `status = 'active'`.
> 2. Student visits `{app_url}/join/{token}` → app validates the token (status, expiry, capacity).
> 3. Student authenticates via **Google OAuth** → creates or links an `auth.users` account → sets `students.auth_user_id`.
> 4. **New student** (no matching `id_number`): student fills in profile form (id_number, name, course, etc.) → creates `students` row.
> 5. **Existing student** (matched by `auth_user_id` or student enters their `id_number` to claim profile): student confirms identity.
> 6. System creates a `student_org_memberships` row:
>    - `status = 'pending_approval'` if `requires_approval = true`
>    - `status = 'active'` if `requires_approval = false`
>    - `data_consent_given_at = now()` (consent is implicit — student is actively joining)
>    - `added_by = NULL` (self-registered)
> 7. `current_registrations` is incremented.
> 8. **Basic/Plus orgs:** Student sees a confirmation page ("You've joined [Org Name]"). No dashboard access.
> 9. **Premium orgs:** Student can access the portal dashboard once `is_verified = true` (officer verifies identity).

> **Auth ≠ Dashboard:** Google OAuth is used purely for **identity verification** on non-Premium tiers. The student's auth session is consumed during registration only. Dashboard access is exclusively a Premium feature gated by `organizations.tier = 'premium'` AND `student_org_memberships.is_verified = true` in RLS policies.

---

### 4.10 `academic_periods`

Org-defined academic year and semester definitions. Used as a foreign key across events, fees, fines, and clearances for consistent filtering and scoping. Each org manages its own periods.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `academic_year` | `text` | Academic year label (e.g., "2025-2026"). |
| `semester` | `text` | Semester within the year (e.g., "1st Semester", "2nd Semester", "Summer"). |
| `start_date` | `date` | Period start date. Used for date-range filtering and default assignment. |
| `end_date` | `date` | Period end date. |
| `is_current` | `bool` | Default `false`. Only one period per org should be `true` at a time. Enforced via a **partial unique index**: `CREATE UNIQUE INDEX idx_one_current_period_per_org ON academic_periods (organization_id) WHERE is_current = true`. Also mirrored in [`organization_settings.default_academic_period_id`](#43-organization_settings). |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

**Unique constraint:** `(organization_id, academic_year, semester)` — one period definition per org per year-semester combination.

---

### 4.11 `event_types`

Org-defined event categories. Controls how events are classified and which fine rules apply when events are archived. Each org configures its own set of event types.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `name` | `text` | Category name (e.g., "Major", "Minor", "Special"). |
| `description` | `text` | Nullable. Explanation of what qualifies as this event type. |
| `is_active` | `bool` | Default `true`. Inactive types are hidden from the event creation UI. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **Connection to fines:** The [`fine_types`](#416-fine_types) table has an `event_type_id` FK. When an event is archived, the automated fine trigger looks up fine_types that match the event's `event_type_id` and `trigger_source` (e.g., `'absence'`) to determine the fine amount.

---

### 4.12 `events`

Attendance events with lifecycle management, categorized by event type and scoped to an academic period. Supports QR-based student self-check-in via a unique event token. All tiers.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `event_type_id` | `uuid` | FK → `event_types`. Classifies this event (e.g., "Major", "Minor"). Determines which fine rules apply on archive. |
| `academic_period_id` | `uuid` | FK → `academic_periods`. Scopes this event to a specific academic year + semester for filtering and reporting. |
| `name` | `text` | Event name (e.g., "General Assembly", "Seminar"). |
| `description` | `text` | Nullable. Optional event details. |
| `event_date` | `date` | Calendar date of the event. |
| `start_time` | `timestamptz` | Scheduled start time. |
| `end_time` | `timestamptz` | Nullable. Scheduled end time. |
| `qr_token` | `text` | Unique. Auto-generated token (e.g., `nanoid` or `gen_random_uuid()`) encoded into a QR code for student self-check-in. Students scan the event's QR to register attendance. Regenerated if the officer resets it (e.g., to invalidate a leaked QR). |
| `status` | `varchar` | `'upcoming'` · `'ongoing'` · `'completed'` · `'archived'`. Archiving triggers automated fine generation (Premium). |
| `has_time_in` | `bool` | Default `true`. Whether check-in tracking is enabled. |
| `has_time_out` | `bool` | Default `false`. Whether check-out tracking is enabled. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., venue, speaker, external links). |
| `created_by` | `uuid` | FK → `auth.users`. Officer who created the event. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **QR Check-In Flow (dual QR model):**
> 1. **Officer scans student QR** — The student's QR (derived from `id_number`) is scanned by the officer's device. The app looks up the student and creates an `attendance_records` row.
> 2. **Student scans event QR** — The event's QR (encoding `qr_token`) is displayed at the venue. Students scan it with their portal app (Premium). The app validates the token, confirms the student's identity via their auth session, and creates the attendance record.
> Both flows converge on the same `attendance_records` table.

---

### 4.13 `attendance_records`

Individual check-in/out records. One row per student per event. All tiers.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `event_id` | `uuid` | FK → `events`. |
| `student_id` | `uuid` | FK → `students`. |
| `organization_id` | `uuid` | FK → `organizations`. Denormalized from `events` for RLS performance — avoids a join on every policy check. |
| `checked_in_at` | `timestamptz` | Nullable. Time-in timestamp. Set by recording officer or student self-check-in. |
| `checked_out_at` | `timestamptz` | Nullable. Time-out timestamp. Only relevant when `events.has_time_out = true`. |
| `recorded_by` | `uuid` | FK → `auth.users`. Officer who recorded this attendance entry. |
| `created_at` | `timestamptz` | Auto-set on insert. |

**Unique constraint:** `(event_id, student_id)` — one attendance record per student per event.

---

### 4.14 `fee_types`

Fee category definitions. Organizations configure these to define what they charge students. **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `name` | `text` | Display name (e.g., "Membership Fee", "Event Fee"). |
| `description` | `text` | Nullable. Additional context for officers. |
| `amount` | `numeric(10,2)` | Default fee amount. Can be overridden per-assignment in `fee_assignments`. |
| `frequency` | `varchar` | `'semester'` · `'event'` · `'one_time'` · `'custom'`. Describes the billing cycle for this fee category. |
| `required_for_clearance` | `bool` | Default `true`. When `true`, unpaid assignments of this fee type block clearance for the academic period. When `false`, the fee is tracked and collectible but does **not** prevent a student from being cleared. Officers configure this per fee type — e.g., "Membership Fee" = `true`, "Optional Social Event Fee" = `false`. The clearance computation function (`fn_get_blocking_items`) only considers fee assignments linked to fee types where `required_for_clearance = true`. |
| `is_active` | `bool` | Default `true`. Inactive fee types cannot be assigned to new students. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

---

### 4.15 `fee_assignments`

Per-student fee instances. Created when a fee type is assigned to a student or group of students. **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `fee_type_id` | `uuid` | FK → `fee_types`. |
| `student_id` | `uuid` | FK → `students`. |
| `organization_id` | `uuid` | FK → `organizations`. Denormalized for RLS. |
| `academic_period_id` | `uuid` | FK → `academic_periods`. Scopes this fee to a specific academic period. Enables per-semester fee filtering and reporting. |
| `amount` | `numeric(10,2)` | Actual amount for this student. Defaults to `fee_types.amount` but can be overridden (e.g., discounts, partial waivers). |
| `status` | `varchar` | `'pending'` · `'partially_paid'` · `'paid'` · `'waived'`. Updated automatically via a database trigger on `payment_allocations`. `'partially_paid'` when the sum of verified allocations is greater than zero but less than `amount`. `'paid'` when fully covered. |
| `due_date` | `date` | Nullable. Optional deadline for payment. |
| `assigned_by` | `uuid` | FK → `auth.users`. Officer who assigned this fee. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., discount reason, batch assignment reference). |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **Fee payment tracking:** Each fee assignment has a precise payment history via [`payment_allocations`](#420-payment_allocations). To display "how was this fee paid?", query:
> ```sql
> SELECT pa.amount_allocated, p.payment_date, p.payment_method, p.reference_number, p.status AS payment_status
> FROM payment_allocations pa
> JOIN payments p ON pa.payment_id = p.id
> WHERE pa.fee_assignment_id = $1 AND p.status = 'verified'
> ORDER BY p.payment_date;
> ```
> This returns every verified payment that contributed to this specific fee, with dates and methods. The UI should display this alongside the fee assignment details so officers can see at a glance: "Membership Fee — ₱200 — Paid (GCash, Feb 15)".
>
> **Computed `amount_paid`:** The total amount paid toward a fee is `SUM(pa.amount_allocated)` from verified payment allocations. This is not stored as a column — it is derived on read to avoid stale data. The `status` column (`pending` → `partially_paid` → `paid`) reflects this computation via trigger.

---

### 4.16 `fine_types`

Fine category definitions with trigger source and event type configuration. Officers define these; Premium orgs can enable automated triggers linked to specific event categories. **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `event_type_id` | `uuid` | FK → `event_types`. Nullable. Links this fine type to a specific event category. When set, automated fines only trigger for events of this type. When null, the fine type applies to all event types (or is manual-only). |
| `name` | `text` | Display name (e.g., "Major Event Absence Fine", "Minor Event Late Fine", "Misconduct Fine"). |
| `description` | `text` | Nullable. |
| `default_amount` | `numeric(10,2)` | Default fine amount applied when this type is triggered. |
| `trigger_source` | `varchar` | `'manual'` · `'absence'` · `'late'`. Values `'absence'` and `'late'` enable automated fine generation via database triggers (Premium only). `'manual'` means officer-assigned only. |
| `is_active` | `bool` | Default `true`. Inactive fine types are not applied by automation and hidden from assignment UI. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **Automation example:** An org creates two absence fine types — one linked to "Major" events at ₱50, another linked to "Minor" events at ₱25. When an event is archived, the trigger checks the event's `event_type_id`, finds matching fine_types with `trigger_source = 'absence'`, and generates fines at the corresponding amount.

---

### 4.17 `fines`

Individual fines linked to students. Can be manually assigned by officers (Plus+) or auto-generated by database triggers (Premium). **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `student_id` | `uuid` | FK → `students`. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `fine_type_id` | `uuid` | FK → `fine_types`. |
| `event_id` | `uuid` | FK → `events`. Nullable. Set for attendance-based fines (absence/late). Null for manually assigned fines unrelated to events. |
| `academic_period_id` | `uuid` | FK → `academic_periods`. Scopes this fine to a specific academic period. Inherited from the event's period for auto-generated fines, or set manually by the officer. |
| `amount` | `numeric(10,2)` | Fine amount. Defaults to `fine_types.default_amount` but can be overridden. |
| `status` | `varchar` | `'pending'` · `'partially_paid'` · `'paid'` · `'waived'` · `'appealed'`. Updated automatically via a database trigger on `payment_allocations`. `'partially_paid'` when the sum of verified allocations is greater than zero but less than `amount`. `'appealed'` is set when a student submits a `fine_appeal` (Premium). |
| `notes` | `text` | Nullable. Officer-provided context or justification. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., trigger metadata for auto-generated fines). |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_by` | `uuid` | FK → `auth.users`. Nullable. Null for auto-generated fines (trigger-created). Set for manually assigned fines. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

---

### 4.18 `payments`

Payment records for accumulated fee and fine settlements. A single payment covers a student's outstanding balance — individual fee/fine allocations are tracked in [`payment_allocations`](#420-payment_allocations). Supports both GCash (with Firebase Storage receipt) and in-person payments. Submitted by students (Premium) or recorded by officers (Plus+). **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `student_id` | `uuid` | FK → `students`. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `payment_date` | `date` | Date the payment was actually made. For GCash, this is the transaction date (may differ from the upload/submission date). For in-person, this is the date cash was received by the officer. |
| `amount` | `numeric(10,2)` | Total payment amount in PHP. Students pay their accumulated balance in a single transaction — the [`payment_allocations`](#420-payment_allocations) table tracks how this amount is distributed across individual fees and fines. |
| `payment_method` | `varchar` | `'gcash'` · `'in_person'`. Determines whether a receipt image is expected. |
| `receipt_file_path` | `text` | Nullable. Firebase Storage path. Format: `receipts/{org_id}/{payment_id}.{ext}`. Required for `payment_method = 'gcash'`, null for `'in_person'`. **Do not store signed URLs** — generate them on demand. See [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md#7-firebase-storage--gcash-receipt-upload-workflow). |
| `reference_number` | `text` | Nullable. GCash transaction reference number for cross-verification. Typically provided for GCash payments. |
| `official_receipt_number` | `text` | Nullable. Assigned by the officer after payment is verified. Org-defined numbering format (e.g., "OR-2026-0001"). |
| `status` | `varchar` | `'pending'` · `'verified'` · `'rejected'`. Officers verify payments by reviewing the receipt image (GCash) or confirming in-person receipt. |
| `submitted_by` | `uuid` | FK → `auth.users`. Student (Premium self-service) or officer (manual recording). |
| `verified_by` | `uuid` | FK → `auth.users`. Nullable. Officer who verified or rejected the payment. |
| `verified_at` | `timestamptz` | Nullable. When the verification decision was made. |
| `rejection_reason` | `text` | Nullable. Required when `status = 'rejected'`. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., notes, reference data). Allocation of payment to specific fees/fines is tracked in [`payment_allocations`](#420-payment_allocations) — do not store allocation data here. |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

---

### 4.19 `waivers`

Unified waiver system for fines and fees. When an officer grants a waiver, a record is created here and the linked fine or fee assignment status is updated to `'waived'`. This table provides a full audit trail for every waiver decision. **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `student_id` | `uuid` | FK → `students`. The student receiving the waiver. |
| `waiver_type` | `varchar` | `'fine'` · `'fee'`. Determines which FK is populated. |
| `fine_id` | `uuid` | FK → `fines`. Nullable. Set when `waiver_type = 'fine'`. |
| `fee_assignment_id` | `uuid` | FK → `fee_assignments`. Nullable. Set when `waiver_type = 'fee'`. |
| `appeal_id` | `uuid` | FK → `fine_appeals`. Nullable. Set when this waiver was auto-created from an approved fine appeal. Provides a direct audit link from waiver back to the originating dispute. |
| `reason` | `text` | Officer's justification for granting the waiver. Required. |
| `status` | `varchar` | `'pending'` · `'approved'` · `'rejected'`. When approved, the linked fine/fee status is set to `'waived'` via a database trigger or the approving Server Action. |
| `requested_by` | `uuid` | FK → `auth.users`. Nullable. The officer or student who initiated the waiver request. Null if system-initiated. |
| `reviewed_by` | `uuid` | FK → `auth.users`. Nullable. Officer (Admin/Manager) who approved or rejected the waiver. |
| `reviewed_at` | `timestamptz` | Nullable. When the review decision was made. |
| `review_notes` | `text` | Nullable. Officer's response or additional context for the decision. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., supporting document reference, partial waiver details). |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

**CHECK constraint:** Exactly one of `fine_id` or `fee_assignment_id` must be non-null:
```sql
CHECK (
  (fine_id IS NOT NULL AND fee_assignment_id IS NULL) OR
  (fine_id IS NULL AND fee_assignment_id IS NOT NULL)
)
```

**Waiver vs. Appeal:** Waivers are officer-initiated (or officer-approved). Appeals ([`fine_appeals`](#421-fine_appeals)) are student-initiated disputes. An approved appeal results in a waiver being created automatically with `appeal_id` set, providing a direct audit link. Both update the fine/fee status to `'waived'`, but the audit trail differs — `appeal_id` traces appeal-originated waivers.

---

### 4.20 `payment_allocations`

Maps each payment to the specific fees and/or fines it covers. A single payment can be allocated across multiple fees and fines (partial or full), enabling the **accumulated balance payment model** — students pay their total outstanding balance in one transaction, and the system tracks exactly which obligations each peso settled. **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `payment_id` | `uuid` | FK → `payments`. The parent payment being allocated. |
| `organization_id` | `uuid` | FK → `organizations`. Denormalized for RLS. |
| `fee_assignment_id` | `uuid` | FK → `fee_assignments`. Nullable. Set when this allocation covers a fee. |
| `fine_id` | `uuid` | FK → `fines`. Nullable. Set when this allocation covers a fine. |
| `amount_allocated` | `numeric(10,2)` | The portion of the payment applied to this specific fee or fine. Must be > 0. The sum of all allocations for a payment must not exceed `payments.amount`. |
| `created_by` | `uuid` | FK → `auth.users`. Nullable. Officer who created the allocation, or null for system auto-allocated payments. |
| `created_at` | `timestamptz` | Auto-set on insert. |

**CHECK constraint:** Exactly one of `fee_assignment_id` or `fine_id` must be non-null:
```sql
CHECK (
  (fee_assignment_id IS NOT NULL AND fine_id IS NULL) OR
  (fee_assignment_id IS NULL AND fine_id IS NOT NULL)
)
```

**Business rules:**

| Rule | Detail |
|------|--------|
| **Accumulated payments.** | Students accumulate fees and fines over an academic period. When they pay, they submit a single payment for their total (or partial) outstanding balance. Officers (or the system) allocate the payment across the individual obligations. |
| **Sum constraint.** | The total of all `amount_allocated` rows for a given `payment_id` must equal `payments.amount`. Enforced by a CHECK constraint or trigger. |
| **No over-allocation.** | The total allocated to a single fee or fine must not exceed its `amount`. Enforced via trigger: `SUM(amount_allocated) WHERE fee_assignment_id = X` must be `<= fee_assignments.amount`. |
| **No overpayment.** | A payment's total allocations must exactly equal `payments.amount` — **overpayments are prevented**, not refunded. The UI and Server Action must calculate the student's total outstanding balance before submission and cap the payment amount. If a student's balance is ₱350, the system should not accept a ₱500 payment. If an edge case produces surplus (e.g., a fine is waived after payment submission but before verification), the officer rejects the payment and the student resubmits at the correct amount. No `student_credits` table is needed — this is a student org context, not a commercial billing system, and the operational overhead of managing credit balances outweighs the benefit. |
| **Status sync.** | A database trigger on `payment_allocations` (after INSERT, UPDATE, DELETE) recalculates the linked fee/fine status. See **Status Transition Logic** below. |
| **Reversal support.** | If a payment is rejected or soft-deleted, a trigger on `payments` deletes (or marks void) its `payment_allocations` rows. This cascades into the status sync trigger, recalculating each linked fee/fine status downward. |
| **Payment history per obligation.** | The full payment history for any fee or fine is queryable via: `SELECT pa.*, p.payment_date, p.payment_method FROM payment_allocations pa JOIN payments p ON pa.payment_id = p.id WHERE pa.fee_assignment_id = $1` — returns every payment that contributed to settling that fee, with dates and methods. |

> **Why accumulated payments?** In student organizations, students accumulate multiple fees (membership, event fees) and fines (absences) throughout a semester. Requiring them to pay each individually would be operationally impractical. The accumulated payment model lets a student pay ₱500 in one transaction and have it distributed across their ₱200 membership fee, ₱150 event fee, and ₱150 in absence fines — with precise allocation tracked for audit and clearance computation.

> **Per-fee / per-fine payment history:** Unlike a flat "payments" list, the allocation model gives officers **per-obligation visibility**. When viewing a student's "Membership Fee" row, the UI joins through `payment_allocations` to show exactly which payments settled it, when, and via what method. The same applies to fines. This is the answer to "how do I know this fee is paid?" — the allocation rows *are* the receipt trail for each individual obligation.
>
> Example UI display for a student's fee list:
> | Fee | Amount | Paid | Status | Payment Details |
> |-----|--------|------|--------|-----------------|
> | Membership Fee | ₱200 | ₱200 | Paid | GCash — Feb 15, 2026 (OR-2026-0042) |
> | Event Fee | ₱150 | ₱100 | Partially Paid | GCash — Feb 15, 2026 (₱100 of ₱150) |
> | Social Event Fee | ₱50 | ₱0 | Pending | *Not required for clearance* |

**Status Transition Logic (trigger on `payment_allocations`):**

When a `payment_allocations` row is inserted, updated, or deleted, the trigger recalculates the status of the linked fee assignment or fine:

```
For each affected fee_assignment_id or fine_id:
  1. Compute: total_paid = SUM(pa.amount_allocated)
     FROM payment_allocations pa
     JOIN payments p ON pa.payment_id = p.id
     WHERE pa.fee_assignment_id = target_id  -- (or fine_id)
       AND p.status = 'verified'             -- only count verified payments

  2. Fetch: obligation_amount = fee_assignments.amount  -- (or fines.amount)
           current_status = fee_assignments.status      -- (or fines.status)

  3. Skip if current_status IN ('waived', 'appealed')   -- waivers and appeals take precedence

  4. New status:
     IF total_paid = 0                      → 'pending'
     IF total_paid > 0 AND < obligation     → 'partially_paid'
     IF total_paid >= obligation             → 'paid'

  5. UPDATE fee_assignments SET status = new_status, updated_at = now()
     WHERE id = target_id AND status != new_status
```

**Key interaction with waivers:** When a waiver is approved, its own trigger sets the fee/fine status to `'waived'` directly. The allocation trigger respects this by skipping recalculation if the current status is `'waived'` or `'appealed'`. If a waiver is later rejected, the waiver trigger must call the same recalculation logic to restore the correct payment-based status.

> **RLS:** RLS is enabled on `payment_allocations`. Officers can read/write allocations where `organization_id` matches their org. Students (Premium portal) can read allocations for their own payments (`payment_id IN (SELECT id FROM payments WHERE student_id = auth.uid())`) but cannot write. Policies follow the same pattern as other org-scoped financial tables.

---

### 4.21 `fine_appeals`

Student-submitted fine disputes. Students can contest fines they believe are incorrect. When approved, the system automatically creates a [`waivers`](#419-waivers) record and updates the fine status. **Premium tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `fine_id` | `uuid` | FK → `fines`. The fine being contested. |
| `student_id` | `uuid` | FK → `students`. Must match `fines.student_id` — students can only appeal their own fines. |
| `organization_id` | `uuid` | FK → `organizations`. Denormalized for RLS. |
| `reason` | `text` | Student's explanation for the appeal. |
| `status` | `varchar` | `'pending'` · `'approved'` · `'rejected'`. When approved, the linked fine's status is updated to `'waived'`. |
| `reviewed_by` | `uuid` | FK → `auth.users`. Nullable. Officer who reviewed the appeal. |
| `reviewed_at` | `timestamptz` | Nullable. When the review decision was made. |
| `review_notes` | `text` | Nullable. Officer's response or justification for the decision. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

---

### 4.22 `clearances`

Per-period clearance status for each student within an organization. Clearance status is recomputed by a database trigger whenever payment allocations, waivers, or fine/fee statuses change. Blocking items (outstanding fees and fines) are computed dynamically via a view or RPC function — not stored in this table. **Plus+ tier required.**

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `student_id` | `uuid` | FK → `students`. |
| `organization_id` | `uuid` | FK → `organizations`. |
| `academic_period_id` | `uuid` | FK → `academic_periods`. The period this clearance applies to. Links to the same `academic_periods` table used by events, fees, and fines for consistent scoping. |
| `status` | `varchar` | `'pending'` · `'cleared'` · `'not_cleared'` · `'overridden'`. `'overridden'` indicates a manual officer override (e.g., special arrangement). Recomputed automatically by a trigger on `payment_allocations`, `waivers`, `fines`, and `fee_assignments` changes. |
| `cleared_at` | `timestamptz` | Nullable. When the student was marked as cleared (automatically via balance check or manually). |
| `overridden_by` | `uuid` | FK → `auth.users`. Nullable. Officer who manually overrode the clearance status. |
| `override_reason` | `text` | Nullable. Required when `status = 'overridden'`. Justification for the manual override. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store (e.g., certificate serial number, additional notes). |
| `deleted_at` | `timestamptz` | Nullable. Set on soft delete. Non-null means the record is logically deleted. |
| `deleted_by` | `uuid` | FK → `auth.users`. Nullable. The user who performed the soft delete. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

**Unique constraint:** `(student_id, organization_id, academic_period_id)` — one clearance record per student per org per period.
> **Blocking items are computed, not stored.** Outstanding fines and fees are already tracked in `fines`, `fee_assignments`, and `payment_allocations`. Storing a denormalized JSON list would duplicate this data and drift out of sync when payments or waivers are processed. Instead, use a **database view or RPC function** (e.g., `fn_get_blocking_items(student_id, org_id, period_id)`) to compute blocking items on demand by querying:
> - **Fines** with `status NOT IN ('paid', 'waived')` — all unpaid fines block clearance.
> - **Fee assignments** with `status NOT IN ('paid', 'waived')` **AND** where the linked `fee_types.required_for_clearance = true` — only clearance-required fees block. Optional fees (e.g., social event fees) are excluded even if unpaid.
>
> If performance requires caching, use a materialized view refreshed by triggers on the relevant tables.
---

### 4.23 `subscription_payments`

Tracks physical cash payments from organizations to the VERIS platform for their annual subscription. Organizations pay in-person (cash); this table logs and tracks those payments. **Platform-scoped** — managed by System Admins.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. The org that made the payment. |
| `amount` | `numeric(10,2)` | Payment amount in PHP. |
| `payment_date` | `date` | Date the physical payment was received. |
| `period_covered_start` | `date` | Start of the subscription period this payment covers. |
| `period_covered_end` | `date` | End of the subscription period this payment covers. |
| `status` | `varchar` | `'pending'` · `'confirmed'` · `'voided'`. System Admin confirms after receiving cash. `'voided'` for erroneous entries. |
| `receipt_number` | `text` | Nullable. Official receipt number issued to the org upon payment. |
| `notes` | `text` | Nullable. Additional context (e.g., "Paid by Org President", "Partial payment"). |
| `received_by` | `uuid` | FK → `auth.users`. System Admin who received or recorded the payment. |
| `confirmed_at` | `timestamptz` | Nullable. When the payment was confirmed. |
| `metadata` | `jsonb` | Nullable. Flexible key-value store for future extensibility. |
| `created_at` | `timestamptz` | Auto-set on insert. |
| `updated_at` | `timestamptz` | Auto-set on insert and update. |

> **Note:** This table is for **platform subscription billing only** (orgs paying for VERIS). It is completely separate from the [`payments`](#418-payments) table, which tracks student fee/fine payments within organizations.

---

### 4.24 `notification_queue`

Transactional email notification queue. System-triggered emails are enqueued here and processed by a background worker (e.g., Supabase Edge Function on a cron schedule or triggered by database events). **No user-composed messages** — all templates are system-defined.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `organization_id` | `uuid` | FK → `organizations`. Nullable. Null for platform-level notifications. |
| `recipient_type` | `varchar` | `'student'` · `'officer'`. Determines which user table to look up for email resolution. |
| `recipient_id` | `uuid` | FK → `students` or `officers` depending on `recipient_type`. The notification target. |
| `notification_type` | `varchar` | Enum-like: `'payment_verified'` · `'payment_rejected'` · `'fine_assigned'` · `'fine_waived'` · `'clearance_updated'` · `'appeal_resolved'` · `'waiver_approved'` · `'waiver_rejected'`. Determines the email template used. |
| `payload` | `jsonb` | Template variables (e.g., `{ "fine_amount": 50, "event_name": "General Assembly", "student_name": "Juan Dela Cruz" }`). Must contain all data needed to render the email without additional DB queries. |
| `status` | `varchar` | `'pending'` · `'sent'` · `'failed'` · `'skipped'`. `'skipped'` when the recipient has no email or has opted out. |
| `sent_at` | `timestamptz` | Nullable. When the email was successfully sent. |
| `error` | `text` | Nullable. Error message if sending failed. Used for retry logic and debugging. |
| `retry_count` | `int4` | Default `0`. Number of send attempts. Workers should cap retries (e.g., max 3). |
| `created_at` | `timestamptz` | Auto-set on insert. |

> **Processing model:** Notifications are enqueued by database triggers or Server Actions. A Supabase Edge Function runs on a cron schedule (e.g., every 60 seconds), picks up `status = 'pending'` rows, sends emails via a transactional email provider (e.g., Resend, Postmark), and updates `status` to `'sent'` or `'failed'`.

---

### 4.25 `audit_log`

Immutable, append-only log of sensitive operations across the platform. Written by database triggers and `SECURITY DEFINER` RPC functions — **never by application code directly**.

| Column | Type | Dev Notes |
|--------|------|-----------|
| `id` | `uuid` | PK. |
| `table_name` | `text` | Source table that triggered the log entry (e.g., `'payments'`, `'fines'`, `'students'`). |
| `record_id` | `uuid` | ID of the affected row in the source table. |
| `action` | `varchar` | `'INSERT'` · `'UPDATE'` · `'DELETE'` · `'SYSADMIN_READ'`. `'SYSADMIN_READ'` is logged when a System Admin accesses student PII via an audited RPC function. |
| `old_data` | `jsonb` | Nullable. Previous row state (null for `INSERT` and `SYSADMIN_READ`). |
| `new_data` | `jsonb` | Nullable. New row state (null for `DELETE`). For `SYSADMIN_READ`, stores the access reason. |
| `performed_by` | `uuid` | FK → `auth.users`. The user who performed the action. |
| `organization_id` | `uuid` | Nullable. Null for platform-level actions (e.g., System Admin operations on `org_invites`). |
| `ip_address` | `inet` | Nullable. Client IP address at the time of the action. Passed by the application layer via `set_config('app.client_ip', ...)` session variable, read by audit triggers. Essential for detecting suspicious access patterns. |
| `user_agent` | `text` | Nullable. Client User-Agent string. Passed via `set_config('app.user_agent', ...)`. Useful for identifying the device/browser used during sensitive operations. |
| `created_at` | `timestamptz` | Auto-set on insert. **Immutable** — no `UPDATE` or `DELETE` policies exist for this table on any role. |

---

## 5. Soft Delete Conventions

Soft delete is applied to **critical tables with financial or legal significance** where permanent deletion would violate audit requirements or orphan historical records.

### Tables with Soft Delete

| Table | Rationale |
|-------|----------|
| `students` | Campus-wide identity. Deletion would orphan attendance, fine, and payment records across all orgs. |
| `student_org_memberships` | Membership history must be preserved for audit and clearance verification. |
| `officers` | Officer identity is referenced by `created_by`, `verified_by`, etc. across many tables. |
| `fines` | Financial records. Required for clearance computation and audit trail. |
| `payments` | Financial records. Required for balance computation and receipt tracking. |
| `fee_assignments` | Financial records. Required for clearance computation. |
| `clearances` | Legal significance — clearance status is used for certificate generation. |

### Implementation Rules

| Rule | Detail |
|------|--------|
| **Columns.** | `deleted_at timestamptz` (nullable, default null) + `deleted_by uuid` (FK → `auth.users`, nullable). |
| **RLS default filter.** | All RLS `SELECT` policies on soft-delete tables must include `AND deleted_at IS NULL`. This ensures soft-deleted rows are invisible to normal queries. |
| **Admin recovery.** | System Admins or Org Admins can view soft-deleted records via a dedicated RPC function that bypasses the default filter. This is audit-logged. |
| **Hard delete.** | Never performed by the application. Reserved for data retention policy enforcement (e.g., automated cleanup after N years), executed by a scheduled database job with full audit logging. |
| **Cascade behavior.** | Soft-deleting a parent (e.g., a student) does **not** cascade to children (e.g., their fines). Children remain visible but the parent shows as `[Deleted]` in the UI. |
| **Unique constraints.** | Soft-deleted rows must be excluded from unique constraint checks. Use partial unique indexes: `CREATE UNIQUE INDEX ... WHERE deleted_at IS NULL`. |

---

## 6. Indexing Strategy

Specific `CREATE INDEX` statements are omitted from this document. Apply the following indexing principles during implementation:

| Strategy | Where to Apply | Rationale |
|----------|---------------|-----------|
| **Index every `organization_id` FK.** | All org-scoped tables (`events`, `attendance_records`, `fines`, `payments`, `payment_allocations`, `fee_assignments`, `fine_types`, `fee_types`, `clearances`, `fine_appeals`, `waivers`, `student_org_memberships`, `bulk_import_requests`, `org_registration_links`, `academic_periods`, `event_types`, `organization_settings`, `notification_queue`). | Every RLS policy filters on `organization_id`. Without an index, every policy check triggers a sequential scan. |
| **Index every `student_id` FK.** | `attendance_records`, `fines`, `payments`, `fee_assignments`, `fine_appeals`, `clearances`, `waivers`, `student_org_memberships`. | Student-centric queries (balance lookups, clearance checks, portal dashboard) filter by `student_id`. |
| **Index every `academic_period_id` FK.** | `events`, `fee_assignments`, `fines`, `clearances`, `organization_settings`. | Period-based filtering is the primary scoping mechanism for UI views and reports. |
| **Index every `event_type_id` FK.** | `events`, `fine_types`. | Used by the automated fine trigger to look up matching fine rules when an event is archived. |
| **Composite index on unique constraints.** | `(event_id, student_id)` on `attendance_records`, `(student_id, organization_id)` on `student_org_memberships`, `(student_id, organization_id, academic_period_id)` on `clearances`, `(organization_id, academic_year, semester)` on `academic_periods`. | Unique constraints implicitly create indexes, but verify these are in place. |
| **Index `students.id_number`.** | `students` table. | Primary lookup key for check-in search, import deduplication, and cross-org matching. High-frequency queries. |
| **Index `events.qr_token`.** | `events` table. | QR token lookup during student self-check-in must be fast — it's on the critical path of the attendance flow. Unique constraint provides this implicitly. |
| **Index `status` columns on high-traffic tables.** | `payments.status`, `fines.status`, `fine_appeals.status`, `waivers.status`, `bulk_import_requests.status`, `student_org_memberships.status`, `events.status`, `notification_queue.status`, `subscription_payments.status`, `org_registration_links.status`. | Officers frequently filter by status (e.g., "show pending payments", "show active members"). Combine with `organization_id` for a composite index where appropriate. |
| **Index `deleted_at` on soft-delete tables.** | `students`, `student_org_memberships`, `officers`, `fines`, `payments`, `fee_assignments`, `clearances`. | Every RLS policy and most queries filter `WHERE deleted_at IS NULL`. A partial index (`WHERE deleted_at IS NULL`) is preferred for write efficiency. |
| **Index `notification_queue` for worker pickup.** | `(status, created_at) WHERE status = 'pending'`. | The background email worker queries for pending notifications ordered by creation time. |
| **Index `subscription_payments.organization_id`.** | `subscription_payments` table. | System Admins query payment history per organization. |
| **Index `audit_log` for query patterns.** | `(table_name, record_id)`, `(performed_by)`, `(organization_id, created_at)`. | System Admins query audit logs by table, by user, and by org + time range. |
| **Index `org_invites.token`.** | `org_invites` table. | Token lookup during registration must be fast — it's on the critical path of the onboarding flow. |
| **Index `org_registration_links.token`.** | `org_registration_links` table. | Token lookup when students visit a public registration link must be fast — it's on the critical path of the self-registration flow. Unique constraint provides this implicitly. |
| **Index `payment_allocations` FKs.** | `(payment_id)`, `(fee_assignment_id) WHERE fee_assignment_id IS NOT NULL`, `(fine_id) WHERE fine_id IS NOT NULL` on `payment_allocations`. | Allocation lookups by payment (for receipt display), by fee (for balance computation), and by fine (for status sync). Critical for the accumulated payment model — every payment verification triggers allocation-based recalculation. |
| **Do not over-index.** | All tables. | Every index adds write overhead. Only add indexes that serve a proven query pattern. Profile with `EXPLAIN ANALYZE` before adding speculative indexes. |

---

## 7. Related Documents

| Document | Contents |
|----------|----------|
| [01-system-overview.md](./01-system-overview.md) | Product context, tiers, student data model, multi-tenancy |
| [02-architecture-principles.md](./02-architecture-principles.md) | Thick DB / Thin API, feature-driven structure |
| [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) | RLS policies, RPC functions, triggers, RBAC enforcement, Firebase upload workflow |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | Phased rollout: Basic → Plus → Premium |
