# veris-architecture

> **The single source of truth for VERIS system design, architecture decisions, and development standards.**

VERIS is a SaaS Student Attendance and Financial Management System built for student organizations. This repository contains the architectural documentation that governs how the system is designed, built, and maintained.

---

## Who This Is For

This repository is for the **Atlas Dev Team** — engineers, contributors, and technical stakeholders working on VERIS. Before writing a single line of code, every developer is expected to read and understand the documents here.

> If your implementation contradicts something in this repo, **the architecture doc wins** — or you open a PR to formally change it.

---

## Repository Structure

| File | Contents |
|------|----------|
| [`01-system-overview.md`](./01-system-overview.md) | Product context, subscription tiers, tech stack, high-level architecture |
| [`02-architecture-principles.md`](./02-architecture-principles.md) | Feature-driven folder structure, Thick DB / Thin API philosophy |
| [`03-nextjs-guidelines.md`](./03-nextjs-guidelines.md) | Data flow matrix, RSC vs Client Components, Server Actions |
| [`04-supabase-firebase-auth.md`](./04-supabase-firebase-auth.md) | RBAC definitions, RLS policies, Firebase upload workflow |
| [`05-database-schema.md`](./05-database-schema.md) | SQL table definitions, relationships, triggers |
| [`06-implementation-roadmap.md`](./06-implementation-roadmap.md) | Phased rollout: Basic → Plus → Premium |

---

## Quick Links

- **Tech Stack:** Next.js (App Router) · Supabase (PostgreSQL + Auth) · Firebase Storage · React Query · Tailwind CSS · TypeScript
- **Core Principle:** Business logic lives in the database (RLS + Triggers), not in the application layer.
- **Data Flow Rule:** RSC for fetching → React Query for live data → Server Actions for all writes. No `app/api/` routes except external webhooks.

---

## Contributing

1. All architectural changes require a PR with a clear rationale.
2. If a decision contradicts an existing doc, update the relevant `.md` file in the same PR.
3. Use the section/heading structure already established in each file — do not restructure arbitrarily.
4. Code snippets in docs must include the target file path as a comment on the first line.

---

*Maintained by the Atlas Dev Team. Questions? Raise an issue or ping the lead architect.*
