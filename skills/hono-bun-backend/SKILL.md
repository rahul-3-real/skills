---
name: hono-bun-backend
description: Build the Hono 4 backend running on Bun — routing, middleware, controllers, typed context, validation, error handling, and the ApiResponse/ApiError contract. Use when creating or modifying routes, controllers, middleware, or server configuration in the Hono backend. Defers schema, query, and migration work to the database skill.
---

The backend is Hono 4 running on Bun, with Zod at the request boundary and the data layer described by the `database` skill. Hono changes frequently — use web search for current APIs when unsure.

## Scope

**In scope:** Hono app setup, routing, middleware, controllers, typed context, request validation, error handling, response shape, server bootstrap, and how controllers call the data layer.

**Out of scope:** Schema design, indexing, queries, transactions, and migrations (see `database` — it routes to [NoSQL/Mongoose](../database/references/nosql-mongodb.md) or [SQL/Drizzle](../database/references/sql-postgres.md) based on the stack), frontend code, WebSocket events (see `socket-io`), TypeScript type rules (see `clean-typescript`), package management and Bun APIs (see `bun-first`), security policy (see `web-security`), general code style (see `clean-code`).

## Project Layout

```
app/backend/src/
├── index.ts                 # Bootstrap — connect DB, then Bun.serve
├── configs/
│   ├── app.config.ts        # Hono app: CORS, routes, onError
│   └── db.config.ts         # Database connection — see `database`
├── routes/                  # One Hono sub-app per resource
├── controllers/             # Request handlers
├── middlewares/             # auth, ownership, error
├── models/                  # Data layer — see `database`
├── services/                # Business logic that outgrows a controller
├── types/                   # Shared types
└── utils/                   # ApiError, ApiResponse, token helpers
```

**One resource per file, one layer per directory.** A route file wires middleware to controllers and nothing else. A controller validates, calls the data layer, and returns a response. Business logic that outgrows a controller moves to `src/services/`.

## Layering Rules

| Layer      | Responsibility                                        | Must NOT                             |
| ---------- | ----------------------------------------------------- | ------------------------------------ |
| Route      | Path → middleware chain → controller                  | Contain logic or touch the DB        |
| Middleware | Cross-cutting concerns (auth, ownership, errors)      | Return business data                 |
| Controller | Validate input, orchestrate, return `ApiResponse`     | Contain DB schema details            |
| Service    | Multi-step business logic, cross-resource writes      | Read `Context` or build responses    |
| Model      | Schema, indexes, hooks, queries (see `database`)      | Import controllers or read `Context` |
| Util       | Pure, reusable helpers                                | Depend on `Context` or models        |

**The database is reached only through the model/service layer.** A controller may call a model directly for a simple read or write; anything involving multiple documents, a transaction, or a computed result belongs in a service. Middleware touches the database only for auth and ownership lookups.

## App Configuration

```typescript
// src/configs/app.config.ts
import { Hono } from "hono";
import { cors } from "hono/cors";
import { errorHandler } from "@/middlewares/error.middleware";
import authRoutes from "@/routes/auth.routes";
import streakRoutes from "@/routes/streak.routes";

const app = new Hono();

app.use(
  "*",
  cors({
    origin: process.env.CLIENT_URL ?? "http://localhost:3000",
    credentials: true,
  })
);

app.route("/api/v1/auth", authRoutes);
app.route("/api/v1/streaks", streakRoutes);

app.onError(errorHandler);

export default app;
```

- **ALWAYS** version the API prefix — `/api/v1/...`
- **ALWAYS** register `app.onError(errorHandler)` last, after routes
- **PREFER** Hono's built-in middleware (`hono/cors`, `hono/cookie`, `hono/logger`) over third-party packages

## Server Bootstrap

```typescript
// src/index.ts
import connectDB from "@/configs/db.config";
import app from "@/configs/app.config";

const PORT = Number(process.env.PORT) || 8000;

try {
  await connectDB(); // top-level await — no .then() chain
  Bun.serve({ fetch: app.fetch, port: PORT });
  console.log(`Server running on http://localhost:${PORT}`);
} catch (error) {
  console.error("Failed to connect to the database:", error);
  process.exit(1);
}
```

- **ALWAYS** use top-level `await` in the bootstrap — Bun supports it natively; a `.then()` chain here mixes two async styles for no benefit
- **ALWAYS** use `Bun.serve` — never `@hono/node-server` unless a dependency demands a Node HTTP server
- **ALWAYS** connect to the DB before serving — fail fast with `process.exit(1)`
- **ALWAYS** keep the connection in `src/configs/db.config.ts` and open it exactly once — see `database` for engine-specific setup
- Bun loads `.env` automatically — **do not** import `dotenv`

## Typed Context

Untyped `c.get("user")` returns `any`, which defeats strict mode. Declare the context environment once and use it everywhere:

```typescript
// src/types/index.ts
import type { HydratedDocument } from "mongoose";

