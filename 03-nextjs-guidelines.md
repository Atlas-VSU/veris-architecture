# 03 — Next.js Guidelines

> **Document Status:** Living Document · Atlas Dev Team  
> **Last Updated:** 2026  
> **Related Docs:** [Architecture Principles](./02-architecture-principles.md) · [Supabase & Auth](./04-supabase-firebase-auth.md) · [Database Schema](./05-database-schema.md)

---

## Purpose

This document is the implementation-level guide for building VERIS with Next.js App Router. It covers routing conventions, component patterns, Server Actions, ShadCN composition, and the best practices developers must follow within the VERIS architecture.

Read [02-architecture-principles.md](./02-architecture-principles.md) first. This document assumes you understand the Feature-Driven Architecture, Thick DB / Thin API, and Strict Data Flow rules defined there. Detailed feature-specific flows (attendance, financials, etc.) are documented in separate feature guides referenced from [06-implementation-roadmap.md](./06-implementation-roadmap.md).

---

## 1. App Router Structure

VERIS uses **Route Groups** to separate the four distinct application surfaces — auth, officer dashboard, student portal, and system admin — without affecting URL paths.

### Full Route Tree (Subject to change over time)

```
src/app/
├── layout.tsx                        # Root layout: fonts, global providers, <html>
├── globals.css                       # Tailwind + ShadCN CSS variables
├── not-found.tsx                     # Global 404 page
│
├── (auth)/                           # Route Group: Authentication
│   ├── layout.tsx                    # Centered auth layout (no sidebar)
│   ├── login/
│   │   └── page.tsx                  # Officer email/password login
│   ├── signup/
│   │   └── page.tsx                  # Officer registration
│   ├── forgot-password/
│   │   └── page.tsx                  # Password recovery
│   └── student-login/
│       └── page.tsx                  # Student Google OAuth sign-in (Premium)
│
├── (dashboard)/                      # Route Group: Officer Dashboard
│   ├── layout.tsx                    # Dashboard shell (Sidebar, Header, auth guard)
│   ├── page.tsx                      # Dashboard home / overview
│   ├── attendance/
│   │   ├── page.tsx                  # Event list (RSC: fetches events)
│   │   └── [eventId]/
│   │       └── page.tsx              # Single event attendance view
│   ├── members/
│   │   ├── page.tsx                  # Member directory
│   │   └── import/
│   │       └── page.tsx              # Bulk import flow
│   ├── financials/                   # Plus+ only
│   │   ├── page.tsx                  # Financial overview
│   │   ├── fees/
│   │   │   └── page.tsx              # Fee configuration
│   │   ├── fines/
│   │   │   └── page.tsx              # Fine management
│   │   └── payments/
│   │       └── page.tsx              # GCash payment verification queue
│   ├── clearance/                    # Plus+ only
│   │   └── page.tsx                  # Clearance management
│   └── settings/
│       ├── page.tsx                  # Organization settings
│       ├── officers/
│       │   └── page.tsx              # Officer management (Premium RBAC)
│       └── branding/
│           └── page.tsx              # Custom branding (Premium)
│
├── (portal)/                         # Route Group: Student Portal (Premium)
│   ├── layout.tsx                    # Portal layout (student nav, auth guard)
│   ├── page.tsx                      # Student dashboard (balance, recent activity)
│   ├── onboarding/
│   │   └── page.tsx                  # Student self-registration form (post-OAuth)
│   ├── payments/
│   │   └── page.tsx                  # Payment history + GCash upload
│   ├── fines/
│   │   ├── page.tsx                  # Fine list
│   │   └── [fineId]/
│   │       └── appeal/
│   │           └── page.tsx          # Fine appeal submission
│   └── clearance/
│       └── page.tsx                  # Clearance self-check + certificate download
│
├── (system-admin)/                   # Route Group: System Admin (Platform Management)
│   ├── layout.tsx                    # System Admin shell (auth guard: system_admin only)
│   ├── page.tsx                      # Platform overview dashboard
│   ├── organizations/
│   │   ├── page.tsx                  # Organization list + management
│   │   └── [orgId]/
│   │       └── page.tsx              # Single org detail (tier, status, officers)
│   ├── invites/
│   │   └── page.tsx                  # Generate & manage org onboarding invite URLs
│   └── audit-log/
│       └── page.tsx                  # Platform-wide audit log viewer
│
└── api/                              # API Routes — ONLY for external webhooks
    └── webhooks/
        └── [provider]/
            └── route.ts              # Webhook handler (e.g., payment gateway callback)
```

