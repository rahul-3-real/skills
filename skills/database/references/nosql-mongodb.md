# NoSQL — MongoDB & Mongoose

Engine-specific reference for the `database` skill. Read this when `package.json` contains `mongoose`. The universal rules in `SKILL.md` still apply — this covers how to express them in Mongoose.

Stack: MongoDB with Mongoose 9, Bun runtime, TypeScript strict mode.

## Connection

```typescript
// src/configs/db.config.ts
import mongoose from "mongoose";

const connectDB = async (): Promise<void> => {
  const uri = process.env.MONGODB_URI;
  if (!uri) throw new Error("MONGODB_URI is not defined");

  await mongoose.connect(uri, {
    dbName: process.env.MONGODB_DB_NAME,
    serverSelectionTimeoutMS: 5_000,
  });

  mongoose.connection.on("disconnected", () => {
    console.error("MongoDB disconnected");
  });
};

export default connectDB;
```

- **ALWAYS** `await mongoose.connect(...)` once at bootstrap, before `Bun.serve` — Mongoose buffers queries otherwise and failures surface as confusing timeouts
- **ALWAYS** set an explicit `dbName` rather than relying on the URI path
- **NEVER** call `connect` inside a model file, a controller, or a request handler
- **NEVER** disable `strictQuery` — unknown filter keys must fail loudly, not silently match everything

## Model Definition

```typescript
// src/models/streak.model.ts
import mongoose, { Schema } from "mongoose";
import type { Streak } from "@/types";

const streakSchema = new Schema<Streak>(
  {
    title: { type: String, required: true, trim: true, maxlength: 100 },
    description: { type: String, trim: true, maxlength: 500 },
    color: { type: String, required: true },
    icon: { type: String, required: true },
    goal: { type: Date, required: true },
    start_date: { type: Date, required: true, default: Date.now },
    buffer: { type: Number, min: 0, max: 1440, default: 0 },
    user_id: {
      type: Schema.Types.ObjectId,
      ref: "User",
      required: true,
      index: true,
    },
  },
  { timestamps: true }
);

// Compound index — matches the list query: filter by user, sort by newest
streakSchema.index({ user_id: 1, createdAt: -1 });

export default mongoose.models.Streak ||
  mongoose.model<Streak>("Streak", streakSchema);
```

### Rules

- **ALWAYS** use the `mongoose.models.X || mongoose.model(...)` export guard — `bun --watch` re-evaluates modules and re-registering a model throws `OverwriteModelError`
- **ALWAYS** enable `{ timestamps: true }` — gives `createdAt`/`updatedAt` for free
- **ALWAYS** type the schema and model: `new Schema<Streak>(...)`, `mongoose.model<Streak>(...)`
- **ALWAYS** declare references with `Schema.Types.ObjectId` + `ref: "ModelName"`, never a bare `String`
- **ALWAYS** use `Ref<T>` in the corresponding TypeScript interface (see `clean-typescript`)
- **ALWAYS** use `Date` for dates — never store an ISO string
- Schema constraints (`required`, `maxlength`, `enum`, `min`/`max`) are a backstop; Zod at the controller is still mandatory
- One model per file, named `<resource>.model.ts`, default-exported

## Sensitive Fields

```typescript
const userSchema = new Schema<User>({
  email: { type: String, required: true, unique: true, lowercase: true, trim: true },
  password: { type: String, required: true, select: false },
  refreshToken: { type: String, select: false },
});
```

- **ALWAYS** mark `password` and `refreshToken` as `select: false` — they are then excluded from every read unless explicitly requested
- To read one deliberately: `User.findOne({ email }).select("+password")`
- **NEVER** return a raw user document to the client; strip via `toJSON` transform or an explicit projection

```typescript
userSchema.set("toJSON", {
  transform: (_doc, ret) => {
    delete ret.password;
    delete ret.refreshToken;
    delete ret.__v;
    return ret;
  },
});
```

## Hooks & Methods

```typescript
// Hash on save — only when the field actually changed
userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = function (candidate: string): Promise<boolean> {
  return bcrypt.compare(candidate, this.password);
};
```

