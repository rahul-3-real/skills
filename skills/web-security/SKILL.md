---
name: web-security
description: Enforce web security and avoid security vulnerabilities across the full stack. Use when writing code that handles user input, authentication, authorization, data exposure, cookies, CORS, or any security-sensitive operations in the Hono backend or Next.js 16 frontend.
---

Web security is a core requirement, not an afterthought. Assume hostile input and untrusted environments by default.

## Scope

**In scope:** Input validation, XSS prevention, injection prevention (NoSQL, command), authentication and authorization, sensitive data protection, CORS configuration, cookie security, mass assignment protection, ownership enforcement, dependency security, rate limiting considerations.

**Out of scope:** Architecture decisions, performance optimization, UI design — defer these to the relevant agents.

## Core Principles

- **NEVER** trust user input
- **ALWAYS** validate and sanitize data at boundaries
- Prefer secure defaults over configurability

## Authentication & Authorization

### Auth Flow

JWT access + refresh tokens stored in HTTP-only cookies. Backend validates via `isAuthenticated` middleware which loads user from DB. Frontend persists auth state in Zustand + localStorage (non-sensitive data only).

### Middleware Chain

Protected endpoints must follow this exact order:

```typescript
// All streak routes require authentication
streaks.use("*", isAuthenticated);

// Ownership check on specific resources
streaks.put("/:id", checkIsOwner("streak"), updateStreakController);
streaks.delete("/:id", checkIsOwner("streak"), deleteStreakController);
```

### Authorization — Ownership Only

This skill describes **ownership-based authorization** (no RBAC):

- `isAuthenticated` verifies the JWT and attaches user to Hono context via `c.set("user", user)`
- `checkIsOwner("streak")` verifies the authenticated user owns the resource
- **No roles** — every authenticated user has the same permissions, scoped to their own data

**Security checks:**

- Every write endpoint must have `isAuthenticated`
- Update/delete endpoints must have `checkIsOwner`
- Missing `checkIsOwner` = any authenticated user can modify any resource (vulnerability)

### Cookie Security

```typescript
// Cookies must use these options
{
  httpOnly: true,           // Not accessible via JavaScript
  secure: process.env.NODE_ENV === "production",  // HTTPS only in production
  sameSite: "lax",          // CSRF protection
}
```

- **NEVER** store tokens in localStorage — use HTTP-only cookies
- Frontend Zustand store persists non-sensitive auth state only (user profile, isAuthenticated flag)

## XSS Prevention

### Frontend

- **AVOID** `dangerouslySetInnerHTML` — if absolutely required, sanitize with DOMPurify
- **NEVER** interpolate untrusted data into `href`, `src`, or event handler attributes
- React auto-escapes JSX expressions — do not bypass this

```tsx
// SAFE — React auto-escapes
<p>{userContent}</p>

// DANGEROUS — raw HTML injection
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// If unavoidable — sanitize first
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

## Injection Prevention

### NoSQL Injection (MongoDB/Mongoose)

- **NEVER** pass raw request body directly to Mongoose query operators
- **ALWAYS** validate with Zod then destructure specific fields
- Watch for `$` operators in user input that could modify query behavior

```typescript
// VULNERABLE — mass assignment + potential NoSQL injection
const user = await User.create(body);

// SAFE — validate with Zod, then destructure
const result = registerSchema.safeParse(body);
if (!result.success) throw new ApiError(400, "Validation failed");
const { email, username, password } = result.data;
const user = await User.create({ email, username, password });

// VULNERABLE — user can inject query operators
const user = await User.findOne({ email: body.email });
// If body.email is { "$ne": null }, this returns any user

// SAFE — validate input type first with Zod
const result = loginSchema.safeParse(body);
const user = await User.findOne({ email: result.data.email });
```

### Mass Assignment

- **ALWAYS** validate with Zod, then destructure specific fields
- **NEVER** pass raw body directly to `.create()`, `.findByIdAndUpdate()`, or `Object.assign()`
- Protect sensitive fields: `password`, `refreshToken`

```typescript
// VULNERABLE — user can set arbitrary fields
const streak = await Streak.findByIdAndUpdate(id, body);

// SAFE — whitelist fields via Zod validation
const result = updateStreakSchema.safeParse(body);
const { title, description, color, icon, goal, buffer } = result.data;
const streak = await Streak.findByIdAndUpdate(id, {
  title,
  description,
  color,
  icon,
  goal,
  buffer,
});
```

## Sensitive Data Protection

### Fields with `select: false`

These fields must never be leaked in responses:

- `password` (bcrypt hash)
- `refreshToken`

### Population Security

```typescript
// VULNERABLE — leaks password hash, tokens
const streak = await Streak.findById(id).populate("user_id");

// SAFE — select specific fields
const streak = await Streak.findById(id).populate("user_id", "username avatar");
```

### Error Messages

- **NEVER** expose stack traces, internal paths, or database details in error responses
- Use `ApiError` with user-friendly messages
- The global error handler strips stack traces in production

## CORS Configuration

The backend configures CORS for the frontend origin with credentials enabled. Ensure:

- `credentials: true` is set (required for cookie-based auth)
- Origins are explicitly listed — never use `*` with credentials

## Input Validation

### Backend Validation (Zod)

```typescript
import { z } from "zod";

const createStreakSchema = z.object({
  title: z.string().min(1, "Title is required").max(100),
  description: z.string().max(500).optional(),
  color: z
    .string()
    .regex(/^#[0-9a-fA-F]{6}$/, "Invalid hex color")
    .optional(),
  icon: z.string().max(50).optional(),
  goal: z.string().datetime().optional(),
  buffer: z.number().int().min(0).max(1440).optional(),
});

// Always use safeParse — never parse() which throws uncaught
const result = createStreakSchema.safeParse(body);
if (!result.success) {
  const errors = result.error.errors.map((e) => e.message);
  throw new ApiError(400, "Validation failed", errors);
}
```

### Frontend Validation (Zod)

```typescript
const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "Password must be at least 8 characters"),
});
```

- **ALWAYS** validate on both client (UX) and server (security)
- Client-side validation is for UX — never trust it for security

## Dependencies & Supply Chain

- Avoid unnecessary packages — each dependency is an attack surface
- Treat third-party code as untrusted input
- Keep dependencies up to date — run `bun audit` periodically
- Review new dependency before adding — check for known vulnerabilities

## Existing Security Stack

- **bcryptjs** — Password hashing (10 rounds)
- **jsonwebtoken** — JWT token generation and verification
- **Zod** — Input validation on both frontend and backend
- **Google OAuth** — via `hono` OAuth helpers or direct implementation
- **Socket.io** — WebSocket connections authenticated via JWT (see socket-io skill for patterns)

## Constraints

- **NEVER** trust user input — validate everything at boundaries with Zod
- **NEVER** pass raw request body directly to database operations — validate and destructure
- **NEVER** use `dangerouslySetInnerHTML` without sanitization
- **NEVER** store sensitive data in localStorage or expose it in responses
- **NEVER** log passwords, tokens, or secrets
- **ALWAYS** use `ApiError` for error responses — never expose internal details
- **ALWAYS** use `isAuthenticated` + `checkIsOwner` for protected endpoints
- **ALWAYS** use `select: false` for sensitive model fields
- **ALWAYS** populate with specific field selections
- **ALWAYS** validate on the server — client validation is for UX only
- **Follow** existing security patterns — do not weaken them