### Route Group Conventions

| Group | URL Prefix | Layout | Auth Requirement |
|-------|-----------|--------|-----------------|
| `(auth)` | `/login`, `/signup`, etc. | Centered, minimal | None (public) |
| `(dashboard)` | `/`, `/attendance`, `/members`, etc. | Sidebar + Header shell | Officer session required |
| `(portal)` | `/portal`, `/portal/payments`, etc. | Student nav | Student session required (Premium org only) |

### Page File Rules

Every `page.tsx` in `src/app/` follows these constraints:

1. **It is a React Server Component** (no `"use client"` directive).
2. **It fetches initial data** using the Supabase server client directly.
3. **It delegates rendering** to feature components imported from `src/features/`.
4. **It contains zero business logic** — no conditional auth checks, no tier validation.

```tsx
// src/app/(dashboard)/attendance/page.tsx
import { createServerClient } from "@/lib/supabase/server";
import { EventList } from "@/features/attendance/components/EventList";

export default async function AttendancePage() {
  const supabase = await createServerClient();

  const { data: events } = await supabase
    .from("events")
    .select("*")
    .order("event_date", { ascending: false });

  return <EventList initialEvents={events ?? []} />;
}
```

**What this page does NOT do:**

- ❌ Check if the user is logged in (the `(dashboard)/layout.tsx` auth guard handles it)
- ❌ Check if the org has the right tier (RLS handles it)
- ❌ Contain any `"use client"` interactivity (that lives in `EventList`)

---

## 2. Layout Patterns

### Root Layout

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { QueryProvider } from "@/components/providers/QueryProvider";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "VERIS",
  description: "Student Attendance & Financial Management",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

### Dashboard Layout (Auth Guard)

```tsx
// src/app/(dashboard)/layout.tsx
import { redirect } from "next/navigation";
import { createServerClient } from "@/lib/supabase/server";
import { DashboardShell } from "@/components/layout/DashboardShell";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const supabase = await createServerClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) redirect("/login");

  // Officers only — students with Google OAuth are redirected to portal
  const role = user.app_metadata?.role;
  if (role === "student") redirect("/portal");
  if (role === "system_admin") redirect("/system-admin");

  return <DashboardShell user={user}>{children}</DashboardShell>;
}
```

### Student Portal Layout (Auth Guard + Tier Check)

```tsx
// src/app/(portal)/layout.tsx
import { redirect } from "next/navigation";
import { createServerClient } from "@/lib/supabase/server";
import { PortalShell } from "@/components/layout/PortalShell";

export default async function PortalLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const supabase = await createServerClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) redirect("/student-login");

  // Verify student role and Premium tier
  const role = user.app_metadata?.role;
  if (role !== "student") redirect("/");

  // Check if student has completed onboarding (self-registration)
  const isVerified = user.app_metadata?.is_verified;
  const currentPath = /* extracted from headers */;
  if (!isVerified && !currentPath.includes("/onboarding")) {
    redirect("/portal/onboarding");
  }

  return <PortalShell user={user}>{children}</PortalShell>;
}
```

### System Admin Layout (Auth Guard)

```tsx
// src/app/(system-admin)/layout.tsx
import { redirect } from "next/navigation";
import { createServerClient } from "@/lib/supabase/server";
import { SystemAdminShell } from "@/components/layout/SystemAdminShell";

export default async function SystemAdminLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const supabase = await createServerClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) redirect("/login");

  // System Admin only — all other roles are redirected
  const role = user.app_metadata?.role;
  if (role !== "system_admin") redirect("/");

  return <SystemAdminShell user={user}>{children}</SystemAdminShell>;
}
```

> **Note:** System Admin accounts are **seeded directly in the database** — there is no self-registration flow. The `(system-admin)` route group is entirely invisible to officers and students.

---

## 3. React Server Components vs Client Components