- **ALWAYS** guard `pre("save")` hooks with `isModified(...)` — without it, every update re-hashes an already-hashed value
- **Hooks and instance methods are the one place `function` is mandatory.** An arrow captures `this` lexically, so `this` would be the module, not the document. Everywhere else in the codebase, use arrows (see `clean-code`).
- **NEVER** rely on `pre("save")` for `findOneAndUpdate` / `updateOne` — those bypass document middleware entirely. Either use `pre("findOneAndUpdate")` or route the write through `save()`.

## Indexes

- **ALWAYS** index `user_id` and any field used in a filter or sort
- **ALWAYS** define compound indexes in the order `{ equality, sort, range }` — `{ user_id: 1, createdAt: -1 }` serves `find({user_id}).sort({createdAt:-1})`
- **ALWAYS** use `unique: true` for `email`/`username` — a "check then create" in the controller is a race condition
- Verify with `await Model.find(filter).sort(sort).explain("executionStats")` — look for `IXSCAN`, not `COLLSCAN`
- **NEVER** rely on `autoIndex` in production; it is convenient in development and a startup hazard at scale

## Queries

```typescript
// List — projected, sorted by an indexed field, paginated
const streaks = await Streak.find({ user_id: user._id })
  .select("title color icon goal createdAt")
  .sort({ createdAt: -1 })
  .skip((page - 1) * limit)
  .limit(limit)
  .lean();
```

- **ALWAYS** `.lean()` for read-only queries — returns plain objects, several times faster, no hydration cost. Omit it only when you need document methods or will call `.save()`.
- **ALWAYS** `.select(...)` on list endpoints — do not ship whole documents to render a row
- **ALWAYS** paginate: a default `limit`, a hard maximum (e.g. 100), and a deterministic sort
- **PREFER** keyset pagination for large collections — `find({ user_id, createdAt: { $lt: cursor } }).limit(limit)`; `skip` degrades linearly
- **NEVER** use `find({})` without a filter and limit

### Updates

```typescript
const streak = await Streak.findOneAndUpdate(
  { _id: id, user_id: user._id }, // ownership in the filter, not a separate read
  { $set: { title, description } },
  { new: true, runValidators: true }
);
if (!streak) throw new ApiError(404, "Streak not found");
```

- **ALWAYS** pass `{ new: true, runValidators: true }` — without `new` you get the pre-update document; without `runValidators` schema constraints are skipped
- **ALWAYS** use `$set` with explicit fields — never hand the whole request body to an update (mass assignment)
- **PREFER** scoping ownership into the filter (`{ _id, user_id }`) so the check and the write are one atomic operation
- **ALWAYS** treat a `null` result as 404 — an update that matched nothing is not a success

### Never Trust Filter Input

```typescript
// VULNERABLE — { "$ne": null } from the client matches any user
const user = await User.findOne({ email: body.email });

// SAFE — Zod guarantees a string before it reaches the query
const { email } = loginSchema.parse(body); // or safeParse per hono-bun-backend
const user = await User.findOne({ email });
```

- **ALWAYS** validate types with Zod before a value reaches a filter — see `web-security` for the full injection rules
- **ALWAYS** validate `ObjectId` params: `mongoose.isValidObjectId(id)` before querying, or a cast error surfaces as a 500

## Relationships

MongoDB has no joins. Choose deliberately:

| Pattern      | Use when                                                        | Cost                                |
| ------------ | --------------------------------------------------------------- | ----------------------------------- |
| **Embed**    | Child is owned, bounded, and always read with the parent        | Document growth; hard to query alone |
| **Reference**| Child is queried independently, unbounded, or shared            | Extra round trip (`populate`)        |

- **PREFER** embedding for small bounded lists (a handful of settings, tags)
- **PREFER** referencing for anything unbounded (a user's streaks, a streak's entries) — documents have a 16 MB ceiling and an unbounded array will find it
- **NEVER** embed something you need to query or update independently

```typescript
// Populate — ALWAYS with an explicit field list
const streak = await Streak.findById(id).populate("user_id", "username avatar");
```

