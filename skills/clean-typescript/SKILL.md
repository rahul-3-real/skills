---
name: clean-typescript
description: Enforces clean, strict TypeScript patterns. Use when writing or reviewing TypeScript code for type safety, strict mode compliance, naming conventions, and avoiding common TypeScript anti-patterns.
---

All TypeScript code must be clean, strict, and follow established project conventions. TypeScript strict mode is enabled in both packages.

## Scope

**In scope:** TypeScript type safety, strict mode compliance, naming conventions, type definitions, avoiding `any`, proper generics usage, interface and enum patterns, path aliases, and module organization.

**Out of scope:** Architecture decisions, runtime logic, API design, database design — defer these to the relevant agents.

## Strict Mode Rules

Both `tsconfig.json` files enable strict mode. All code must comply with:

- `strict: true` — enables all strict type checking options
- No implicit `any` — every variable, parameter, and return type must be explicitly typed or inferable
- No implicit `this` — always bind context explicitly
- Strict null checks — handle `null` and `undefined` explicitly
- Strict property initialization — class properties must be initialized or marked optional

## Naming Conventions

| Category          | Convention                    | Example                                       |
| ----------------- | ----------------------------- | --------------------------------------------- |
| Interfaces        | Plain names (no prefix)       | `User`, `Streak`                              |
| Enums             | PascalCase                    | `AuthProvider`                                |
| Types             | PascalCase                    | `Ref<T>`                                      |
| Functions         | camelCase                     | `createStreak`, `validateToken`               |
| Classes           | PascalCase                    | `ApiError`, `ApiResponse`                     |
| Variables         | camelCase                     | `accessToken`, `refreshToken`                 |
| Constants         | UPPER_SNAKE_CASE or camelCase | depends on context                            |
| Files/directories | kebab-case                    | `streak.controllers.ts`, `auth.middleware.ts` |

## Type System

All shared types live in `src/types/index.ts`:

```typescript
type Ref<T> = Types.ObjectId | T;

enum AuthProvider {
  GOOGLE = "google",
  EMAIL = "email",
}
```

API-specific types go in `src/types/api.ts`. Payload types go in `src/types/payload.ts`.

### Path Aliases

- Backend: `@/*` → `./src/*` (e.g., `import User from "@/models/user.model"`)
- Frontend: `@/*` → `./src/*` (e.g., `import Button from "@/components/common/Button"`)

**Always use `@/` aliases — never use relative `../` imports.** Relative imports are fragile and break when files are moved. The `@/` alias is configured in both `tsconfig.json` files and must be used for all intra-package imports.

## Anti-Patterns to Avoid

### Never use relative `../` imports — always use `@/` path aliases

```typescript
// BAD
import { appConfig } from "../configs/app.config";
import type { JwtPayload } from "../../types/payload";

// GOOD
import { appConfig } from "@/configs/app.config";
import type { JwtPayload } from "@/types/payload";
```

Both the backend (`app/backend`) and frontend (`app/frontend`) have `@/*` → `src/*` configured in `tsconfig.json`. Use `@/` for every intra-package import without exception.

### Never use `any`

```typescript
// BAD
const data: any = await response.json();

// GOOD
interface ApiResponseData<T> {
  statusCode: number;
  data: T;
  message: string;
  success: boolean;
}
const data: ApiResponseData<Streak[]> = await response.json();
```

### Never use `@ts-ignore` or `@ts-expect-error`

Fix the underlying type issue instead of suppressing it.

### Never use type assertions without justification

```typescript
// BAD — hides type errors
const user = body as User;

// GOOD — validate with Zod and use typed result
const result = createStreakSchema.safeParse(body);
if (!result.success) throw new ApiError(400, "Validation failed");
const { title, description } = result.data;
```

### Always use `Ref<T>` for Mongoose references

```typescript
// BAD
user_id: Types.ObjectId;

// GOOD
user_id: Ref<User>;
```

### Always use explicit return types for non-trivial functions

```typescript
// BAD — no return type, and a function declaration
function getUser(id: string) {
  return User.findById(id);
}

// GOOD — arrow function, explicit return type
const getUser = async (id: string): Promise<User | null> => {
  return User.findById(id);
};
```

### Always use arrow functions

Declare every function as an arrow assigned to a `const`. Reserve the `function` keyword for the two cases where it is genuinely required:

1. **A dynamic `this` binding** — Mongoose hooks and instance methods, and any callback whose caller sets `this`
2. **Generators** — `function*` has no arrow equivalent

```typescript
// BAD
export function generateAccessToken(userId: string): string { ... }

// GOOD
export const generateAccessToken = (userId: string): string => { ... };

// NECESSARY EXCEPTION — `this` must bind to the Mongoose document
userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  ...
});
```

- **NEVER** use `function` out of habit — if `this` is not involved, it is an arrow
- **NEVER** rely on hoisting; declare before use, which arrows enforce anyway

### Never leave unused imports or variables

Remove them immediately — TypeScript strict mode will flag them.

## Backend Type Patterns

- Mongoose model types: `mongoose.Schema<User>`, `mongoose.model<User>("User", userSchema)`
- Model export pattern: `mongoose.models.User || mongoose.model<User>("User", userSchema)`
- Error type: `ApiError` — custom class with `statusCode` and optional `errors` array
- Response type: `ApiResponse` — wrapper with `statusCode`, `data`, `message`, `success`

## Frontend Type Patterns

- Component props: inline `interface` definitions (e.g., `interface ButtonProps { ... }`)
- Zustand stores: typed with `create<StateType>()`
- TanStack Query: properly typed `useQuery<TData>` and `useMutation<TData, TError, TVariables>`
- Zod schemas: `z.infer<typeof schema>` for deriving form types

## Code Formatting

```json
{
  "singleQuote": false,
  "bracketSpacing": true,
  "tabWidth": 2,
  "semi": true,
  "trailingComma": "es5"
}
```

- **Double quotes** — never single quotes
- **Semicolons** — always
- **2-space indentation** — no tabs
- **Trailing commas** — ES5 positions

## Constraints

- **Always** use TypeScript strict mode — no opt-outs
- **Never** use `any` — find or create the correct type
- **Never** use `@ts-ignore` or `@ts-expect-error`
- **Always** use path aliases (`@/*` for both backend and frontend)
- **Always** add new types to `src/types/index.ts` (shared) or `src/types/api.ts` (API-specific)
- **Follow** established naming conventions exactly