export interface AppEnv {
  Variables: {
    user: HydratedDocument<User>;
  };
}
```

```typescript
// Usage — routes, middleware, and controllers all take the same env
import { Hono } from "hono";
import type { Context, Next } from "hono";
import type { AppEnv } from "@/types";

const streaks = new Hono<AppEnv>();

export const isAuthenticated = async (c: Context<AppEnv>, next: Next) => {
  /* ... */
};

export const getStreaksController = async (c: Context<AppEnv>) => {
  const user = c.get("user"); // fully typed — no `any`
};
```

- **ALWAYS** type handlers as `Context<AppEnv>` — never bare `Context` when reading context variables
- **NEVER** widen a context variable to `any` to make an error go away

## Routes

```typescript
// src/routes/streak.routes.ts
import { Hono } from "hono";
import type { AppEnv } from "@/types";
import { isAuthenticated } from "@/middlewares/auth.middleware";
import { checkIsOwner } from "@/middlewares/ownership.middleware";
import {
  getStreaksController,
  createStreakController,
  updateStreakController,
  deleteStreakController,
} from "@/controllers/streak.controllers";

const streaks = new Hono<AppEnv>();

streaks.use("*", isAuthenticated);

streaks.get("/", getStreaksController);
streaks.post("/", createStreakController);
streaks.put("/:id", checkIsOwner("streak"), updateStreakController);
streaks.delete("/:id", checkIsOwner("streak"), deleteStreakController);

export default streaks;
```

- Route files are declarative — **NEVER** inline a handler body in a route file
- Apply blanket middleware with `use("*", ...)` before route definitions
- Order matters: static segments before dynamic ones (`/streaks/active` before `/streaks/:id`)
- Controller names end in `Controller`; files are kebab-case (`streak.controllers.ts`)

## Controllers

Every controller follows the same four steps: **read → validate → act → respond.**

```typescript
// src/controllers/streak.controllers.ts
import type { Context } from "hono";
import { z } from "zod";
import type { AppEnv } from "@/types";
import Streak from "@/models/streak.model";
import ApiError from "@/utils/apiError";
import ApiResponse from "@/utils/apiResponse";

