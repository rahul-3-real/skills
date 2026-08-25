---
name: modern-best-practice-react-components
description: Build clean, modern React 19 components that apply common best practices and avoid common pitfalls like unnecessary state management or useEffect usage. Use when creating or modifying components, hooks, event handlers, or component composition in the Next.js 16 frontend.
---

All React components must follow modern best practices focused on clarity, correctness, and maintainability.

## Scope

**In scope:** React component structure, state management patterns, event handling, props and composition, rendering and derivation, performance, hooks usage, and component organization.

**Out of scope:** Backend code, API design, database design, routing decisions (defer to Next.js skill), styling decisions (defer to Tailwind skill).

## TypeScript Only

- **NEVER** write `.js` or `.jsx` files — TypeScript only
- Components use `.tsx`; hooks, stores, utils, and types use `.ts`
- **AVOID** `any`; prefer `unknown` with narrowing, or a precise type
- Type all props, hook returns, and API payloads explicitly
- **DO NOT** use `// @ts-ignore` — fix the type or use a documented `@ts-expect-error`

## Component Structure & Style

- **PREFER** small, focused components with a single responsibility
- **PREFER** arrow function components assigned to a named `const`, then default-exported on a separate line
  - **NEVER** default-export an anonymous arrow (`export default () => ...`) — it breaks DevTools names, stack traces, and Fast Refresh
  - Use a `function` declaration only where hoisting or a `this` binding is genuinely required — in React components, it never is
- **PREFER** explicit return types and props typing (TypeScript) where applicable
- Keep components flat and readable; avoid deeply nested JSX
- Group related logic together (event handlers, derived values, helpers)

### Component Pattern

```tsx
"use client";

interface StreakCardProps {
  title: string;
  description?: string;
  color: string;
  currentStreak: number;
  onClick: () => void;
}

const StreakCard = ({
  title,
  description,
  color,
  currentStreak,
  onClick,
}: StreakCardProps) => {
  return (
    <article
      onClick={onClick}
      className="cursor-pointer rounded-lg border p-4 transition-colors hover:shadow-md"
    >
      <div className="flex items-center gap-2">
        <span
          className="h-3 w-3 rounded-full"
          style={{ backgroundColor: color }}
        />
        <h3 className="font-semibold">{title}</h3>
      </div>
      {description && (
        <p className="mt-1 text-sm text-gray-500">{description}</p>
      )}
      <p className="mt-2 text-2xl font-bold">{currentStreak} days</p>
    </article>
  );
};

export default StreakCard;
```

Key rules:

- Props defined as TypeScript interfaces inline or in `types/`
- Always include accessibility attributes (`aria-*`, `role`, semantic HTML)
- Shared components go in `components/<category>/`

## Smart & Dumb Components

Split components by responsibility: **smart** (container) components own data and behavior; **dumb** (presentational) components own markup only.

| | Smart / Container | Dumb / Presentational |
| --- | --- | --- |
| Knows about | Data sources, stores, mutations | Props only |
| Contains | Hooks, queries, handlers | JSX + styling |
| Reusable | Rarely | Highly |
| Testable | Via integration tests | Via props, trivially |
| Location | `app/**` or `components/<domain>/` | `components/ui/`, `components/<category>/` |

Rules:

- **DO NOT** call `useQuery`, `useMutation`, or Zustand stores inside a dumb component
- Dumb components receive data and callbacks via props — no data fetching, no routing, no global state
- A dumb component should render identically for the same props, every time
- Keep the smart component thin: fetch, derive, pass down
- **PREFER** Server Components as the container when no interactivity is needed

```tsx
// components/streaks/streak-list.tsx — dumb
interface StreakListProps {
  streaks: Streak[];
  onSelect: (id: string) => void;
}

const StreakList = ({ streaks, onSelect }: StreakListProps) => {
  return (
    <ul className="grid gap-4 sm:grid-cols-2">
      {streaks.map((streak) => (
        <li key={streak.id}>
          <StreakCard {...streak} onClick={() => onSelect(streak.id)} />
        </li>
      ))}
    </ul>
  );
};

export default StreakList;
```

