---
name: stack-auth-convex
description: Use when wiring Stack Auth to Convex in any Next.js app. Covers Convex auth setup, Stack-to-Convex token passing, getCurrentStackUser, API route fetchQuery/fetchMutation usage, and obsolete anti-patterns.
---

# Stack Auth Convex

Use for Stack Auth + Convex integration in Next.js apps, including Turborepos. Paths may move, but the boundary must stay the same:

- Next.js API routes get the Convex token and pass `{ token }`.
- Convex functions authorize with `getCurrentStackUser(ctx)`.
- Browser/API callers pass business args only; Convex derives user/team ownership from auth.

## Setup

Convex trusts Stack Auth JWTs:

```ts
// convex/auth.config.ts
import { getConvexProvidersConfig } from "@stackframe/stack/convex-auth.config";

export default {
  providers: getConvexProvidersConfig({
    projectId: process.env.NEXT_PUBLIC_STACK_PROJECT_ID!,
  }),
};
```

Register the Stack Auth Convex component:

```ts
// convex/convex.config.ts
import stackAuthComponent from "@stackframe/stack/convex.config";
import { defineApp } from "convex/server";

const app = defineApp();
app.use(stackAuthComponent);
export default app;
```

Edit `stack/server.ts` to expose one server-only helper for API routes:

```ts
// stack/server.ts
import "server-only";
import { StackServerApp } from "@stackframe/stack";
import { NextRequest } from "next/server";

export const stackServerApp = new StackServerApp({
  tokenStore: "nextjs-cookie",
});

export const getStackAuthConvexServerToken = async (request: NextRequest) => {
  const token = await stackServerApp.getConvexHttpClientAuth({
    tokenStore: request,
  });

  return token.length ? token : null;
};
```

In Convex, read auth from `ctx.auth`, never Stack server helpers:

```ts
// convex/stack/auth.ts
import { z } from "zod";
import type { MutationCtx, QueryCtx } from "../_generated/server";

type Ctx = MutationCtx | QueryCtx;

const StackUserSchema = z.object({
  id: z.string(),
  email: z.string(),
  isAnonymous: z.boolean(),
  isRestricted: z.boolean(),
  name: z.string(),
  role: z.literal("authenticated"),
  selectedTeamId: z.string(),
});

export const getCurrentStackUser = async (ctx: Ctx) => {
  const identity = await ctx.auth.getUserIdentity();

  if (identity == null) {
    return { authenticated: false, error: "Unauthenticated." } as const;
  }

  const user = StackUserSchema.safeParse({
    id: identity.subject,
    email: identity.email,
    isAnonymous: identity.is_anonymous,
    isRestricted: identity.is_restricted,
    name: identity.name,
    role: identity.role,
    selectedTeamId: identity.selected_team_id,
  });

  if (!user.success) {
    return { authenticated: false, error: "Missing Stack user claims." } as const;
  }

  return { authenticated: true, user: user.data } as const;
};
```

## Usage

Convex queries and mutations take business args only. Do not accept `userId`, `teamId`, or `ownerUserId`; derive them from `auth.user`.

```ts
const auth = await getCurrentStackUser(ctx);
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
import { getStackAuthConvexServerToken } from "@/stack/server";
import { fetchMutation, fetchQuery } from "convex/nextjs";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const token = await getStackAuthConvexServerToken(request);
  if (token == null) return NextResponse.json({ error: "Unauthenticated" }, { status: 401 });

  return NextResponse.json(await fetchQuery(api.todoApi.list, {}, { token }));
}

export async function POST(request: NextRequest) {
  const token = await getStackAuthConvexServerToken(request);
  if (token == null) return NextResponse.json({ error: "Unauthenticated" }, { status: 401 });

  const { text } = (await request.json()) as { text: string };
  return NextResponse.json(await fetchMutation(api.todoApi.create, { text }, { token }));
}
```

## Anti-Patterns

Do not double authenticate in API routes. API routes pass `{ token }`; Convex calls `getCurrentStackUser(ctx)`.

Legacy patterns that no longer work:

- `stackServerApp.getPartialUser({ from: "convex", ctx })` in Convex functions. Use `getCurrentStackUser(ctx)`.
- Loading a full Stack user in an API route before calling Convex. This creates double authentication.
- Creating `stack/convex.ts` or a Convex-side `StackServerApp` for normal queries/mutations. Use `ctx.auth.getUserIdentity()`.
- `inheritsFrom: stackClientApp` in `StackServerApp`. Use `tokenStore: "nextjs-cookie"`.
- Passing `teamId`, `userId`, or `ownerUserId` from the browser or API route. Derive them in Convex.
- Using Convex actions for normal auth flow. Use queries/mutations with `getCurrentStackUser(ctx)`.
- Using `ConvexHttpClient` as the API route pattern. Use `fetchQuery` and `fetchMutation` from `convex/nextjs`.
- Helpers that return either a token or `NextResponse`. Prefer `string | null`; let the route return 401.
- Importing `getConvexProvidersConfig` from `@stackframe/stack`. Use `@stackframe/stack/convex-auth.config`.

Rules:

- Do not import `stack/server.ts` into Convex.
- Do not import Next.js modules into Convex.
- In Turborepos, shared packages are fine, but keep Next.js token-store code out of Convex-imported modules.
- Do not write env var fallback chains.
- Do not silently continue when auth is missing.
- Keep authenticated route responses `Cache-Control: private, no-store`.
