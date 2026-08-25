---
name: modern-best-practice-nextjs
description: Build modern Next.js 16 apps with App Router, Server Components, server actions, and best practices. Use when creating pages, layouts, route groups, data fetching, metadata, error handling, or loading states in the Next.js frontend.
---

Next.js is a library that changes frequently — use web search for exploring the current documentation when unsure about APIs.

## Scope

**In scope:** App Router pages, layouts, route groups, Server Components, Client Components, server actions, metadata, loading states, error boundaries, streaming, data fetching patterns, and Next.js-specific configurations.

**Out of scope:** Backend code, database design, API contract design, component library design — defer these to the relevant agents.

## TypeScript Only

- **NEVER** write `.js` or `.jsx` files — TypeScript only
- Routes, layouts, and components use `.tsx`; hooks, stores, utils, config, and types use `.ts`
- This includes config files: `next.config.ts`, `middleware.ts`, `tailwind.config.ts`
- Type route props explicitly — in Next.js 16 `params` and `searchParams` are Promises:

```tsx
interface StreakPageProps {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ tab?: string }>;
}

const StreakPage = async ({ params, searchParams }: StreakPageProps) => {
  const { id } = await params;
  const { tab } = await searchParams;
  // ...
};

export default StreakPage;
```

- **PREFER** arrow functions assigned to a named `const`, default-exported on a separate line — this applies to pages, layouts, `loading.tsx`, `error.tsx`, and `not-found.tsx` alike
- **NEVER** `export default` an anonymous arrow — the component name is what appears in stack traces and React DevTools

- **AVOID** `any`; prefer `unknown` with narrowing
- **DO NOT** use `// @ts-ignore` — fix the type or use a documented `@ts-expect-error`

## Routing & Structure

### Route Groups

The project uses two route groups:

```
src/app/
├── layout.tsx                # Root layout
├── (marketing)/              # Public landing pages
│   └── page.tsx
└── (app)/                    # Protected app routes
    ├── layout.tsx
    ├── dashboard/
    │   └── page.tsx
    └── auth/
        ├── login/
        │   └── page.tsx
        └── register/
            └── page.tsx
```

- `(marketing)/` — Public-facing pages (landing, about, etc.)
- `(app)/` — Protected application pages (dashboard, streaks, profile, etc.)
- Route groups do not affect the URL — `(app)/dashboard/page.tsx` renders at `/dashboard`

### Page Pattern

```tsx
// src/app/(app)/dashboard/page.tsx
const DashboardPage = () => {
  return (
    <main className="min-h-screen p-6">
      <h1 className="text-3xl font-bold">Dashboard</h1>
    </main>
  );
};

export default DashboardPage;
```

### Layout Pattern

```tsx
// src/app/(app)/layout.tsx
const AppLayout = ({ children }: { children: React.ReactNode }) => {
  return (
    <div className="flex min-h-screen">
      <nav className="w-64 border-r">{/* Sidebar */}</nav>
      <div className="flex-1">{children}</div>
    </div>
  );
};

export default AppLayout;
```

## Server Components vs Client Components

- **Default to Server Components** — they render on the server, reduce JS bundle, and can directly access server resources
- Add `"use client"` directive **only** when the component needs:
  - React hooks (`useState`, `useEffect`, `useRef`, etc.)
  - Event handlers (`onClick`, `onChange`, etc.)
  - Browser APIs (`localStorage`, `window`, etc.)
  - TanStack Query hooks
  - Zustand store access

```tsx
// Server Component (default) — no directive needed
const StreakList = ({ streaks }: { streaks: Streak[] }) => {
  return (
    <ul>
      {streaks.map((streak) => (
        <li key={streak._id}>{streak.title}</li>
      ))}
    </ul>
  );
};

export default StreakList;

// Client Component — needs interactivity
"use client";

import { useState } from "react";

const StreakFilter = () => {
  const [selected, setSelected] = useState<string | null>(null);
  // ...
};

export default StreakFilter;
```

## Smart Pages, Dumb Components

The App Router maps naturally onto the container/presentational split — see the [React components skill](../modern-best-practice-react-components/SKILL.md) for the full pattern.

| | Route file (smart) | Component (dumb) |
| --- | --- | --- |
| Lives in | `app/**/page.tsx`, `layout.tsx` | `components/<category>/` |
| Does | Await params, fetch data, check auth, set metadata | Render props |
| Knows about | Routing, session, data sources | Nothing outside its props |

Rules:

- **KEEP** `page.tsx` thin — resolve params, fetch, delegate to a presentational component
- **DO NOT** put markup-heavy JSX directly in `page.tsx`; extract it to `components/`
- Dumb components take no `params`, call no `useRouter`, read no stores — pass values and callbacks down
- **PUSH** `"use client"` to the leaf — a client boundary on a layout makes the whole subtree client-side
- When a page needs client interactivity, keep the page a Server Component and render a small client container inside it

```tsx
// src/app/(app)/streaks/page.tsx — smart, Server Component
import StreakList from "@/components/streaks/streak-list";
import { getStreaks } from "@/lib/api/streaks";

export const metadata: Metadata = { title: "Streaks | App" };

const StreaksPage = async () => {
  const streaks = await getStreaks();

  return (
    <main className="p-6">
      <h1 className="text-3xl font-bold">Streaks</h1>
      <StreakList streaks={streaks} />
    </main>
  );
};

export default StreaksPage;
```

## Hooks & Logic Live in Separate Files

- **NEVER** define custom hooks inside a route or component file — extract and import them
- Hooks go in `src/hooks/use<Name>.ts`; server-side data helpers in `src/lib/api/`
- Server actions go in their own `"use server"` file (`src/actions/<domain>.ts`) — never inline in `page.tsx`
- Route files should contain: metadata, param resolution, data fetching calls, and composition — nothing else