### Default to RSC

Every component in Next.js App Router is a **React Server Component by default**. Only add `"use client"` when the component explicitly requires browser APIs or interactivity.

### When to Use Each

| Use RSC When | Use Client Component When |
|-------------|--------------------------|
| Fetching data for initial page render | The component has `useState`, `useEffect`, `useRef` |
| Rendering static or read-only content | The component responds to user events (onClick, onSubmit, onChange) |
| Accessing server-only resources (env vars, DB) | The component uses React Query (`useQuery`, `useMutation`) |
| Rendering ShadCN components that need no interactivity | The component uses browser APIs (localStorage, window, navigator) |
| Displaying data passed as props from a page | The component uses ShadCN interactive primitives (Dialog, Select, Popover) |

### Composition Pattern: RSC Parent → Client Child

The standard pattern in VERIS is an RSC page that fetches data. It then passes that data as props to a Client Component that handles interactivity.

```
┌─────────────────────────────────────────────────┐
│  page.tsx (RSC)                                 │
│  • Fetches data server-side                     │
│  • Passes data as props                         │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │  FeatureComponent.tsx ("use client")      │  │
│  │  • Receives initialData as prop           │  │
│  │  • Uses React Query for live updates      │  │
│  │  • Handles user interactions              │  │
│  │  • Composes ShadCN primitives             │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## 4. Server Actions — Detailed Patterns

### File Organization

Server Actions live in their feature's `actions/` folder. One action per file. Each file exports a single named function.

```
src/features/attendance/
├── actions/
│   ├── createEvent.ts            # Creates a new event
│   ├── checkInStudent.ts         # Records attendance check-in
│   ├── checkOutStudent.ts        # Records attendance check-out
│   └── archiveEvent.ts           # Archives a completed event
├── components/
├── hooks/
└── types/
```

### Action Structure Template

Every Server Action follows this exact structure:

```typescript
// src/features/[domain]/actions/[actionName].ts
"use server";

import { z } from "zod";
import { createServerClient } from "@/lib/supabase/server";
import { revalidatePath } from "next/cache";
import type { ActionResponse } from "@/types/action-response";

// 1. Define Zod schema for input validation
const InputSchema = z.object({
  // ... fields
});

// 2. Export a single named async function
export async function actionName(
  input: z.infer<typeof InputSchema>
): Promise<ActionResponse<ReturnType>> {
  // 3. Validate input
  const parsed = InputSchema.safeParse(input);
  if (!parsed.success) {
    return { error: "Invalid input.", details: parsed.error.flatten().fieldErrors };
  }

  // 4. Execute database operation — RLS enforces auth + org scoping
  const supabase = await createServerClient();
  const { data, error } = await supabase
    .from("table_name")
    .insert({ /* ... */ })
    .select()
    .single();

  // 5. Handle database errors
  if (error) {
    return { error: "Operation failed." };
  }

  // 6. Invalidate relevant caches
  revalidatePath("/relevant/path");

  // 7. Return success
  return { success: true, data };
}
```

### Calling Server Actions from Client Components

Use `useTransition` for non-form-based calls, or `useFormStatus` for form submissions:

```tsx
// src/features/attendance/components/CreateEventButton.tsx
"use client";

import { useTransition } from "react";
import { Button } from "@/components/ui/button";
import { createEvent } from "@/features/attendance/actions/createEvent";
import { useQueryClient } from "@tanstack/react-query";

export function CreateEventButton() {
  const [isPending, startTransition] = useTransition();
  const queryClient = useQueryClient();

  function handleCreate() {
    startTransition(async () => {
      const result = await createEvent({
        name: "General Assembly",
        eventDate: new Date().toISOString(),
      });

      if (result.success) {
        // Invalidate React Query cache for live components
        queryClient.invalidateQueries({ queryKey: ["events"] });
      }
    });
  }

  return (
    <Button onClick={handleCreate} disabled={isPending}>
      {isPending ? "Creating..." : "Create Event"}
    </Button>
  );
}
```

---

## 5. React Query — Detailed Patterns

### Hook Organization

React Query hooks live in their feature's `hooks/` folder. Each hook wraps one `useQuery` or `useMutation` call.

```
src/features/attendance/
├── hooks/
│   ├── useEventList.ts           # useQuery: fetches paginated events
│   ├── useEventDetails.ts        # useQuery: fetches single event + attendance
│   ├── useLiveAttendance.ts      # useQuery + Supabase Realtime: live check-in feed
│   └── useCheckInMutation.ts     # useMutation: wraps checkInStudent Server Action
```

### Query Hook Pattern

```typescript
// src/features/attendance/hooks/useEventList.ts
"use client";

