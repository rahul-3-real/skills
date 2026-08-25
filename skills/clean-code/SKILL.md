---
name: clean-code
description: General coding standards that apply to every file in the repo — naming, function and module design, error handling, comments, control flow, async correctness, dead code, and dependency discipline. Use when writing or reviewing any code in either the Hono backend or the Next.js frontend.
---

These rules apply to **all** code in the repo, backend and frontend alike. They govern how code is structured and expressed; the stack-specific skills govern what APIs to use.

## Scope

**In scope:** Naming, function design, module boundaries, control flow, error handling, comments, duplication, dead code, magic values, async correctness, logging, dependency discipline, and the definition of done.

**Out of scope:** Type system rules (see `clean-typescript`), stack APIs (see `hono-bun-backend`, `modern-best-practice-nextjs`, `modern-best-practice-react-components`, `socket-io`), styling (see `modern-tailwind`), security (see `web-security`), commits (see `git-conventions`).

## The Prime Directive

**Match the surrounding code.** Before writing anything, read the neighbouring files and follow their naming, structure, error handling, and idiom. A consistent codebase written in a style you'd personally do differently beats a codebase with two styles. Introduce a new pattern only when the existing one genuinely cannot express the requirement — and then say so explicitly.

## Naming

| Rule                          | Bad                       | Good                       |
| ----------------------------- | ------------------------- | -------------------------- |
| Say what it is                | `data`, `info`, `obj`     | `streaks`, `activeStreak`  |
| Booleans read as assertions   | `flag`, `status`          | `isOwner`, `hasExpired`    |
| Functions are verb phrases    | `streakData()`            | `getStreakById()`          |
| No abbreviations              | `usrCnt`, `strk`          | `userCount`, `streak`      |
| No type noise in the name     | `streakArray`, `strName`  | `streaks`, `name`          |
| Async is implied, not spelled | `fetchUserAsync`          | `fetchUser`                |

- Name by intent, not implementation — `notifyOwner()` survives a switch from email to push; `sendOwnerEmail()` does not
- Keep one word per concept repo-wide: pick `get` or `fetch` and stick with it
- Loop and callback variables get real names too — `streaks.map((streak) => ...)`, never `(s) => ...`
- Files and directories are kebab-case; the file name matches its primary export's concept

## Modern Syntax (ES2026)

Bun implements current ECMAScript, so there is no transpilation excuse for legacy syntax. Write to the latest standard and reach for older forms only where they are genuinely required.

### Arrow functions by default

```typescript
// BAD — function declaration, no return type
export function generateToken(userId: string) { ... }

// GOOD
export const generateToken = (userId: string): string => { ... };
```

`function` is correct in exactly two situations:

1. **A dynamic `this` binding is required** — Mongoose `pre`/`post` hooks and `schema.methods`, or any callback whose caller sets `this`. An arrow captures `this` lexically and silently breaks these.
2. **Generators** — `function*` has no arrow form.

Everything else is an arrow: components, hooks, handlers, controllers, middleware, utils, and callbacks. Assign it to a named `const` — **never** export an anonymous arrow, which erases the name from stack traces.

### Prefer the modern form

| Instead of                              | Write                                   | Why                                           |
| --------------------------------------- | --------------------------------------- | --------------------------------------------- |
| `.then()` chains at module scope        | top-level `await`                       | One async style; flatter, catchable           |
| `try { } finally { resource.close() }`  | `using` / `await using`                 | Disposal on every exit path, including throws |
| `a && a.b && a.b.c`                     | `a?.b?.c`                               | Shorter and null-safe                          |
| `value \|\| fallback`                   | `value ?? fallback`                     | `\|\|` also replaces `0`, `""`, `false`       |
| `if (!x) x = y`                         | `x ??= y`                               | Intent is the assignment, not the branch      |
| `arr[arr.length - 1]`                   | `arr.at(-1)`                            | No length arithmetic                          |
| `[...arr].sort()`, `[...arr].reverse()` | `arr.toSorted()`, `arr.toReversed()`    | Non-mutating by construction                  |
| `Object.prototype.hasOwnProperty.call`  | `Object.hasOwn(obj, key)`               | Reads as what it does                          |
| manual `reduce` into buckets            | `Object.groupBy(items, fn)`             | Purpose-built and clearer                     |
| `new Error(msg)` after a catch          | `new Error(msg, { cause: error })`      | Preserves the original failure                |
| `err instanceof Error`                  | `Error.isError(err)`                    | Correct across realms                          |
| `str.replace(/[.*+?]/g, "\\$&")`        | `RegExp.escape(str)`                    | Correct escaping, no hand-rolled regex        |
| numeric literals like `86400000`        | `86_400_000`                            | Readable at a glance                          |