```ts
// src/actions/streaks.ts
"use server";

import { revalidatePath } from "next/cache";

export const createStreak = async (data: CreateStreakInput): Promise<void> => {
  // ...
  revalidatePath("/streaks");
};
```

- Server actions must be `async` and exported by name — an arrow assigned to an exported `const` satisfies both

## Performance: Code Splitting

- Next.js splits per route automatically — the biggest win is **keeping `"use client"` at the leaf** so routes stay server-rendered
- **USE** `next/dynamic` for heavy or conditionally rendered client components (charts, editors, modals, date pickers)
- **SET** `ssr: false` only for components that touch `window`; note it is not allowed in Server Components
- **PROVIDE** a `loading` fallback matching the final layout to avoid layout shift
- **USE** `loading.tsx` + `<Suspense>` to stream slow segments instead of blocking the whole route
- **AVOID** splitting small components — request overhead outweighs the savings
- Import from exact paths, not barrel files — barrels defeat tree shaking

```tsx
"use client";

import dynamic from "next/dynamic";

const StreakHeatmap = dynamic(
  () => import("@/components/streaks/streak-heatmap"),
  {
    loading: () => <div className="h-48 animate-pulse rounded-lg bg-gray-100" />,
    ssr: false,
  },
);
```

Split at the library level too — import heavy dependencies inside the handler that needs them:

```ts
const handleExport = async (): Promise<void> => {
  const { generatePdf } = await import("@/lib/pdf");
  await generatePdf(streaks);
};
```

## Data Fetching & Caching

### Server Components — Fetch data directly

```tsx
// Server Component — fetches on the server
const StreaksPage = async () => {
  const res = await fetch(`${process.env.BACKEND_API_BASE_URL}/streaks`, {
    cache: "no-store",
  });
  const data = await res.json();

  return <StreakList streaks={data.data} />;
};

export default StreaksPage;
```

### Client Components — Use TanStack Query

```tsx
"use client";

import { useQuery } from "@tanstack/react-query";
import axiosInstance from "@/lib/axiosInstance";

const StreakListClient = () => {
  const { data, isLoading, error } = useQuery({
    queryKey: ["streaks"],
    queryFn: async () => {
      const { data } = await axiosInstance.get("/streaks");
      return data.data;
    },
  });

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error loading streaks</p>;

  return <StreakList streaks={data} />;
};

export default StreakListClient;
```

## UI States

### Loading States

```tsx
// src/app/(app)/dashboard/loading.tsx
const DashboardLoading = () => {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <p>Loading...</p>
    </div>
  );
};

export default DashboardLoading;
```

### Error Boundaries

```tsx
"use client";

// src/app/(app)/dashboard/error.tsx
interface DashboardErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

const DashboardError = ({ error, reset }: DashboardErrorProps) => {
  return (
    <div className="flex min-h-screen flex-col items-center justify-center">
      <h2 className="text-xl text-red-500">Something went wrong</h2>
      <button onClick={reset} className="mt-4 rounded-lg px-4 py-2">
        Try again
      </button>
    </div>
  );
};

export default DashboardError;
```

### Suspense Boundaries

```tsx
import { Suspense } from "react";

const DashboardPage = () => {
  return (
    <main className="p-6">
      <Suspense fallback={<p>Loading streaks...</p>}>
        <StreakList />
      </Suspense>
    </main>
  );
};

export default DashboardPage;
```

## Metadata & SEO

```tsx
// Static metadata
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Dashboard | App",
  description: "Your app dashboard",
};

// Dynamic metadata — `params` is a Promise in Next.js 16
export const generateMetadata = async ({
  params,
}: {
  params: Promise<{ id: string }>;
}): Promise<Metadata> => {
  const { id } = await params;
  const item = await fetchItem(id);

  return {
    title: `${item.title} | App`,
    description: item.description ?? "View this item",
  };
};
```

- `generateMetadata` must be exported by name — an exported `const` arrow qualifies
- **ALWAYS** `await params` here too; it is a Promise in Next.js 16 exactly as it is in the page component
- **PREFER** `??` over `||` for fallbacks — `||` also replaces a legitimate empty string

## Existing Stack Context

- **Next.js 16** (App Router), **React 19**
- **Tailwind CSS 4** via `@tailwindcss/postcss`
- **Zustand 5** (localStorage persistence for auth)
- **TanStack React Query 5** (staleTime: 60s, retry: 3, no refetch on window focus)
- **Axios** via `axiosInstance` (withCredentials: true)
- **React Hook Form 7** + Zod
- **Socket.io Client** (real-time updates)
- **TypeScript** strict mode, path alias `@/*` → `./src/*`

## Build & Dev Commands

```bash
bun install              # Install dependencies
bun run dev              # Next.js dev server (port 3000)
bun run build            # Production build
bun run lint             # ESLint
```

## Constraints

- **Default** to Server Components — add `"use client"` only when necessary
- **AVOID** fetching via `useEffect()` and `fetch()` — use TanStack Query in Client Components or fetch directly in Server Components
- **ALWAYS** provide `loading.tsx` and `error.tsx` for route segments
- **ALWAYS** use `axiosInstance` for client-side API calls — never raw `fetch()` or `axios`
- **Follow** the existing route group pattern: `(marketing)/` for public, `(app)/` for protected
- **Do not** modify backend code
- **Do not** make architectural decisions — defer to architect agents
- **Do not** write JavaScript — `.tsx` for routes and components, `.ts` for everything else
- **Do not** define hooks, server actions, or business logic inline in a route file
- **Keep** `page.tsx` thin — fetch and delegate; markup belongs in `components/`
- **Keep** `"use client"` at the leaf, never on a layout that wraps server content