import { useQuery } from "@tanstack/react-query";
import { createBrowserClient } from "@/lib/supabase/client";

export function useEventList() {
  const supabase = createBrowserClient();

  return useQuery({
    queryKey: ["events"],
    queryFn: async () => {
      const { data, error } = await supabase
        .from("events")
        .select("*")
        .order("event_date", { ascending: false });

      if (error) throw error;
      return data;
    },
    // Refetch every 30 seconds for near-real-time updates
    refetchInterval: 30_000,
  });
}
```

### Mutation Hook Pattern (Wrapping a Server Action)

```typescript
// src/features/attendance/hooks/useCheckInMutation.ts
"use client";

import { useMutation, useQueryClient } from "@tanstack/react-query";
import { checkInStudent } from "@/features/attendance/actions/checkInStudent";
import { toast } from "sonner";

export function useCheckInMutation(eventId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (studentId: string) =>
      checkInStudent({ eventId, studentId }),
    onSuccess: (result) => {
      if (result.success) {
        toast.success("Student checked in.");
        queryClient.invalidateQueries({ queryKey: ["events", eventId, "attendance"] });
      } else {
        toast.error(result.error);
      }
    },
    onError: () => {
      toast.error("Something went wrong.");
    },
  });
}
```

### React Query + Supabase Realtime

For truly live data (e.g., attendance check-in feed that updates instantly across devices):

```typescript
// src/features/attendance/hooks/useLiveAttendance.ts
"use client";

import { useEffect } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createBrowserClient } from "@/lib/supabase/client";

