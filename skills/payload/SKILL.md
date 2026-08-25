---
name: payload
description: Build, extend, and debug Payload CMS applications and plugins. Use for Payload configuration, collections, fields, hooks, access control, Local API queries, endpoints, adapters, transactions, jobs, and generated types.
---

# Payload CMS Development

Payload APIs and supported framework versions change frequently. Inspect the installed version and existing project configuration first, and verify version-sensitive behavior against the current official Payload documentation.

## Scope

**In scope:** `payload.config.ts`, collections, globals, fields, hooks, access control, Local API operations, custom endpoints, adapters, transactions, jobs, plugins, and generated types.

**Out of scope:** General Next.js UI work, database design outside Payload's adapter boundary, and infrastructure configuration unrelated to Payload.

## Inspect Before Editing

Read the relevant files instead of assuming a template layout:

- `package.json` for the installed `payload` and `@payloadcms/*` versions and available Bun scripts
- `payload.config.ts` for the database adapter, auth collection, editor, plugins, localization, and generated-type path
- Existing collections, globals, access functions, hooks, and generated types for local conventions
- `next.config.*` and the `(payload)` route group when the change affects Next.js integration or the Admin Panel

Use Bun for package and script operations:

```bash
bunx create-payload-app@latest my-app
bun add payload @payloadcms/next
bun run dev
```

Prefer an existing project script for code generation. If none exists, use Payload's CLI through Bun:

```bash
bunx payload generate:types
```

## Route to the Relevant Reference

Read only the references needed for the task, but read each selected reference in full before implementing:

| Task | Reference |
| --- | --- |
| Collections, globals, auth, uploads, drafts, live preview | [Collections](references/COLLECTIONS.md) |
| Fields, validation, relationships, blocks, joins, virtual fields | [Fields](references/FIELDS.md) |
| Runtime narrowing for field configurations | [Field type guards](references/FIELD-TYPE-GUARDS.md) |
| Collection and field hooks, context, revalidation | [Hooks](references/HOOKS.md) |
| Collection, field, and global authorization | [Access control](references/ACCESS-CONTROL.md) |
| Multi-tenant, contextual, time-based, and reusable access patterns | [Advanced access control](references/ACCESS-CONTROL-ADVANCED.md) |
| Local, REST, and GraphQL querying | [Queries](references/QUERIES.md) |
| Custom root, collection, and global endpoints | [Endpoints](references/ENDPOINTS.md) |
| Database, storage, email, and transaction behavior | [Adapters](references/ADAPTERS.md) |
| Authentication, jobs, components, localization, and advanced features | [Advanced features](references/ADVANCED.md) |
| Reusable Payload plugin packages | [Plugin development](references/PLUGIN-DEVELOPMENT.md) |

## Non-Negotiable Invariants

### Local API access

Payload Local API operations bypass access control by default. When acting on behalf of a user, pass both the user and `overrideAccess: false`:

```ts
const posts = await payload.find({
  collection: "posts",
  overrideAccess: false,
  user,
});
```

Use the default privileged behavior only for an intentional trusted system operation. Do not pass a user while silently leaving access overridden.

### Transactions

Pass the originating `req` to nested Payload operations in hooks and request-scoped workflows. Payload uses it to carry the transaction context:

```ts
await req.payload.create({
  collection: "audit-log",
  data: { documentId: doc.id },
  req,
});
```

Confirm the configured adapter's transaction requirements. MongoDB transactions require a replica set; adapter capabilities and configuration differ.

### Hook recursion

An operation inside a hook can trigger the same hook again. Use a typed context flag and return early when that flag is present. Pass the context to the nested operation.

### Access-control return types

Collection access may return a boolean or a query constraint. Field-level access returns a boolean only. Prefer query constraints over per-document asynchronous checks when row-level filtering can be expressed declaratively.

### Generated types

Regenerate types after collection, global, or field-schema changes. Import the generated types; never hand-edit the generated output. Use Payload's exported generic helpers for reusable plugins that cannot know the consuming project's collection slugs at authoring time.

### Query depth and data exposure

Choose `depth`, `select`, and `populate` intentionally. Do not rely on defaults for security-sensitive or high-volume responses, and do not expose hidden or relationship data merely because Payload can populate it.

## Implementation Workflow

1. Identify the installed Payload version, adapter, auth model, and local conventions.
2. Read the task-specific references above.
3. Implement the smallest configuration or code change that fits the existing architecture.
4. Regenerate types when the schema changes.
5. Run the project's typecheck, tests, and production build with Bun.
6. For authorization or hook changes, test authenticated, unauthenticated, permitted, and denied paths, plus recursion and transaction failure behavior where relevant.

## Authoritative Resources

- [Payload documentation](https://payloadcms.com/docs)
- [Payload machine-readable documentation](https://payloadcms.com/llms-full.txt)
- [Payload repository](https://github.com/payloadcms/payload)
- [Payload examples](https://github.com/payloadcms/payload/tree/3.x/examples)
