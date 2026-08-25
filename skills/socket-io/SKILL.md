---
name: socket-io
description: Build real-time features with Socket.IO across the stack — Bun engine setup alongside Hono, JWT handshake auth, rooms, typed events, and the Next.js client. Use when adding or modifying WebSocket events, live updates, presence, or any real-time behavior on either the backend or frontend.
---

Socket.IO powers all real-time communication. This is a single skill covering **both** the backend server (Hono on Bun) and the frontend client (Next.js), because the two sides share one event contract and must change together.

## Scope

**In scope:** Socket.IO server setup with the Bun engine, handshake authentication, rooms and namespaces, the shared event contract, emitting from HTTP controllers, the frontend socket client, React integration, and reconnection handling.

**Out of scope:** HTTP routing and controllers (see `hono-bun-backend`), React component structure (see `modern-best-practice-react-components`), auth token issuance and security policy (see `web-security`).

## Golden Rules

- **One event contract, shared by both sides** — never define an event name as a loose string in two places
- **Sockets carry deltas, HTTP carries truth** — a socket event is a notification, not the system of record
- **Authenticate at the handshake**, never per-event
- **One socket instance per browser tab** — never one per component

## Backend — Bun Engine + Hono

Socket.IO runs on `@socket.io/bun-engine` so it shares the port with Hono under `Bun.serve`. **Do not** use `@hono/node-server` to host Socket.IO — that gives up Bun's native WebSockets.

```bash
bun add socket.io @socket.io/bun-engine
```

```typescript
// src/configs/socket.config.ts
import { Server } from "socket.io";
import { Server as Engine } from "@socket.io/bun-engine";
import type { ClientToServerEvents, ServerToClientEvents } from "@/types/socket";

export const io = new Server<ClientToServerEvents, ServerToClientEvents>();

export const engine = new Engine({
  path: "/socket.io/",
  cors: {
    origin: process.env.CLIENT_URL ?? "http://localhost:3000",
    credentials: true,
  },
});

io.bind(engine);
```

```typescript
// src/configs/app.config.ts — route the handshake path through Hono
import { engine } from "@/configs/socket.config";

app.all("/socket.io/*", (c) => engine.handleRequest(c.req.raw, c.env));
```

```typescript
// src/index.ts — one server, both protocols
import { engine } from "@/configs/socket.config";
import "@/sockets"; // registers connection handlers

await connectDB();

Bun.serve({
  port: PORT,
  idleTimeout: 30, // MUST exceed the engine pingInterval (25s default)
  ...engine.handler(), // provides fetch + websocket
  fetch: app.fetch, // Hono handles everything else
});
```

- **ALWAYS** set `idleTimeout` above the engine's `pingInterval`, or Bun kills idle sockets mid-heartbeat
- **ALWAYS** spread `engine.handler()` **before** `fetch: app.fetch` so the Hono fetch handler wins for HTTP
- **ALWAYS** set `credentials: true` in the engine CORS config — cookie auth fails silently without it

## Handshake Authentication

Auth happens once, in an `io.use` middleware, using the same HTTP-only cookie the REST API uses. The frontend sends it via `withCredentials: true`.

```typescript
// src/sockets/auth.socket.ts
import { io } from "@/configs/socket.config";
import { parse } from "hono/utils/cookie";
import jwt from "jsonwebtoken";
import User from "@/models/user.model";

io.use(async (socket, next) => {
  try {
    const cookies = parse(socket.handshake.headers.cookie ?? "");
    const token = cookies.accessToken;
    if (!token) return next(new Error("Unauthorized"));

    const secret = process.env.JWT_SECRET;
    if (!secret) return next(new Error("Server misconfigured"));

    const payload = jwt.verify(token, secret) as { userId: string };
    const user = await User.findById(payload.userId).select("_id username avatar");
    if (!user) return next(new Error("Unauthorized"));

    socket.data.user = user; // typed via SocketData
    next();
  } catch {
    next(new Error("Unauthorized"));
  }
});
```

