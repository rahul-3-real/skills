# SQL — Postgres & Drizzle

Engine-specific reference for the `database` skill. Read this when `package.json` contains `drizzle-orm`, `pg`, or `postgres`, or when the project uses `Bun.sql` / `bun:sqlite` directly. The universal rules in `SKILL.md` still apply — this covers how to express them in SQL.

Stack when relational: Postgres via **Bun's native SQL driver**, with **Drizzle ORM** for schema, types, and migrations.

> This project currently runs MongoDB — see [`nosql-mongodb.md`](nosql-mongodb.md). Use this reference only for a new relational store or a migration off Mongo, and get approval before adding either.

## Why Drizzle over raw SQL

`Bun.sql` alone is fine for a handful of queries, but it gives no schema source of truth, no generated types, and no migration history. Drizzle adds those and compiles to plain SQL with no runtime engine of its own.

- **PREFER** Drizzle for anything with more than a few tables
- **AVOID** Prisma here — it ships a separate query engine binary and fights Bun's strengths
- Raw `Bun.sql` is acceptable for one-off scripts, health checks, and hand-tuned analytical queries

```bash
bun add drizzle-orm
bun add -d drizzle-kit
```

## Connection

```typescript
// src/configs/db.config.ts
import { drizzle } from "drizzle-orm/bun-sql";
import { SQL } from "bun";
import * as schema from "@/db/schema";

const url = process.env.DATABASE_URL;
if (!url) throw new Error("DATABASE_URL is not defined");

const client = new SQL(url);

export const db = drizzle({ client, schema });

export const connectDB = async (): Promise<void> => {
  await client`SELECT 1`; // fail fast if the database is unreachable
};
```

- **ALWAYS** pass `schema` into `drizzle(...)` — without it the relational query API (`db.query.*`) is unavailable
- **ALWAYS** create exactly one `db` instance at module scope and import it — never construct a client per request
- **ALWAYS** verify connectivity at bootstrap before `Bun.serve`, and exit non-zero on failure
- For SQLite use `drizzle-orm/bun-sqlite` with `bun:sqlite` — never `better-sqlite3`

## Schema

```typescript
// src/db/schema/streak.schema.ts
import {
  pgTable,
  uuid,
  varchar,
  text,
  integer,
  timestamp,
  index,
} from "drizzle-orm/pg-core";
import { users } from "@/db/schema/user.schema";

export const streaks = pgTable(
  "streaks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    userId: uuid("user_id")
      .notNull()
      .references(() => users.id, { onDelete: "cascade" }),
    title: varchar("title", { length: 100 }).notNull(),
    description: text("description"),
    color: varchar("color", { length: 7 }).notNull(),
    buffer: integer("buffer").notNull().default(0),
    goal: timestamp("goal", { withTimezone: true }).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true })
      .notNull()
      .defaultNow()
      .$onUpdate(() => new Date()),
  },
  (table) => [index("streaks_user_id_created_at_idx").on(table.userId, table.createdAt)]
);

export type Streak = typeof streaks.$inferSelect;
export type NewStreak = typeof streaks.$inferInsert;
```

### Rules

- **Table and column names are `snake_case`; TypeScript properties are `camelCase`.** Always give the explicit SQL name as the first argument.
- **Table names are plural** (`streaks`, `users`); the exported const matches.
- **ALWAYS** export `$inferSelect` / `$inferInsert` types and use them everywhere instead of hand-written interfaces
- **ALWAYS** declare foreign keys with `.references(...)` and an explicit `onDelete` — orphan rows are a silent data bug
- **ALWAYS** use `timestamp(..., { withTimezone: true })` — a naive timestamp loses the offset and cannot be recovered
- **ALWAYS** `.notNull()` unless the column is genuinely optional; nullable-by-default schemas rot fast
- **PREFER** `uuid().defaultRandom()` for public identifiers — sequential integer ids leak volume and enable enumeration
- Money is `integer` in minor units or `numeric` — **never** `real`/`double precision`
- Enums: `pgEnum("auth_provider", ["google", "email"])` — never free-text `varchar`
- One table per file under `src/db/schema/`, re-exported from `src/db/schema/index.ts`

## Migrations

```typescript
// drizzle.config.ts (project root)
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema/index.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

```bash
bunx drizzle-kit generate    # Write a migration from schema changes — commit the SQL file
bunx drizzle-kit migrate     # Apply pending migrations
bunx drizzle-kit studio      # Browse data locally
bunx drizzle-kit push        # LOCAL PROTOTYPING ONLY — skips migration history
```

- **ALWAYS** `generate` + commit the generated SQL — the migration file is the reviewable artifact, not the schema diff
- **ALWAYS** read the generated SQL before committing; Drizzle cannot tell a rename from a drop-and-add, and the difference is your users' data
- **NEVER** use `push` against staging or production — it has no history and can silently drop columns
- **NEVER** edit an applied migration; write a new one
- Destructive changes ship in stages: add column → backfill → deploy code reading the new column → drop the old one in a later release

## Queries

```typescript
import { and, eq, desc, lt, inArray } from "drizzle-orm";
import { db } from "@/configs/db.config";
import { streaks } from "@/db/schema";

// List — projected, indexed sort, paginated
const rows = await db
  .select({
    id: streaks.id,
    title: streaks.title,
    color: streaks.color,
    createdAt: streaks.createdAt,
  })
  .from(streaks)
  .where(eq(streaks.userId, userId))
  .orderBy(desc(streaks.createdAt))
  .limit(limit)
  .offset((page - 1) * limit);
