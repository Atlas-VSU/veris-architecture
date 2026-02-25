# VERIS

**Student Attendance & Financial Management System for Student Organizations**

> *VERIS — from Latin: truth, record.*

VERIS is a multi-tenant SaaS platform that centralizes attendance tracking, membership management, financial collection, and student clearance for Philippine academic student organizations. Organizations subscribe to a tier, and VERIS handles the operational overhead so officers don't have to.

---

## Subscription Tiers

| Tier | Price | Core Function |
|---|---|---|
| **Basic** | ₱2 / student / yr | Attendance management |
| **Plus** | ₱3 / student / yr | Basic + financial management & manual fines |
| **Premium** | ₱4 / student / yr | Full suite + student self-service portal |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router) |
| Database & Auth | Supabase (PostgreSQL + Supabase Auth) |
| File Storage | Firebase Storage *(GCash receipts only)* |
| Client State | React Query (TanStack Query) |
| Styling | Tailwind CSS |

---

## Project Structure

```
veris/
├── src/
│   ├── app/                    # Next.js App Router (pages & layouts)
│   │   ├── (officer)/          # Officer-facing routes
│   │   ├── (student)/          # Student portal routes (Premium)
│   │   └── api/                # External webhooks ONLY (no internal mutations)
│   │
│   ├── features/               # Feature-driven modules (primary domain code)
│   │   ├── attendance/
│   │   ├── events/
│   │   ├── members/
│   │   ├── financials/         # Plus + Premium
│   │   ├── fines/              # Plus + Premium
│   │   ├── clearance/          # Plus + Premium
│   │   └── student-portal/     # Premium only
│   │
│   ├── components/             # Shared, domain-agnostic UI components only
│   ├── lib/                    # Supabase client, Firebase client, utility fns
│   └── types/                  # Global TypeScript types & Supabase DB types
│
├── supabase/
│   ├── migrations/             # All schema changes as versioned migrations
│   └── seed.sql                # Dev seed data
│
└── docs/
    └── architecture/           # System Architecture Document (SAD)
```

> **Convention:** All domain logic lives inside `src/features/<domain>/`. Do not create a global `hooks/`, `utils/`, or `services/` folder for domain-specific code. See the System Architecture Document for the full feature-module structure.

---

## Core Data Flow Rules

These are **non-negotiable** conventions. Read them before writing any data-fetching or mutation code.

| Operation | Method | Why |
|---|---|---|
| Initial page data | React Server Component (RSC) | Zero client JS overhead; data arrives with the HTML |
| Live / reactive data | React Query (`useQuery`) inside Client Components | Background refetch, caching, optimistic UI |
| All writes / mutations | Next.js Server Actions | Never exposes DB credentials to the client; no `app/api/` routes for internal data |
| File uploads (GCash receipts) | Server Action–generated Firebase signed URL | Client uploads directly to Firebase; Server Action records the reference in Supabase |

---

## Getting Started

### Prerequisites

- Node.js >= 18
- A Supabase project (with the VERIS schema applied via migrations)
- A Firebase project (Storage bucket configured)

### Environment Variables

Create a `.env.local` file at the project root. Required variables:

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=        # Server-side only — never expose to client

# Firebase
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=            # Server-side only
FIREBASE_PRIVATE_KEY=             # Server-side only
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
```

> **Security:** Variables prefixed with `NEXT_PUBLIC_` are exposed to the browser. Service role keys and Firebase private keys must **never** carry the `NEXT_PUBLIC_` prefix.

### Installation

```bash
# Install dependencies
npm install

# Apply Supabase migrations
npx supabase db push

# Run development server
npm run dev
```

---

## Architecture Documentation

Full architectural decisions, database schema, RLS policies, and implementation roadmap are documented in `/docs/architecture/`.

| Document | Status |
|---|---|
| Section 1 — System Overview | ✅ Draft |
| Section 2 — Architecture Principles & Conventions | 🔲 Pending |
| Section 3 — Dev Notes: Next.js | 🔲 Pending |
| Section 4 — Dev Notes: Supabase & Firebase | 🔲 Pending |
| Section 5 — Database Schema & Data Dictionary | 🔲 Pending |
| Section 6 — Implementation Roadmap | 🔲 Pending |

---

## Development Conventions (Quick Reference)

**Do:**
- Write business rules as PostgreSQL RLS policies and triggers — not in Server Actions
- Scope all data queries to `org_id` — multi-tenancy is enforced at the DB level
- Use Server Actions for every mutation — no exceptions for internal operations
- Keep feature code inside `src/features/<domain>/`

**Do Not:**
- Create `app/api/` routes for internal mutations
- Fetch data in Client Components on initial render (use RSC instead)
- Store Firebase private keys or Supabase service role keys in `NEXT_PUBLIC_` variables
- Bypass RLS by using the service role key in client-accessible code paths

---

## License

Proprietary. All rights reserved. © 2026 VERIS / Atlas Dev Team.