- **ALWAYS** reject with `next(new Error(...))` — never let an unauthenticated socket connect and check later
- **ALWAYS** use a generic failure message — never reveal whether the token expired vs. the user was deleted
- **NEVER** trust `socket.handshake.auth` values or any client-supplied `userId` — derive identity from the verified token only
- **NEVER** re-verify per event; `socket.data.user` is set once and is authoritative for the connection's lifetime

## Rooms & Authorization

Rooms are the authorization boundary. A client receives an event only if the server put it in the room.

```typescript
// src/sockets/index.ts
import { io } from "@/configs/socket.config";
import "@/sockets/auth.socket";

io.on("connection", (socket) => {
  const user = socket.data.user;

  // Every user joins a private room keyed by their own id
  socket.join(`user:${user._id}`);

  socket.on("streak:subscribe", async (streakId) => {
    const streak = await Streak.findById(streakId);
    // Ownership check — identical rule as the HTTP layer
    if (!streak || streak.user_id.toString() !== user._id.toString()) {
      return socket.emit("error", { message: "Forbidden" });
    }
    socket.join(`streak:${streakId}`);
  });

  socket.on("disconnect", () => {
    // Rooms are left automatically; clean up only external state here
  });
});
```

- **ALWAYS** authorize inside the handler before `socket.join` — a room name from the client is untrusted input
- **ALWAYS** emit to a room or a socket, **never** broadcast globally with `io.emit` unless the data is genuinely public
- **PREFER** rooms over namespaces — namespaces are for separate protocols, not separate users
- Room naming: `resource:id` (`user:123`, `streak:456`) — always prefixed, never a bare id

## Emitting from HTTP Controllers

Real-time updates originate from the same controllers that own the write. Emit **after** the write succeeds.

```typescript
// src/controllers/streak.controllers.ts
import { io } from "@/configs/socket.config";

export const updateStreakController = async (c: Context<AppEnv>) => {
  const streak = await Streak.findByIdAndUpdate(id, result.data, {
    new: true,
    runValidators: true,
  });
  if (!streak) throw new ApiError(404, "Streak not found");

  io.to(`streak:${streak._id}`).emit("streak:updated", streak.toJSON());

  return c.json(new ApiResponse(200, streak, "Streak updated successfully"), 200);
};
```

- **ALWAYS** emit after the DB write resolves — never optimistically before
- **ALWAYS** emit the same serialized shape the REST endpoint returns, so the client has one parser
- **NEVER** emit documents containing `select: false` fields — emit `.toJSON()` of a sanitized query

## The Shared Event Contract

Both sides import the same types. Backend keeps the source of truth in `src/types/socket.ts`; the frontend mirrors it at `src/types/socket.ts`. **When one changes, change the other in the same commit.**

```typescript
// src/types/socket.ts — identical on both sides
export interface ServerToClientEvents {
  "streak:updated": (streak: Streak) => void;
  "streak:deleted": (payload: { id: string }) => void;
  error: (payload: { message: string }) => void;
}

export interface ClientToServerEvents {
  "streak:subscribe": (streakId: string) => void;
  "streak:unsubscribe": (streakId: string) => void;
}

export interface SocketData {
  user: { _id: string; username: string; avatar?: string };
}
```

- Event names are `resource:action` in lowercase — `streak:updated`, never `updateStreak` or `STREAK_UPDATED`
- Payloads are **always objects or full entities** — never positional primitives beyond a single id
- **ALWAYS** parameterize `Server<ClientToServerEvents, ServerToClientEvents, DefaultEventsMap, SocketData>` and `Socket<ServerToClientEvents, ClientToServerEvents>` — an untyped socket makes every `emit` a typo waiting to happen
- **NEVER** add an event without adding it to the interface first

## Frontend — The Client

A single module owns the socket. It is created lazily, never during module evaluation, so it cannot run during SSR.