- **PREFER** `structuredClone(value)` over `JSON.parse(JSON.stringify(value))` — the latter destroys `Date`, `Map`, `Set`, and `undefined`
- **PREFER** `Array.fromAsync(asyncIterable)` over a manual `for await` accumulator loop
- **PREFER** object and array destructuring with defaults over positional access and manual fallbacks
- **NEVER** use `var`; **NEVER** use `arguments` — use rest parameters
- **AVOID** a new syntax feature purely because it is new — if the older form reads better in context, keep it and say why

## Functions

- **One job per function.** If you need "and" to describe it, split it.
- **Keep them small enough to read without scrolling.** Roughly 40 lines is a smell, not a hard limit.
- **Three parameters is the ceiling.** Beyond that, take a single options object with named fields.
- **Never take a boolean parameter that selects behavior** — `render(true)` is unreadable. Write two functions.
- **Return early.** Guard clauses at the top; the happy path stays at the leftmost indentation.
- **No output parameters.** Return a new value instead of mutating an argument.
- **Keep one level of abstraction per function** — don't mix `await Streak.findById(id)` with byte-level string fiddling in the same body.

```typescript
// BAD — nested, mixed levels, mutates its argument
function process(streak, opts) {
  if (streak) {
    if (streak.active) {
      if (opts.recalc) {
        streak.count = streak.entries.filter((e) => !e.skipped).length;
        return true;
      }
    }
  }
  return false;
}

// GOOD — arrow, guards first, one job, returns a value
const countCompletedEntries = (streak: Streak): number =>
  streak.entries.filter((entry) => !entry.skipped).length;
```

## Control Flow

- **AVOID** nesting deeper than two levels — extract a function or invert the condition
- **PREFER** `map` / `filter` / `reduce` / `some` / `every` over manual index loops when transforming data
- **AVOID** `reduce` when it obscures intent; a plain `for...of` that reads clearly beats a clever one-liner
- **NEVER** mutate function arguments, module-level state, or props
- **PREFER** `const`; use `let` only when reassignment is the point; **never** `var`
- Prefer `?.` and `??` over `&&` chains and `||` fallbacks — `||` swallows `0` and `""`

## Magic Values

Every literal that carries meaning gets a name.

```typescript
// BAD
if (Date.now() - streak.lastEntry > 86400000) { ... }
setTimeout(retry, 3000);

// GOOD
const ONE_DAY_MS = 24 * 60 * 60 * 1000;
const RETRY_DELAY_MS = 3_000;
```

- Domain constants live beside the module that owns them; cross-cutting ones in `src/constants/`
- Configuration comes from env vars, never hardcoded — and read env vars in one config module, not scattered across files
- Repeated string literals used as keys or identifiers become named constants or a union type

## Duplication

- **Two occurrences: leave it.** Three: extract.
- Extract on **shared meaning**, not shared shape. Two functions that happen to look alike but change for different reasons must stay separate — merging them creates a parameter-riddled monster later.
- Copy-pasting a block and editing two lines is always a signal — either extract, or ask whether the second case really belongs.

## Module Boundaries

- **A module exports the smallest surface that gets the job done.** Helpers stay private.
- **Dependencies point one way.** Utils never import controllers or components. Shared code moves down, never sideways.
- **NEVER** create circular imports — if two modules need each other, a third module owns the shared piece.
- Barrel files (`index.ts`) are for public entry points only, not for re-exporting everything.
- A file that has grown past ~300 lines or covers two unrelated concepts should be split by concept, not by arbitrary line count.

## Error Handling

- **Fail loudly at boundaries, handle deliberately inside.** Validate input where it enters the system; trust it afterwards.
- **NEVER** swallow an error:

```typescript
// BAD — the failure vanishes
try {
  await syncStreak(id);
} catch (e) {}

// BAD — logs and continues as if nothing happened
catch (e) { console.log(e); }

// GOOD — recover meaningfully, or let it propagate with the original attached
try {
  await syncStreak(id);
} catch (error) {
  throw new ApiError(502, "Failed to sync streak with provider", [], {
    cause: error,
  });
}
```

