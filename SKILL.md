---
name: hexclave-convex
description: Use when wiring Hexclave auth to Convex in any Next.js app. Covers Convex auth setup, Hexclave-to-Convex token passing, getCurrentHexclaveUser, API route fetchQuery/fetchMutation usage, migration from legacy Stack Auth naming, and obsolete anti-patterns.
---

# Hexclave Convex

Use for Hexclave + Convex integration in Next.js apps, including Turborepos. Paths may move, but the boundary must stay the same:

- Next.js API routes get the Convex token and pass `{ token }`.
- Convex functions authorize with `getCurrentHexclaveUser(ctx)`.
- Browser/API callers pass business args only; Convex derives user/team ownership from auth.

Stack Auth is now Hexclave. Prefer `@hexclave/*`, `Hexclave*`, and `HEXCLAVE_*` names in new code. Legacy `@stackframe/*`, `Stack*`, and `STACK_*` names can still work during migration, but do not introduce them unless preserving an existing app's compatibility.

Package migration map:

- `@stackframe/stack` -> `@hexclave/next`
- `@stackframe/react` -> `@hexclave/react`
- `@stackframe/js` -> `@hexclave/js`

## Setup

Convex trusts Hexclave JWTs:

```ts
// convex/auth.config.ts
import { getConvexProvidersConfig } from "@hexclave/next/convex-auth.config";

export default {
  providers: getConvexProvidersConfig({
    projectId: process.env.NEXT_PUBLIC_HEXCLAVE_PROJECT_ID!,
  }),
};
```

Register the Hexclave Convex component:

```ts
// convex/convex.config.ts
import hexclaveComponent from "@hexclave/next/convex.config";
import { defineApp } from "convex/server";

const app = defineApp();
app.use(hexclaveComponent);
export default app;
```

Edit `hexclave/server.ts` to expose one server-only helper for API routes:

```ts
// hexclave/server.ts
import "server-only";
import { HexclaveServerApp } from "@hexclave/next";
import { NextRequest } from "next/server";

export const hexclaveServerApp = new HexclaveServerApp({
  tokenStore: "nextjs-cookie",
});

export const getHexclaveConvexServerToken = async (request: NextRequest) => {
  const token = await hexclaveServerApp.getConvexHttpClientAuth({
    tokenStore: request,
  });

  return token.length ? token : null;
};
```

In Convex, read auth from `ctx.auth`, never Hexclave server helpers:

```ts
// convex/hexclave/auth.ts
import { z } from "zod";
import type { MutationCtx, QueryCtx } from "../_generated/server";

type Ctx = MutationCtx | QueryCtx;

const HexclaveUserSchema = z.object({
  id: z.string(),
  email: z.string(),
  isAnonymous: z.boolean(),
  isRestricted: z.boolean(),
  name: z.string(),
  role: z.literal("authenticated"),
  selectedTeamId: z.string(),
});

export const getCurrentHexclaveUser = async (ctx: Ctx) => {
  const identity = await ctx.auth.getUserIdentity();

  if (identity == null) {
    return { authenticated: false, error: "Unauthenticated." } as const;
  }

  const user = HexclaveUserSchema.safeParse({
    id: identity.subject,
    email: identity.email,
    isAnonymous: identity.is_anonymous,
    isRestricted: identity.is_restricted,
    name: identity.name,
    role: identity.role,
    selectedTeamId: identity.selected_team_id,
  });

  if (!user.success) {
    return { authenticated: false, error: "Missing Hexclave user claims." } as const;
  }

  return { authenticated: true, user: user.data } as const;
};
```

## Usage

Convex queries and mutations take business args only. Do not accept `userId`, `teamId`, or `ownerUserId`; derive them from `auth.user`.

```ts
const auth = await getCurrentHexclaveUser(ctx);
if (!auth.authenticated) return { ok: false, error: auth.error } as const;

// Query by auth-owned scope.
const rows = await ctx.db
  .query("todos")
  .withIndex("by_team", (q) => q.eq("teamId", auth.user.selectedTeamId))
  .collect();

// Write auth-owned scope.
await ctx.db.insert("todos", {
  completed: false,
  ownerUserId: auth.user.id,
  teamId: auth.user.selectedTeamId,
  text: args.text,
});
```

API routes get the token, return 401 if missing, and pass `{ token }` as the third argument to `fetchQuery`/`fetchMutation`.

```ts
import { api } from "@/convex/_generated/api";
import { getHexclaveConvexServerToken } from "@/hexclave/server";
import { fetchMutation, fetchQuery } from "convex/nextjs";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const token = await getHexclaveConvexServerToken(request);
  if (token == null) return NextResponse.json({ error: "Unauthenticated" }, { status: 401 });

  return NextResponse.json(await fetchQuery(api.todoApi.list, {}, { token }));
}

export async function POST(request: NextRequest) {
  const token = await getHexclaveConvexServerToken(request);
  if (token == null) return NextResponse.json({ error: "Unauthenticated" }, { status: 401 });

  const { text } = (await request.json()) as { text: string };
  return NextResponse.json(await fetchMutation(api.todoApi.create, { text }, { token }));
}
```

## Anti-Patterns

Do not double authenticate in API routes. API routes pass `{ token }`; Convex calls `getCurrentHexclaveUser(ctx)`.

Legacy patterns that no longer work:

- `hexclaveServerApp.getPartialUser({ from: "convex", ctx })` in Convex functions. Use `getCurrentHexclaveUser(ctx)`.
- Loading a full Hexclave user in an API route before calling Convex. This creates double authentication.
- Creating `hexclave/convex.ts` or a Convex-side `HexclaveServerApp` for normal queries/mutations. Use `ctx.auth.getUserIdentity()`.
- `inheritsFrom: hexclaveClientApp` in `HexclaveServerApp`. Use `tokenStore: "nextjs-cookie"`.
- Passing `teamId`, `userId`, or `ownerUserId` from the browser or API route. Derive them in Convex.
- Using Convex actions for normal auth flow. Use queries/mutations with `getCurrentHexclaveUser(ctx)`.
- Using `ConvexHttpClient` as the API route pattern. Use `fetchQuery` and `fetchMutation` from `convex/nextjs`.
- Helpers that return either a token or `NextResponse`. Prefer `string | null`; let the route return 401.
- Importing `getConvexProvidersConfig` from `@hexclave/next`. Use `@hexclave/next/convex-auth.config`.
- Hardcoding `https://api.stack-auth.com` in new Hexclave SDK projects. Use `https://api.hexclave.com`.
- Using `@stackframe/*`, `Stack*`, `STACK_*`, or `X-Stack-*` names in new code unless intentionally maintaining a legacy migration surface.

Rules:

- Do not import `hexclave/server.ts` into Convex.
- Do not import Next.js modules into Convex.
- In Turborepos, shared packages are fine, but keep Next.js token-store code out of Convex-imported modules.
- Do not write env var fallback chains.
- Do not silently continue when auth is missing.
- Keep authenticated route responses `Cache-Control: private, no-store`.
