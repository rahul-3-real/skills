---
name: database
description: Design and query the database correctly — schema design, indexing, transactions, migrations, connection handling, and the data-access layer. Routes to the NoSQL (MongoDB/Mongoose) or SQL (Postgres/Drizzle) reference based on the project's stack. Use when creating or modifying models, schemas, queries, migrations, or any database access in the backend.
---

The database is the only part of the system that is genuinely hard to change after launch. Schema and index decisions outlive every controller above them — get them right first.

This skill holds the rules that hold **regardless of engine**, then routes to the reference for the stack in use.

## Scope

**In scope:** Schema and model design, indexing, query patterns, relationships, transactions, migrations, connection management, pagination, seeding, and the boundary between the data layer and everything above it.

**Out of scope:** HTTP routing, controllers, and request validation (see `hono-bun-backend`), injection and data-exposure policy (see `web-security`), TypeScript typing rules (see `clean-typescript`), package management (see `bun-first`).

## Pick the Reference First

**Before writing any data-layer code, determine the engine, then read the matching reference in full.**

```bash
grep -E '"(mongoose|drizzle-orm|pg|postgres|prisma)"' app/backend/package.json
```

| Signal in `package.json` / `src/configs/db.config.ts` | Engine            | Read                                             |
| ----------------------------------------------------- | ----------------- | ------------------------------------------------ |
| `mongoose`                                            | MongoDB (NoSQL)   | [NoSQL — MongoDB & Mongoose](references/nosql-mongodb.md) |
| `drizzle-orm`, `pg`, `postgres`, `Bun.sql`, `bun:sqlite` | Postgres / SQLite | [SQL — Postgres & Drizzle](references/sql-postgres.md)    |

**Current stack: MongoDB with Mongoose** — [`references/nosql-mongodb.md`](references/nosql-mongodb.md) is the active reference. The SQL reference applies if the project migrates or adds a relational store.

- **NEVER** mix engine idioms — no Mongo-style embedded blobs in Postgres `jsonb` to dodge a join, no join tables faked with `$lookup` chains in Mongo
- **NEVER** introduce a second database engine without explicit approval
- If the two references disagree with each other, that is intentional — follow the one matching the engine you're writing against

## Universal Rules

These apply to both engines. The references cover how to express each one.

### The Data Layer Is a Boundary

- **Only models/repositories touch the database.** Controllers call the data layer; they never build queries inline beyond a simple `findById`.
- **NEVER** let a raw database object reach the client — serialize through a known shape
- **NEVER** import the database client into utils, middleware (beyond ownership checks), or anything frontend
- Non-trivial multi-step logic belongs in `src/services/`, not in a controller and not in a model

### Schema Design

- **Model the queries you actually run**, not an abstract picture of the domain. Write down the top five reads before designing the schema.
- **Every table/collection gets `createdAt` and `updatedAt`.**
- **Every foreign relationship is explicit and typed** — never a loose string id with no reference declared
- **Store the smallest correct type**: dates as native date types (never strings), money as integer minor units (never floats), enums as constrained unions (never free text)
- **Store UTC.** Convert to local time at the presentation layer only.
- **NEVER** store derived data you can compute cheaply. If you must denormalize for read performance, document why and update it in the same write path.
- **NEVER** store secrets in plain text — hashes only, and exclude them from default reads

### Indexes

- **Index every field you filter, sort, or join on.** An unindexed filter is a full scan; it works fine on 100 rows and falls over at 100,000.
- **Compound indexes are order-sensitive** — the prefix must match your query's filter order
- **ALWAYS** add a unique index for fields that must be unique (`email`, `username`) — application-level "check then insert" is a race condition, not a constraint
- **AVOID** indexing everything: each index slows writes and consumes memory
- Verify with the engine's query planner before assuming an index is used

### Queries

- **NEVER** issue a query inside a loop** — batch with an `in` / `$in` lookup or a join
- **ALWAYS** select only the fields you need — never `SELECT *` / unprojected reads on wide documents feeding a list view
- **ALWAYS** paginate any endpoint that returns a collection. Default limit, hard maximum limit, stable sort.
- **PREFER** cursor/keyset pagination over offset for large or live datasets — offset drifts as rows are inserted
- **ALWAYS** sort by something deterministic; ties on a non-unique sort key produce duplicate or missing rows across pages
- **NEVER** interpolate user input into a query string or query operator position — parameterize/validate (see `web-security`)

### Writes & Consistency

- **Multi-document writes that must succeed or fail together go in a transaction.** A partial write is worse than a failed one.
- **ALWAYS** make writes idempotent where a client can retry — use a natural key or unique constraint rather than hoping the retry never happens
- **ALWAYS** enforce invariants in the database (unique, not-null, foreign keys, schema constraints) *and* validate at the boundary with Zod. The Zod schema is UX and shape; the database constraint is the guarantee.
- **NEVER** trust an update to be a no-op — check the affected-row/document count and throw `ApiError(404)` when nothing matched

### Connections

- **One connection/pool per process, created at bootstrap.** Never per request, never per module import.
- **ALWAYS** connect before serving traffic and exit non-zero on failure — fail fast beats serving 500s
- **ALWAYS** read connection strings from env vars; never hardcode, never commit them
- Handle the disconnect/reconnect event by logging it, not by rebuilding the client

### Schema Evolution

- **Every schema change is a migration**, including the "additive, harmless" ones. SQL: a generated migration file. NoSQL: a scripted backfill under `src/migrations/`.
- **ALWAYS** make changes backward-compatible in one release: add the new field, backfill, switch reads, *then* drop the old one — never all at once
- **NEVER** edit a migration that has already run anywhere; write a new one
- **NEVER** run a destructive change (drop, rename, type narrowing) against production data without a verified backup and an explicit go-ahead

### Testing & Seeding

- Test the data layer against a real database (a disposable local instance), not a mock — mocks validate your assumptions about the engine, not the engine
- **ALWAYS** reset state between tests; leaking fixtures cause order-dependent failures
- Seed scripts live in `src/seeds/` and are idempotent — running twice must not duplicate data
- **NEVER** point a test or seed script at a production connection string; assert on the DB name before destructive operations

## Constraints

- **ALWAYS** read the engine-specific reference before writing data-layer code
- **ALWAYS** keep database access inside models/repositories and services
- **ALWAYS** index the fields you filter and sort on, and paginate collection reads
- **ALWAYS** use transactions for writes that must be atomic
- **ALWAYS** treat every schema change as a migration
- **NEVER** query inside a loop or return unbounded result sets
- **NEVER** expose raw database objects or sensitive fields to the client
- **NEVER** mix engine idioms or add a second engine without approval
- **Follow** the existing models in `src/models/` before inventing a new pattern