- **ALWAYS** pass `{ cause }` when wrapping an error — without it the original failure is gone and the log says nothing useful
- **PREFER** `Error.isError(err)` over `err instanceof Error` — `instanceof` returns false for errors crossing a realm boundary

- **Catch only what you can act on.** If you can't recover or add context, don't catch — let the global handler deal with it.
- **Throw typed errors** (`ApiError`), never bare strings.
- **NEVER** use exceptions for expected outcomes — "not found" on a lookup is a return value decision, not necessarily a throw.
- Error messages describe what failed and what the caller can do about it — no internal paths, no stack traces to users.

## Async Correctness

- **ALWAYS** `await` or explicitly return every promise — a floating promise is an unhandled rejection waiting to happen
- **NEVER** `await` inside a loop when the iterations are independent — use `Promise.all`
- **PREFER** `Promise.all` for parallel work and `Promise.allSettled` when partial failure is acceptable
- **NEVER** mix `.then()` chains with `async/await` in the same function
- Async functions return `Promise<T>` — do not wrap an already-async value in `new Promise`

```typescript
// BAD — sequential for no reason
for (const id of ids) {
  results.push(await fetchStreak(id));
}

// GOOD
const results = await Promise.all(ids.map((id) => fetchStreak(id)));
```

## Comments

Comments explain **why**, never **what**. If a comment restates the code, delete the comment; if the code needs a comment to be understood, rename things instead.

```typescript
// BAD — restates the obvious
// increment the counter
counter++;

// GOOD — explains a non-obvious decision
// The provider rounds to whole days in UTC, so we normalize before
// comparing or a user in UTC+13 loses a day at the boundary.
const normalized = startOfUtcDay(entry.date);
```

- **NEVER** leave commented-out code — git history is the archive
- **NEVER** write changelog comments (`// added by ...`, `// fixed 2026-07-01`) — that's what commits are for
- `TODO` is acceptable only with concrete context: `// TODO: remove once the v2 endpoint ships (#142)`
- Match the comment density of the surrounding file — a sparsely commented codebase should not suddenly grow annotated blocks

## Dead Code

Delete on sight:

- Unused imports, variables, parameters, exports, and files
- Functions with no callers
- Feature flags for shipped features
- "Just in case" abstractions with a single implementation

Do not write code for requirements that don't exist yet. An interface with one implementer, a config option nobody sets, and a generic wrapper around one call site are all costs with no benefit.

## Logging

- Log at boundaries and on failures — not to trace normal control flow
- **NEVER** log secrets, tokens, passwords, or full request bodies
- **NEVER** ship `console.log` debugging statements; `console.error` for genuine failures is fine
- Log messages are complete sentences with context: `Failed to connect to MongoDB:` beats `error!`

## Dependencies

- **Every dependency is permanent.** Prefer a 20-line local helper over a package for something small.
- **NEVER** add a dependency without explicit approval — recommend it and say what it replaces
- Before adding: is it in the runtime already (Bun, Web APIs), in an existing dependency, or trivially written?
- Wrap third-party APIs behind your own thin module when they're used in more than a few places, so a swap is one file

## Definition of Done

Before calling any change complete:

1. It compiles under strict TypeScript with no new errors
2. `bun run lint` passes on the touched packages
3. Existing tests pass; new behavior has a test if the module is tested
4. No unused imports, no leftover `console.log`, no commented-out code
5. Error paths are handled, not just the happy path
6. It follows the conventions of the files around it

Report honestly: if something is untested, unverified, or deliberately left out, say so plainly rather than implying completeness.

## Constraints

- **ALWAYS** match the conventions of the surrounding code
- **ALWAYS** write arrow functions assigned to a named `const` — `function` only for a required `this` binding or a generator
- **ALWAYS** prefer the current-standard form (`??`, `?.`, `??=`, `at`, `toSorted`, `using`, top-level `await`) over its legacy equivalent
- **ALWAYS** name things by intent; no abbreviations, no generic `data`/`info`/`temp`
- **ALWAYS** return early and keep nesting shallow
- **ALWAYS** await or return every promise
- **ALWAYS** delete dead code rather than commenting it out
- **NEVER** swallow errors or catch what you cannot act on
- **NEVER** hardcode magic numbers, strings, or configuration
- **NEVER** add a dependency without approval
- **NEVER** build abstractions for requirements that don't exist yet