- **NEVER** `populate("user_id")` without a projection — it pulls the entire user document
- **AVOID** deep/nested populate chains; if you need three levels, reconsider the schema or use an aggregation
- Populate is a second query, not a join — populating inside a loop is an N+1

## Aggregation

Use aggregation for grouping, counts, and computed reads — not for things a `find` already does.

```typescript
const stats = await Streak.aggregate([
  { $match: { user_id: user._id } }, // FIRST — uses the index, shrinks the pipeline
  { $group: { _id: "$color", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
]);
```

- **ALWAYS** put `$match` first so the pipeline can use an index
- **ALWAYS** `$project` away fields before `$group` on large documents
- **NEVER** interpolate unvalidated user input into a pipeline stage
- Aggregation results are plain objects and bypass schema types — type the return explicitly

## Transactions

```typescript
// ES2026 explicit resource management — the session is disposed automatically
await using session = await mongoose.startSession();

await session.withTransaction(async () => {
  await Streak.deleteMany({ user_id: userId }, { session });
  await User.findByIdAndDelete(userId, { session });
});
```

- **PREFER** `await using` — `Symbol.asyncDispose` on `ClientSession` is stable in the MongoDB driver from 7.4.0, and it ends the session on every exit path including a throw
- If the bundled driver predates 7.4.0, fall back to `try { ... } finally { await session.endSession(); }` — never leave the cleanup off entirely
- **ALWAYS** pass `{ session }` to every operation inside the transaction — an omitted session silently runs outside it
- Transactions require a replica set — a standalone `mongod` will reject them. Local development should use a single-node replica set or the change must be designed to be idempotent instead.
- Keep transactions short; long ones hold locks and hit the default 60s limit

## Schema Evolution

MongoDB is schemaless at the engine level — which means **the migration burden moves to you**.

- **ALWAYS** write a backfill script in `src/migrations/` for any field added to existing documents; do not rely on `default` alone, which applies only to new writes
- **ALWAYS** make the code tolerate both shapes during rollout: add field → deploy → backfill → switch reads → remove tolerance
- **NEVER** rename a field in place; add the new one, dual-write, backfill, then drop
- Migration scripts are idempotent, logged, and run with `bun run src/migrations/<name>.ts`

## Pitfalls

| Pitfall                                        | Consequence                                  | Fix                                            |
| ---------------------------------------------- | -------------------------------------------- | ---------------------------------------------- |
| Missing `mongoose.models.X \|\|` guard         | `OverwriteModelError` under `bun --watch`    | Use the export guard                            |
| No `runValidators` on `findByIdAndUpdate`      | Invalid data written silently                | Pass `{ new: true, runValidators: true }`       |
| `pre("save")` expected on `findOneAndUpdate`   | Passwords stored unhashed                    | Hook the query middleware or use `save()`       |
| Unprojected `populate`                         | Password hash leaked in the response         | `populate("user_id", "username avatar")`        |
| Query inside `for` loop                        | N+1, latency scales with result count        | `find({ _id: { $in: ids } })`                   |
| `.skip()` on a large collection                | Slows linearly, drifts as rows insert        | Keyset pagination on an indexed sort key        |
| Comparing `ObjectId` with `===`                | Ownership check always false                 | `a.toString() === b.toString()`                 |
| Unvalidated `id` param                         | `CastError` → 500 instead of 400/404         | `mongoose.isValidObjectId(id)` first            |

## Testing

```typescript
import { describe, it, expect, beforeAll, afterEach, afterAll } from "bun:test";
import mongoose from "mongoose";
import Streak from "@/models/streak.model";

beforeAll(async () => {
  await mongoose.connect(process.env.MONGODB_TEST_URI!);
});

afterEach(async () => {
  await Streak.deleteMany({});
});

afterAll(async () => {
  await mongoose.connection.dropDatabase();
  await mongoose.disconnect();
});
```

- **ALWAYS** use a dedicated test database — assert the DB name contains `test` before any `deleteMany`/`dropDatabase`
- **ALWAYS** clean collections in `afterEach` so tests are order-independent
- **NEVER** mock Mongoose to test a query — you would be testing the mock