```tsx
// app/streaks/streak-list-container.tsx — smart
"use client";

import { useRouter } from "next/navigation";
import { useStreaks } from "@/hooks/useStreaks";
import StreakList from "@/components/streaks/streak-list";

const StreakListContainer = () => {
  const router = useRouter();
  const { data: streaks, isPending, isError } = useStreaks();

  const handleSelect = (id: string) => {
    router.push(`/streaks/${id}`);
  };

  if (isPending) return <StreakListSkeleton />;
  if (isError) return <ErrorState message="Could not load streaks" />;

  return <StreakList streaks={streaks} onSelect={handleSelect} />;
};

export default StreakListContainer;
```

## Hooks & Logic Live in Separate Files

- **NEVER** define custom hooks inside a component file — extract and import them
- Hooks go in `src/hooks/use<Name>.ts` (one hook or one cohesive group per file)
- Pure helpers, formatters, and validators go in `src/lib/` or `src/utils/`
- Zod schemas and shared types go in `src/schemas/` and `src/types/`
- Component files should contain: imports, props interface, the component, its handlers — nothing else
- If a component file grows a `useEffect`, a query, or a reducer, that logic belongs in a hook

```ts
// src/hooks/useStreakForm.ts
export const useStreakForm = () => {
  const { mutateAsync, isPending } = useCreateStreak();
  const form = useForm<CreateStreakFormData>({
    resolver: zodResolver(createStreakSchema),
  });

  const submit = async (data: CreateStreakFormData): Promise<void> => {
    await mutateAsync(data);
    form.reset();
  };

  return { form, submit, isPending };
};
```

```tsx
// src/components/streaks/create-streak-form.tsx
import { useStreakForm } from "@/hooks/useStreakForm";

const CreateStreakForm = () => {
  const { form, submit, isPending } = useStreakForm();
  // render only
};

export default CreateStreakForm;
```

## Performance: Code Splitting

Ship less JavaScript on first load. Split anything that is heavy, below the fold, or conditionally rendered.

- **PREFER** route-level splitting first — Next.js App Router splits per route automatically; keep `"use client"` at the leaf, not the layout
- **USE** `next/dynamic` for heavy or rarely-used client components (charts, editors, modals, date pickers)
- **SET** `ssr: false` only for components that touch `window` or have no server-rendered value
- **PROVIDE** a `loading` fallback that matches the final layout to avoid layout shift
- **LAZY-LOAD** below-the-fold and dialog content — it costs nothing until opened
- **AVOID** splitting small components; the request overhead outweighs the savings
- Prefer named dynamic imports over barrel files — barrels defeat tree shaking

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

const SettingsDialog = dynamic(
  () => import("@/components/settings/settings-dialog"),
);

const StreakDashboard = () => {
  const [isSettingsOpen, setIsSettingsOpen] = useState(false);

  return (
    <>
      <StreakHeatmap />
      {isSettingsOpen && (
        <SettingsDialog onClose={() => setIsSettingsOpen(false)} />
      )}
    </>
  );
};

export default StreakDashboard;
```

Also split at the library level — import heavy dependencies inside the handler that needs them:

```ts
const handleExport = async (): Promise<void> => {
  const { generatePdf } = await import("@/lib/pdf");
  await generatePdf(streaks);
};
```

## State Management

- **AVOID** `useEffect()`
  - See the ["You Might Not Need An Effect" guide](references/you-dont-need-useeffect.md) for detailed guidance
  - **PREFER** deriving values during render instead of synchronizing state
  - Fetch data via TanStack Query (`@tanstack/react-query`)
- **AVOID** unnecessary `useState()` or `useReducer()` usage
  - Derive state from props or other state when possible
  - Localize state to the lowest possible component
- **DO NOT** mirror props in state unless absolutely necessary
- Prefer controlled components over syncing uncontrolled state

### Data Fetching — TanStack Query

```tsx
"use client";

import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import axiosInstance from "@/lib/axiosInstance";

export const useStreaks = () =>
  useQuery({
    queryKey: ["streaks"],
    queryFn: async () => {
      const { data } = await axiosInstance.get("/streaks");
      return data.data;
    },
  });

export const useCreateStreak = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload: {
      title: string;
      description?: string;
      color?: string;
      icon?: string;
      goal?: string;
      buffer?: number;
    }) => {
      const { data } = await axiosInstance.post("/streaks", payload);
      return data.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["streaks"] });
    },
  });
};
```

### Zustand Store Pattern

```tsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface UserState {
  user: User | null;
  isAuthenticated: boolean;
  setUser: (user: User) => void;
  clearUser: () => void;
}