```

- **ALWAYS** project explicit columns on list endpoints — `db.select().from(t)` is `SELECT *`
- **ALWAYS** `.limit(...)` every collection read, with a hard maximum
- **PREFER** keyset pagination on large tables: `.where(and(eq(streaks.userId, userId), lt(streaks.createdAt, cursor)))`
- **NEVER** build a `where` clause by string concatenation — use the operator helpers, which parameterize
- Raw escape hatch when needed: `sql\`...\`` with `${value}` interpolation is parameterized; `sql.raw()` is **not** — never pass user input to `sql.raw`

### Writes

```typescript
const [streak] = await db
  .update(streaks)
  .set({ title, description })
  .where(and(eq(streaks.id, id), eq(streaks.userId, userId))) // ownership in the predicate
  .returning();

if (!streak) throw new ApiError(404, "Streak not found");
```

- **ALWAYS** use `.returning()` on insert/update/delete so you get the row back in one round trip
- **ALWAYS** destructure and check for `undefined` — an empty array means nothing matched, which is a 404, not a success
- **ALWAYS** scope ownership into the `where` predicate so the check and the write are atomic
- **ALWAYS** `.set({ explicit, fields })` — never spread an unvalidated request body
- Upserts: `.onConflictDoUpdate({ target: table.email, set: {...} })` — a unique constraint plus upsert beats check-then-insert, which races

## Relations & Joins

```typescript
// src/db/schema/relations.ts
export const streaksRelations = relations(streaks, ({ one }) => ({
  user: one(users, { fields: [streaks.userId], references: [users.id] }),
}));

// Relational query API — one SQL statement, nested result
const streak = await db.query.streaks.findFirst({
  where: eq(streaks.id, id),
  columns: { id: true, title: true, createdAt: true },
  with: {
    user: { columns: { id: true, username: true, avatar: true } }, // never all columns
  },
});
```

- **PREFER** `db.query.*` with `with` for nested reads — it emits a single statement, so there is no N+1
- **ALWAYS** restrict `columns` on both sides — a bare `with: { user: true }` selects the password hash
- Use explicit `.leftJoin(...)` when you need aggregate or non-nested shapes; remember a `leftJoin` makes the right side nullable in the result type
- **NEVER** loop over ids issuing one query each — use `inArray(table.id, ids)`

## Transactions

```typescript
await db.transaction(async (tx) => {
  await tx.delete(streaks).where(eq(streaks.userId, userId));
  await tx.delete(users).where(eq(users.id, userId));
});
```

- **ALWAYS** use `tx` — not `db` — for every statement inside the callback, or that statement runs outside the transaction
- Throwing inside the callback rolls back; **never** swallow an error inside a transaction
- Keep transactions short — no HTTP calls, no socket emits, no file I/O inside them (emit after it commits)
- `ON DELETE CASCADE` on the foreign key often removes the need for a transaction entirely — prefer the constraint

## Indexes & Constraints

- **ALWAYS** index foreign keys — Postgres does *not* create them automatically for FK columns
- **ALWAYS** define compound indexes in filter-then-sort order: `index(...).on(table.userId, table.createdAt)`
- **ALWAYS** express uniqueness as a `uniqueIndex(...)` or `.unique()` — the constraint is the guarantee, application checks are a race
- Use partial indexes for filtered subsets (`.where(sql\`deleted_at IS NULL\`)`) to keep them small
- Verify with `EXPLAIN ANALYZE` — look for `Index Scan`, not `Seq Scan`, on tables of any size

## Pitfalls

| Pitfall                                       | Consequence                                    | Fix                                          |
| --------------------------------------------- | ---------------------------------------------- | -------------------------------------------- |
| `db` instead of `tx` inside a transaction     | Statement commits independently; no rollback   | Use `tx` for every statement                 |
| `db.select().from(t)` on a wide table         | `SELECT *`, leaks columns, slow                | Project explicit columns                     |
| `with: { user: true }`                        | Password hash in the response                  | Restrict `columns` on the relation           |
| No index on a foreign key                     | Seq scan on every join and cascade             | Add an explicit index                        |
| `drizzle-kit push` on a shared database       | Silent column drops, no history                | `generate` + `migrate`, review the SQL       |
| `sql.raw(userInput)`                          | SQL injection                                  | Use `sql\`\`` interpolation or operators     |
| Missing `.returning()`                        | Extra round trip, no way to detect 0 rows      | `.returning()` and check the result          |
| `timestamp()` without `withTimezone`          | Offsets lost, unrecoverable                    | `{ withTimezone: true }`, store UTC          |

## Testing

```typescript
import { describe, it, expect, beforeAll, afterEach } from "bun:test";
import { migrate } from "drizzle-orm/bun-sql/migrator";
import { db } from "@/configs/db.config";
import { streaks } from "@/db/schema";

beforeAll(async () => {
  await migrate(db, { migrationsFolder: "./drizzle" });
});

afterEach(async () => {
  await db.delete(streaks);
});
```

- **ALWAYS** run migrations against the test database rather than hand-creating tables — otherwise tests pass against a schema production never had
- **ALWAYS** point at a dedicated test `DATABASE_URL` and assert the database name before deleting anything
- **PREFER** wrapping each test in a transaction that rolls back, when the driver allows it — faster and fully isolated
- **NEVER** mock the query builder to test a query
