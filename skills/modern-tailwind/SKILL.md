---
name: modern-tailwind
description: Build clean, scalable UIs with Tailwind CSS 4 using modern utilities, variants, and the project's design tokens. Use when styling components, pages, layouts, or responsive designs in the Next.js 16 frontend.
---

All styling must use Tailwind CSS utility classes. When design tokens are defined, use them — never use hardcoded hex values for token-defined colors.

## Scope

**In scope:** Tailwind CSS styling, design tokens, responsive design, variants, theming, component styling patterns, class organization, dark mode (if applicable), and accessible styling.

**Out of scope:** Backend code, API design, database design, component logic (defer to React component skill), HTML structure (defer to accessible HTML skill).

## Design Tokens (Tailwind CSS 4)

Design tokens will be defined in the project's Tailwind CSS configuration using the `@theme` directive. When tokens are established, **always use them — never hardcoded hex values**.

**Note:** Design tokens will be defined as the UI is built out. Until then, use Tailwind's default palette sensibly and consistently. Once tokens are set, all code should migrate to use them.

### Usage Pattern

```tsx
// GOOD — uses Tailwind utilities consistently
<div className="bg-white text-gray-900 dark:bg-gray-900 dark:text-white">
  <h1 className="text-3xl font-bold">My Streaks</h1>
  <p className="text-sm text-gray-500">Track your daily habits</p>
  <button className="rounded-lg bg-teal-600 px-4 py-2 text-white hover:bg-teal-700">
    New Streak
  </button>
</div>

// BAD — hardcoded values
<div className="bg-[#1e1e1e] text-[#eaeaea]">
  <h1 className="text-[32px]">My Streaks</h1>
</div>
```

## Core Principles

- Prefer utility classes over custom CSS for all styling
- Keep class lists readable by grouping: **layout → spacing → typography → color → effects**
- Use semantic HTML first; utilities should enhance, not replace structure

### Class Order Convention

```tsx
<div className="
  flex items-center justify-between          {/* layout */}
  gap-4 p-6                                   {/* spacing */}
  text-sm font-medium                         {/* typography */}
  bg-white text-gray-900 border-gray-200      {/* color */}
  rounded-lg shadow-md transition-colors      {/* effects */}
">
```

## Variants & State

- Use `hover`, `focus-visible`, `disabled`, `dark`, and `motion-safe` variants where appropriate
- Prefer `data-*` and `aria-*` variants for stateful styling tied to DOM semantics
- Use `group` and `peer` for parent/sibling state without extra JS

```tsx
// Interactive button with proper states
<button className="
  rounded-lg bg-teal-600 px-4 py-2 text-white
  hover:bg-teal-700
  focus-visible:ring-2 focus-visible:ring-teal-500 focus-visible:outline-none
  disabled:cursor-not-allowed disabled:opacity-50
  transition-colors
">
  Complete Streak
</button>

// Group hover pattern
<div className="group flex items-center gap-2 rounded-lg p-3 hover:bg-gray-100">
  <span className="text-gray-700 group-hover:text-gray-900 transition-colors">Streak Title</span>
</div>
```

## Responsive Design

- Start with base (mobile) styles, then add responsive variants (`sm`, `md`, `lg`, `xl`, `2xl`)
- Use container query utilities when layout depends on parent size

```tsx
// Mobile-first responsive layout
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
  {streaks.map((streak) => (
    <article key={streak._id} className="rounded-lg border p-4">
      <h3 className="text-lg font-semibold">{streak.title}</h3>
    </article>
  ))}
</div>
```

## Theming & Customization

- Extend the theme via `@theme` in the Tailwind config CSS — not ad-hoc custom classes
- Use `@layer` for custom utilities/components when repetition is unavoidable
- **Avoid** `@apply` except for small, repeatable patterns

## Component Styling Patterns

### Card Pattern

```tsx
<article className="rounded-lg border border-gray-200 bg-white p-6 shadow-sm">
  <h3 className="text-xl font-semibold">{title}</h3>
  <p className="mt-2 text-sm text-gray-500">{description}</p>
</article>
```

### Input Pattern

```tsx
<input
  type="text"
  className="w-full rounded-lg border border-gray-300 p-3 text-gray-900 placeholder:text-gray-400 focus:border-teal-500 focus:outline-none"
  placeholder="Enter streak title..."
/>
```

### Badge Pattern

```tsx
<span className="inline-flex items-center rounded-full bg-teal-100 px-3 py-1 text-xs font-medium text-teal-700">
  Active
</span>
```

## Maintainability

- Extract reusable UI into React components instead of repeating large class strings
- Keep class names deterministic; avoid dynamic string concatenation when possible
- When conditional classes are needed, use template literals or a utility like `clsx`:

```tsx
<button
  className={`rounded-lg px-4 py-2 text-sm font-medium transition-colors ${
    isActive ? "bg-teal-600 text-white" : "bg-gray-100 text-gray-700"
  }`}
>
  {label}
</button>
```

## Constraints

- **NEVER** use hardcoded hex values when design tokens are defined — use token classes
- **PREFER** utility classes over custom CSS
- **AVOID** `@apply` except for small, repeatable patterns
- **Follow** mobile-first responsive design
- **Follow** existing styling patterns in `src/components/`
