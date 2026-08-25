---
name: bun-first
description: Describes efficient usage of Bun and Bun APIs instead of Node.js (and its APIs)
---

All code must follow a Bun-first approach. Bun is the JavaScript runtime, package manager, and task runner. Assume Bun is available unless explicitly stated otherwise.

## Scope

**In scope:** Using Bun as the runtime, package manager, script runner, and leveraging Bun's native APIs wherever possible across both backend and frontend.

**Out of scope:** Architecture decisions, database design, API design, frontend component design — defer these to the relevant agents.

## General Principles

- **PREFER** Bun over Node.js, npm, pnpm, or yarn in all contexts
- **PREFER** Bun's built-in features over third-party tools when available
- **PREFER** Bun's native APIs (e.g., for file access, SQL, S3, etc.) over Node.js APIs

## Package Management

```bash
bun install              # Install dependencies
bun add <package>        # Add a dependency
bun remove <package>     # Remove a dependency
bun add -d <package>     # Add a dev dependency
```

- **USE** `bun install`, `bun add`, `bun remove`
- **AVOID** `npm`, `yarn`, `pnpm`
- Lock file: `bun.lock` — never `package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml`
- Keep dependencies minimal and intentional

## Scripts & Tooling

### Backend (`cd apps/backend`)

```bash
bun install              # Install dependencies
bun run dev              # Dev server with watch (port 8000)
bun run build            # Compile TypeScript
bun run start            # Run production build
```

### Frontend (`cd apps/frontend`)

```bash
bun install              # Install dependencies
bun run dev              # Next.js dev server (port 3000)
bun run build            # Production build
bun run lint             # ESLint
```

- **PREFER** `bun run` for all scripts
- **USE** Bun's built-in test runner (`bun:test`) for backend tests — `describe`, `it`, `expect`, `mock`, `beforeAll`, `afterAll`, `beforeEach`, `afterEach`
- **AVOID** introducing extra task runners unless required

## Runtime & APIs

- **PREFER** Bun's native APIs (fetch, fs, path, env handling)
- Write code assuming modern Web APIs are available in the runtime
- Avoid Node-specific APIs unless explicitly required

## Build Setup

### Backend

- TypeScript strict mode, path alias: `@/*` → `./src/*`
- Hono handles routing natively — no Express-like build pipeline needed

### Frontend

- TypeScript strict mode, ESNext modules, bundler module resolution
- Path alias: `@/*` → `./src/*`
- Next.js 16 with App Router

## Performance & DX

- Prefer simple, explicit scripts over complex toolchains
- **AVOID** unnecessary abstractions
- Leverage Bun's fast startup and native performance where possible

## Constraints

- **Always** use `bun` commands — never `npm`, `yarn`, or `pnpm`
- **Always** use `bun.lock` — never other lock files
- **Do not** install dependencies without explicit instruction — recommend them instead
- **Follow** existing patterns in the codebase