const createStreakSchema = z.object({
  title: z.string().min(1, "Title is required").max(100),
  description: z.string().max(500).optional(),
  color: z.string().regex(/^#[0-9a-fA-F]{6}$/, "Invalid hex color"),
  icon: z.string().min(1, "Icon is required").max(50),
  goal: z.string().datetime("Invalid date format"),
  buffer: z.number().int().min(0).max(1440).optional(),
});

export const createStreakController = async (c: Context<AppEnv>) => {
  const body = await c.req.json();

  const result = createStreakSchema.safeParse(body);
  if (!result.success) {
    throw new ApiError(
      400,
      "Validation failed",
      result.error.issues.map((e) => e.message)
    );
  }

  const { title, description, color, icon, goal, buffer } = result.data;
  const user = c.get("user");

  const streak = await Streak.create({
    title,
    description,
    color,
    icon,
    goal: new Date(goal),
    start_date: new Date(),
    buffer,
    user_id: user._id,
  });

  return c.json(
    new ApiResponse(201, streak, "Streak created successfully"),
    201
  );
};
```

### Validation

- **ALWAYS** use `safeParse` — never `parse()`, which throws an unstructured error
- **ALWAYS** destructure validated fields explicitly — never spread `result.data` into a create/update
- Define schemas at module scope, above the controllers that use them
- Derive update schemas: `const updateStreakSchema = createStreakSchema.partial();`
- Validate params and query too — `c.req.param("id")` and `c.req.query()` are untrusted strings

### Async & Errors

- **ALWAYS** `await` every promise — a floating promise rejects outside the error handler and crashes the process
- **NEVER** wrap controllers in `try/catch` just to re-throw — `app.onError` catches thrown errors, including from async handlers
- Use `try/catch` only when you genuinely recover or need to add context

## Response Contract

Every response — success or failure — has the same shape:

```typescript
{ statusCode: number, data: T, message: string, success: boolean }
```

```typescript
// utils/apiResponse.ts — success
return c.json(new ApiResponse(200, streaks, "Streaks fetched successfully"), 200);

// utils/apiError.ts — failure, thrown not returned
throw new ApiError(404, "Streak not found");
throw new ApiError(400, "Validation failed", issues);

// Wrapping a lower-level failure — keep the original via `cause`
throw new ApiError(502, "Provider unreachable", [], { cause: error });
```

`ApiError` forwards `cause` to `Error` so the original failure survives into the logs:

```typescript
class ApiError extends Error {
  statusCode: number;
  errors: string[];
  success = false;

  constructor(
    statusCode: number,
    message: string,
    errors: string[] = [],
    options?: ErrorOptions
  ) {
    super(message, options); // `options.cause` preserved here
    this.statusCode = statusCode;
    this.errors = errors;
  }
}

export default ApiError;
```

- **ALWAYS** return `new ApiResponse(...)` — never a bare object or array
- **ALWAYS** pass the status code to `c.json` as well as to the constructor
- **ALWAYS** `throw` an `ApiError` — never `return c.json({ error })` from a controller
- Status codes: `200` read/update, `201` create, `400` validation, `401` unauthenticated, `403` not owner, `404` missing, `409` conflict

## Error Middleware

```typescript
// src/middlewares/error.middleware.ts
import type { Context } from "hono";
import type { ContentfulStatusCode } from "hono/utils/http-status";
import ApiError from "@/utils/apiError";

export const errorHandler = (err: Error, c: Context) => {
  if (err instanceof ApiError) {
    return c.json(
      {
        statusCode: err.statusCode,
        message: err.message,
        errors: err.errors,
        success: false,
      },
      err.statusCode as ContentfulStatusCode
    );
  }

  console.error("Unhandled error:", err);
  return c.json(
    { statusCode: 500, message: "Internal server error", errors: [], success: false },
    500
  );
};
```

- Unknown errors become a generic 500 — **NEVER** leak `err.message` or a stack trace from an unhandled error
- Use `ContentfulStatusCode` from `hono/utils/http-status` rather than a hand-written union of codes

## Middleware Patterns

```typescript
// Plain middleware
export const isAuthenticated = async (c: Context<AppEnv>, next: Next) => {
  const token = getCookie(c, "accessToken");
  if (!token) throw new ApiError(401, "Unauthorized — no access token");
  // ...verify, load user
  c.set("user", user);
  await next();
};

// Parameterized middleware — a factory returning a handler
export const checkIsOwner =
  (resource: Resource) => async (c: Context<AppEnv>, next: Next) => {
    // ...
    await next();
  };
```

- **ALWAYS** `await next()` — a missing await silently truncates the chain
- **ALWAYS** throw `ApiError` from middleware rather than returning a response
- Middleware that loads a resource for the handler should store it on the context (`c.set(...)`) so the controller does not re-query

## Data Layer

**Schema design, indexing, query construction, transactions, and migrations are owned by the `database` skill — read it before writing any model or query.** It routes by engine:

- **[NoSQL — MongoDB & Mongoose](../database/references/nosql-mongodb.md)** — the current stack
- **[SQL — Postgres & Drizzle](../database/references/sql-postgres.md)** — if a relational store is added

What this skill requires of controllers when they call into it:

- **ALWAYS** paginate and project collection reads — a controller returning an unbounded list is a bug regardless of engine
- **ALWAYS** treat "nothing matched" as `throw new ApiError(404, ...)`, never a 200 with an empty body
- **ALWAYS** scope the query by the authenticated user (`user_id` / `userId`) rather than trusting a prior middleware check alone
- **ALWAYS** hand the data layer explicit, Zod-validated fields — never the raw request body
- **NEVER** build queries in a route file or a `Context`-aware helper; the model owns the query
- **NEVER** return a raw document to `c.json` when it may carry hidden fields — serialize deliberately

```typescript
// Controller → data layer: validated fields in, checked result out
const streak = await Streak.findOneAndUpdate(
  { _id: id, user_id: user._id },
  { $set: { title, description } },
  { new: true, runValidators: true }
);
if (!streak) throw new ApiError(404, "Streak not found");

return c.json(new ApiResponse(200, streak, "Streak updated successfully"), 200);
```

## Testing

```typescript
import { describe, it, expect } from "bun:test";
import app from "@/configs/app.config";

describe("GET /api/v1/streaks", () => {
  it("rejects unauthenticated requests", async () => {
    const res = await app.request("/api/v1/streaks");
    expect(res.status).toBe(401);
  });
});
```

- **USE** `app.request(...)` — Hono apps are testable without binding a port
- **USE** `bun:test` — never Jest or Vitest
- Test the middleware chain (401/403 paths), not just the happy path

## Constraints

- **ALWAYS** use `Bun.serve` with `app.fetch`
- **ALWAYS** type context as `Context<AppEnv>` when reading context variables
- **ALWAYS** validate request bodies with Zod `safeParse` before touching the database
- **ALWAYS** return `ApiResponse` and throw `ApiError` — no ad-hoc response shapes
- **ALWAYS** use `@/` path aliases — never relative `../` imports
- **ALWAYS** consult the `database` skill before writing a model, query, or migration
- **NEVER** put logic in route files or DB queries in middleware beyond ownership checks
- **NEVER** catch an error only to re-throw it — let `app.onError` handle it
- **NEVER** import `dotenv` — Bun loads `.env` natively
- **Follow** existing patterns in `src/` before introducing a new one