export function useLiveAttendance(eventId: string) {
  const supabase = createBrowserClient();
  const queryClient = useQueryClient();

  // Base query — fetches current attendance list
  const query = useQuery({
    queryKey: ["events", eventId, "attendance"],
    queryFn: async () => {
      const { data, error } = await supabase
        .from("attendance_records")
        .select("*, students(first_name, last_name, id_number)")
        .eq("event_id", eventId)
        .order("checked_in_at", { ascending: false });

      if (error) throw error;
      return data;
    },
  });

  // Realtime subscription — invalidates query on INSERT
  useEffect(() => {
    const channel = supabase
      .channel(`attendance:${eventId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "attendance_records",
          filter: `event_id=eq.${eventId}`,
        },
        () => {
          queryClient.invalidateQueries({
            queryKey: ["events", eventId, "attendance"],
          });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [eventId, supabase, queryClient]);

  return query;
}
```

---

## 6. ShadCN Composition — Practical Examples

This section shows how ShadCN primitives from `src/components/ui/` are composed into feature-specific components.

### Example: Shared DataTable

A composed component that lives in `src/components/shared/` because it's used by attendance, members, financials, and clearance (4+ features → Rule of Three satisfied).

```tsx
// src/components/shared/DataTable.tsx
"use client";

import {
  ColumnDef,
  flexRender,
  getCoreRowModel,
  getPaginationRowModel,
  getSortedRowModel,
  SortingState,
  useReactTable,
} from "@tanstack/react-table";
import { useState } from "react";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { Button } from "@/components/ui/button";

interface DataTableProps<TData, TValue> {
  columns: ColumnDef<TData, TValue>[];
  data: TData[];
}

export function DataTable<TData, TValue>({
  columns,
  data,
}: DataTableProps<TData, TValue>) {
  const [sorting, setSorting] = useState<SortingState>([]);

  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getSortedRowModel: getSortedRowModel(),
    onSortingChange: setSorting,
    state: { sorting },
  });

  return (
    <div>
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((headerGroup) => (
              <TableRow key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <TableHead key={header.id}>
                    {header.isPlaceholder
                      ? null
                      : flexRender(header.column.columnDef.header, header.getContext())}
                  </TableHead>
                ))}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            {table.getRowModel().rows.length ? (
              table.getRowModel().rows.map((row) => (
                <TableRow key={row.id}>
                  {row.getVisibleCells().map((cell) => (
                    <TableCell key={cell.id}>
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </TableCell>
                  ))}
                </TableRow>
              ))
            ) : (
              <TableRow>
                <TableCell colSpan={columns.length} className="h-24 text-center">
                  No results.
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </div>
      <div className="flex items-center justify-end space-x-2 py-4">
        <Button
          variant="outline"
          size="sm"
          onClick={() => table.previousPage()}
          disabled={!table.getCanPreviousPage()}
        >
          Previous
        </Button>
        <Button
          variant="outline"
          size="sm"
          onClick={() => table.nextPage()}
          disabled={!table.getCanNextPage()}
        >
          Next
        </Button>
      </div>
    </div>
  );
}
```

### Example: Feature-Specific Attendance Table

This component lives in `src/features/attendance/components/` — it composes the shared `DataTable` with attendance-specific columns.

```tsx
// src/features/attendance/components/AttendanceTable.tsx
"use client";

import { ColumnDef } from "@tanstack/react-table";
import { DataTable } from "@/components/shared/DataTable";
import { Badge } from "@/components/ui/badge";
import type { AttendanceRecord } from "@/features/attendance/types/attendance.types";

const columns: ColumnDef<AttendanceRecord>[] = [
  {
    accessorKey: "students.id_number",
    header: "Student ID",
  },
  {
    accessorFn: (row) => `${row.students.last_name}, ${row.students.first_name}`,
    header: "Name",
  },
  {
    accessorKey: "checked_in_at",
    header: "Time In",
    cell: ({ row }) => new Date(row.getValue("checked_in_at")).toLocaleTimeString(),
  },
  {
    accessorKey: "checked_out_at",
    header: "Time Out",
    cell: ({ row }) => {
      const value = row.getValue("checked_out_at");
      return value ? new Date(value as string).toLocaleTimeString() : (
        <Badge variant="secondary">Active</Badge>
      );
    },
  },
];

interface AttendanceTableProps {
  records: AttendanceRecord[];
}

export function AttendanceTable({ records }: AttendanceTableProps) {
  return <DataTable columns={columns} data={records} />;
}
```

### Example: Feature-Specific Check-In Form

```tsx
// src/features/attendance/components/CheckInForm.tsx
"use client";

import { useState } from "react";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { useCheckInMutation } from "@/features/attendance/hooks/useCheckInMutation";

interface CheckInFormProps {
  eventId: string;
}

export function CheckInForm({ eventId }: CheckInFormProps) {
  const [search, setSearch] = useState("");
  const checkIn = useCheckInMutation(eventId);

  function handleCheckIn(studentId: string) {
    checkIn.mutate(studentId);
    setSearch("");
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle>Check-In</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <Input
          placeholder="Search by Student ID or Name..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
        />
        {/* Search results would render here as a list of matches.
            Each result item calls handleCheckIn(studentId) on click. */}
      </CardContent>
    </Card>
  );
}
```

---

## 7. Data Flow in Practice

Every interactive page in VERIS follows the same four-step data lifecycle. Specific implementation details for each feature domain (attendance, financials, clearance, etc.) are documented in their respective feature guides referenced from [06-implementation-roadmap.md](./06-implementation-roadmap.md).

### The Four-Step Lifecycle

| Step | Layer | What Happens | Pattern Used |
|------|-------|-------------|-------------|
| **1. Initial Load** | Server (RSC) | `page.tsx` fetches data via Supabase server client. RLS auto-filters by org. | React Server Component |
| **2. Client Hydration** | Client | Client Component receives `initialData` as props. React Query takes over for live updates and Supabase Realtime subscriptions. | `useQuery` with `initialData` |
| **3. Mutation** | Client → Server | User action triggers `useMutation` → calls a Server Action → Supabase insert/update. RLS enforces auth. DB triggers fire side effects. | `useMutation` → Server Action |
| **4. Live Update** | Server → Client | Supabase Realtime pushes change events to all subscribed clients. React Query `invalidateQueries` triggers refetch. | Supabase Realtime → `invalidateQueries` |

### General Pattern (Pseudocode)

```tsx
// Step 1: page.tsx (RSC) — initial server-side fetch
export default async function FeaturePage() {
  const supabase = await createServerClient();
  const { data } = await supabase.from("table").select("*"); // RLS scopes automatically
  return <FeatureView initialData={data ?? []} />;
}
```

```tsx
// Steps 2-4: FeatureView.tsx ("use client") — hydration, mutation, live update
"use client";

function FeatureView({ initialData }: { initialData: Item[] }) {
  const supabase = createBrowserClient();
  const queryClient = useQueryClient();

  // Step 2: React Query hydrates with server data, then manages live state
  const { data } = useQuery({
    queryKey: ["items"],
    queryFn: () => supabase.from("table").select("*").then(({ data }) => data),
    initialData,
  });

  // Step 3: Mutations go through Server Actions
  const mutation = useMutation({
    mutationFn: createItem, // Server Action
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["items"] }),
  });

  // Step 4: Realtime subscription invalidates cache on external changes
  useEffect(() => {
    const channel = supabase
      .channel("items")
      .on("postgres_changes", { event: "*", schema: "public", table: "table" }, () => {
        queryClient.invalidateQueries({ queryKey: ["items"] });
      })
      .subscribe();
    return () => { supabase.removeChannel(channel); };
  }, [supabase, queryClient]);

  return (/* render data + mutation triggers */);
}
```

> **Note:** Detailed data flow walkthroughs for specific features (attendance check-in, payment verification, fine generation, student onboarding, etc.) are documented in the phase-specific implementation guides.

---

## 8. Error Boundaries & Loading States

### File Convention per Route Segment

Next.js App Router supports co-located `loading.tsx`, `error.tsx`, and `not-found.tsx` files at each route segment:

```
src/app/(dashboard)/attendance/
├── page.tsx                    # Route content
├── loading.tsx                 # Shown while page.tsx is streaming
├── error.tsx                   # Catches runtime errors in this segment
└── not-found.tsx               # Shown when notFound() is called
```

### Loading State (Skeleton)

```tsx
// src/app/(dashboard)/attendance/loading.tsx
import { Skeleton } from "@/components/ui/skeleton";
import { Card, CardContent, CardHeader } from "@/components/ui/card";

export default function AttendanceLoading() {
  return (
    <div className="space-y-4">
      <Skeleton className="h-8 w-48" />
      <Card>
        <CardHeader>
          <Skeleton className="h-6 w-32" />
        </CardHeader>
        <CardContent className="space-y-2">
          {Array.from({ length: 5 }).map((_, i) => (
            <Skeleton key={i} className="h-12 w-full" />
          ))}
        </CardContent>
      </Card>
    </div>
  );
}
```

### Error Boundary

```tsx
// src/app/(dashboard)/attendance/error.tsx
"use client"; // Error boundaries must be Client Components

import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function AttendanceError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <Card className="mx-auto max-w-md mt-12">
      <CardHeader>
        <CardTitle>Something went wrong</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <p className="text-sm text-muted-foreground">
          {error.message || "Failed to load attendance data."}
        </p>
        <Button onClick={reset} variant="outline">
          Try Again
        </Button>
      </CardContent>
    </Card>
  );
}
```

---

## 9. Supabase Client Initialization

VERIS maintains **two Supabase client factories** — one for server contexts and one for browser contexts. These live in `src/lib/supabase/`.

### Server Client (RSC + Server Actions)

```typescript
// src/lib/supabase/server.ts
import { createServerClient as _createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createServerClient() {
  const cookieStore = await cookies();

  return _createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Ignored in RSC — cookies can only be set in Server Actions or Route Handlers
          }
        },
      },
    }
  );
}
```

### Browser Client (Client Components + React Query)

```typescript
// src/lib/supabase/client.ts
import { createBrowserClient as _createBrowserClient } from "@supabase/ssr";

export function createBrowserClient() {
  return _createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### React Query Provider

```tsx
// src/components/providers/QueryProvider.tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

---

## 10. Feature Folder Deep Dive: Attendance

To make the folder structure concrete, here is the **complete file tree** for the Attendance feature module:

```
src/features/attendance/
│
├── components/                        # All UI for the attendance domain
│   ├── EventList.tsx                  # Client: event cards grid with search/filter
│   ├── EventCard.tsx                  # Client: single event card (composes ui/card)
│   ├── EventAttendanceView.tsx        # Client: full event page (table + check-in form)
│   ├── AttendanceTable.tsx            # Client: composes shared/DataTable
│   ├── CheckInForm.tsx                # Client: student search + check-in
│   ├── CheckInSearchResults.tsx       # Client: search result list for check-in
│   ├── CreateEventDialog.tsx          # Client: composes ui/dialog for event creation
│   └── AttendanceStats.tsx            # Client: summary cards (total, present, absent)
│
├── hooks/                             # React Query hooks
│   ├── useEventList.ts                # useQuery: paginated event list
│   ├── useEventDetails.ts             # useQuery: single event metadata
│   ├── useLiveAttendance.ts           # useQuery + Realtime: live attendance feed
│   ├── useCheckInMutation.ts          # useMutation: wraps checkInStudent action
│   ├── useCheckOutMutation.ts         # useMutation: wraps checkOutStudent action
│   └── useStudentSearch.ts            # useQuery: debounced student search for check-in
│
├── actions/                           # Server Actions
│   ├── createEvent.ts                 # Creates a new event
│   ├── updateEvent.ts                 # Updates event details
│   ├── archiveEvent.ts                # Archives completed event
│   ├── checkInStudent.ts              # Records time-in
│   └── checkOutStudent.ts             # Records time-out
│
├── types/                             # TypeScript types
│   └── attendance.types.ts            # Event, AttendanceRecord, CheckInPayload, etc.
│
└── utils/                             # Feature-specific helpers
    ├── formatAttendanceTime.ts        # Time display formatting
    └── attendanceStatusHelpers.ts     # Status derivation (present, late, absent)
```

> Other feature modules (members, financials, clearance, organizations, auth) follow the same internal structure. Their specific file trees will be documented in the corresponding phase implementation guides in [06-implementation-roadmap.md](./06-implementation-roadmap.md).

---

## 11. Summary: File Decision Matrix

When creating a new file, use this matrix to determine where it goes:

| What are you building? | Where does it go? | Import alias |
|------------------------|-------------------|-------------|
| A ShadCN primitive (via CLI) | `src/components/ui/` | `@/components/ui/button` |
| A layout shell (Sidebar, Header) | `src/components/layout/` | `@/components/layout/DashboardShell` |
| A composed component used by 3+ features | `src/components/shared/` | `@/components/shared/DataTable` |
| A feature-specific component | `src/features/[domain]/components/` | `@/features/attendance/components/EventCard` |
| A React Query hook | `src/features/[domain]/hooks/` | `@/features/attendance/hooks/useEventList` |
| A Server Action | `src/features/[domain]/actions/` | `@/features/attendance/actions/checkInStudent` |
| TypeScript types for one feature | `src/features/[domain]/types/` | `@/features/attendance/types/attendance.types` |
| Global database/enum types | `src/types/` | `@/types/action-response` |
| Supabase/Firebase client setup | `src/lib/supabase/` or `src/lib/firebase/` | `@/lib/supabase/server` |
| A truly generic utility (`cn()`, `formatDate()`) | `src/lib/utils.ts` | `@/lib/utils` |
| A route page | `src/app/(group)/[route]/page.tsx` | — (not imported, it's a route) |
| A loading/error boundary | `src/app/(group)/[route]/loading.tsx` | — (not imported, it's a convention file) |

---

## 12. Related Documents

| Document | When to Read |
|----------|-------------|
| [01-system-overview.md](./01-system-overview.md) | System context, tiers, high-level architecture |
| [02-architecture-principles.md](./02-architecture-principles.md) | Core constraints (Feature-Driven, Thick DB, ShadCN strategy) |
| [04-supabase-firebase-auth.md](./04-supabase-firebase-auth.md) | RLS policies, RBAC, Firebase upload flow, student verification |
| [05-database-schema.md](./05-database-schema.md) | SQL table definitions referenced in code examples |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | Phase-specific feature domain build guides |