const useUserStore = create<UserState>()(
  persist(
    (set) => ({
      user: null,
      isAuthenticated: false,
      setUser: (user) => set({ user, isAuthenticated: true }),
      clearUser: () => set({ user: null, isAuthenticated: false }),
    }),
    { name: "user-store" },
  ),
);

export default useUserStore;
```

- Store files: `use<Name>Store.ts` in `src/store/`
- Use `persist` middleware for data that survives page refresh
- Keep stores isolated by domain — one store per concern

## Rendering & Derivation

- **PREFER** computing derived values inline or via helper functions
- Use `useMemo()` sparingly and only for proven performance issues
- **AVOID** premature optimization
- Keep render logic deterministic and free of side effects

## Event Handling

- **AVOID** inline event handlers in JSX
- **PREFER** named handler functions:

```tsx
const handleClick = () => {
  // ...
};

<button onClick={handleClick} />;
```

- Name handlers clearly (`handleSubmit`, `handleChange`, `handleClose`)
- Keep handlers small; extract complex logic into helpers

## Effects, Data, and Side Effects

- **AVOID** effects for:
  - Derived state
  - Data transformations
  - Event-based logic that can live in handlers
- If side effects are unavoidable, keep them minimal, isolated, and well-documented
- Prefer framework-level or external abstractions (routers, data libraries) over raw effects

## Props & Composition

- **PREFER** composition over configuration
- **AVOID** excessive boolean props; prefer expressive APIs
- Use `children` intentionally and document expected structure
- Keep prop names semantic and predictable

## Performance & Stability

- **PREFER** stable references only when required (not by default)
- **AVOID** unnecessary memoization (`memo`, `useCallback`) unless absolutely required
- Keep keys stable and meaningful when rendering lists

## Form Pattern (React Hook Form + Zod)

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const createStreakSchema = z.object({
  title: z.string().min(1, "Title is required"),
  description: z.string().max(500).optional(),
  color: z
    .string()
    .regex(/^#[0-9a-fA-F]{6}$/)
    .optional(),
});

type CreateStreakFormData = z.infer<typeof createStreakSchema>;

const CreateStreakForm = () => {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<CreateStreakFormData>({
    resolver: zodResolver(createStreakSchema),
  });

  const onSubmit = (data: CreateStreakFormData) => {
    // call mutation
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register("title")}
        className="rounded-lg border p-2"
        placeholder="Streak title"
      />
      {errors.title && (
        <span className="text-sm text-red-500">{errors.title.message}</span>
      )}
    </form>
  );
};

export default CreateStreakForm;
```

## Naming Conventions

- **Components:** PascalCase (e.g., `StreakCard`, `AuthLayout`)
- **Files:** kebab-case for multi-word names (e.g., `streak-card.tsx`)
- **Functions/variables:** camelCase (e.g., `fetchStreaks`, `isLoading`)
- **Stores:** `use<Name>Store.ts` (e.g., `useUserStore.ts`)
- **Hooks:** `use<Name>.ts` (e.g., `useStreaks.ts`)
- **Lib files:** camelCase descriptive (e.g., `axiosInstance.tsx`, `queryProvider.tsx`)

## Constraints

- **Follow** existing patterns — do not invent new patterns or restructure the codebase
- **Do not** modify backend code
- **Do not** make architectural decisions — defer to architect agents
- **Do not** use `fetch()` directly — always use `axiosInstance`
- **Do not** write JavaScript — `.tsx` for components, `.ts` for everything else
- **Do not** define hooks or business logic inline in a component file — extract to `hooks/` or `lib/`
- **Do not** fetch data or read stores inside presentational components
- **Always** define components, hooks, and handlers as arrow functions assigned to a named `const` — never a `function` declaration, never an anonymous default export
- **Always** use semantic HTML and include accessibility attributes
- **Always** mark client components with `"use client"` only when necessary
- **Prefer** Server Components by default

## General Principles

- Write code for humans first, compilers second
- Prefer explicitness over cleverness
- Optimize for readability and long-term maintenance
- If a pattern feels complex, reconsider the component boundary