```typescript
// src/lib/socket.ts
"use client";

import { io, type Socket } from "socket.io-client";
import type { ClientToServerEvents, ServerToClientEvents } from "@/types/socket";

type AppSocket = Socket<ServerToClientEvents, ClientToServerEvents>;

let socket: AppSocket | null = null;

export const getSocket = (): AppSocket => {
  socket ??= io(process.env.NEXT_PUBLIC_API_URL!, {
    withCredentials: true, // sends the HTTP-only auth cookie
    autoConnect: false,
    transports: ["websocket"],
  });

  return socket;
};

export const disconnectSocket = (): void => {
  socket?.disconnect();
  socket = null;
};
```

- **ALWAYS** `withCredentials: true` — the handshake auth depends on the cookie
- **ALWAYS** `autoConnect: false` and connect explicitly once the user is authenticated
- **NEVER** call `io(...)` at module top level or inside a Server Component — it must be lazy and client-only
- **NEVER** pass tokens through `auth: { token }` — the cookie is the transport

## Frontend — React Integration

Socket subscriptions are one of the few legitimate `useEffect` cases: an external system synchronized with component lifetime.

```tsx
"use client";

import { useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { getSocket } from "@/lib/socket";
import type { Streak } from "@/types";

export const useStreakSubscription = (streakId: string): void => {
  const queryClient = useQueryClient();

  useEffect(() => {
    const socket = getSocket();
    if (!socket.connected) socket.connect();

    socket.emit("streak:subscribe", streakId);

    const onUpdated = (streak: Streak) => {
      queryClient.setQueryData(["streak", streak._id], streak);
    };

    socket.on("streak:updated", onUpdated);

    return () => {
      socket.emit("streak:unsubscribe", streakId);
      socket.off("streak:updated", onUpdated); // remove THIS listener only
    };
  }, [streakId, queryClient]);
};
```

- **ALWAYS** remove listeners in the cleanup function with the same handler reference
- **NEVER** use `socket.off("event")` with no handler — it kills other components' listeners
- **NEVER** `socket.disconnect()` in a component cleanup — the socket outlives the component; disconnect only on logout
- **PREFER** pushing socket data into the TanStack Query cache (`setQueryData` / `invalidateQueries`) over a parallel `useState` — one source of truth for server data
- Define handlers inside the effect so cleanup captures the exact reference

### Connect / Disconnect Lifecycle

- Connect once when auth state becomes authenticated (in the app-level provider), not in each feature hook
- Call `disconnectSocket()` on logout so the next user does not inherit the connection
- Socket.IO reconnects automatically — **do not** write manual reconnect loops
- Handle `connect_error` once at the provider level; a repeated `Unauthorized` means the session expired → refresh or redirect to login

## Reliability

- **Treat every event as possibly duplicated or dropped.** Reconnection replays nothing by default — refetch on `connect` for critical views rather than assuming continuity
- **Make handlers idempotent** — applying the same `streak:updated` twice must be a no-op
- **NEVER** make a socket event the only path for a state change; the HTTP endpoint must produce the same result on refresh
- **AVOID** high-frequency emits (per keystroke, per scroll) — debounce on the client and rate-limit on the server

## Constraints

- **ALWAYS** use `@socket.io/bun-engine` with `Bun.serve` — never `@hono/node-server` for sockets
- **ALWAYS** authenticate in `io.use` at handshake time via the HTTP-only cookie
- **ALWAYS** authorize room joins with the same ownership rules as the HTTP layer
- **ALWAYS** type both `Server` and client `Socket` with the shared event interfaces
- **ALWAYS** keep the frontend and backend event contracts in sync in the same commit
- **ALWAYS** clean up listeners with the exact handler reference
- **NEVER** trust any identity or room name supplied by the client
- **NEVER** emit sensitive fields (`password`, `refreshToken`) over sockets
- **NEVER** create more than one socket instance per tab
