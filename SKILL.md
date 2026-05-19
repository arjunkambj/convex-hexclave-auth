---
name: stack-auth-convex
description: Use when wiring Stack Auth to Convex in any Next.js app. Convex setup, Stack-to-Convex token passing, Convex getCurrentStackUser auth, API route fetchQuery/fetchMutation usage, and anti-patterns that no longer work.
---

# Stack Auth Convex

Use this as a general Stack Auth + Convex integration skill for any Next.js app.

This works the same in a Turborepo. Shared packages may move file paths around, but the boundary stays the same: Next.js API routes get the Convex token and pass `{ token }`; Convex functions authorize with `getCurrentStackUser(ctx)`.

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

Convex registers the Stack Auth component:

```ts
// convex/convex.config.ts
import stackAuthComponent from "@stackframe/stack/convex.config";
import { defineApp } from "convex/server";

const app = defineApp();

app.use(stackAuthComponent);

export default app;
```

Next.js API routes use one helper to get the Convex auth token:

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

Convex reads auth from `ctx.auth`, not from Stack server helpers:

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

Convex query: no `userId` or `teamId` args.

```ts
import { query } from "./_generated/server";
import { getCurrentStackUser } from "./stack/auth";

export const list = query({
  args: {},
  handler: async (ctx) => {
    const auth = await getCurrentStackUser(ctx);

    if (!auth.authenticated) {
      return { ok: false, error: auth.error } as const;
    }

    const todos = await ctx.db
      .query("todos")
      .withIndex("by_team", (q) => q.eq("teamId", auth.user.selectedTeamId))
      .collect();

    return { ok: true, todos } as const;
  },
});
```

Convex mutation: derive ownership from auth.

```ts
import { v } from "convex/values";
import { mutation } from "./_generated/server";
import { getCurrentStackUser } from "./stack/auth";

export const create = mutation({
  args: {
    text: v.string(),
  },
  handler: async (ctx, args) => {
    const auth = await getCurrentStackUser(ctx);

    if (!auth.authenticated) {
      return { ok: false, error: auth.error } as const;
    }

    const todoId = await ctx.db.insert("todos", {
      completed: false,
      ownerUserId: auth.user.id,
      teamId: auth.user.selectedTeamId,
      text: args.text,
    });

    return { ok: true, todoId } as const;
  },
});
```

API route query: get token, pass token as the third argument.

```ts
import { api } from "@/convex/_generated/api";
import { getStackAuthConvexServerToken } from "@/stack/server";
import { fetchQuery } from "convex/nextjs";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const token = await getStackAuthConvexServerToken(request);
  if (token == null) return NextResponse.json({ error: "Unauthenticated" }, { status: 401 });

  const result = await fetchQuery(api.todoApi.list, {}, { token });

  return NextResponse.json(result);
}
```

API route mutation: only business args go in the second argument.

```ts
import { api } from "@/convex/_generated/api";
import { getStackAuthConvexServerToken } from "@/stack/server";
import { fetchMutation } from "convex/nextjs";
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const token = await getStackAuthConvexServerToken(request);
  if (token == null) return NextResponse.json({ error: "Unauthenticated" }, { status: 401 });

  const { text } = (await request.json()) as { text: string };
  const result = await fetchMutation(api.todoApi.create, { text }, { token });

  return NextResponse.json(result);
}
```

Remember the argument positions:

```ts
await fetchQuery(api.todoApi.list, {}, { token });
await fetchMutation(api.todoApi.create, { text }, { token });
```

## Anti-Patterns

Do not double authenticate in API routes.
API routes should only pass `{ token }` to Convex. Convex should call `getCurrentStackUser(ctx)`.
Do not pass auth scope as Convex arguments.

```ts
// Wrong
await fetchQuery(api.todoApi.list, { teamId, userId }, { token });
await fetchMutation(api.todoApi.create, { text, teamId, ownerUserId }, { token });
```

Use business args only.

```ts
await fetchQuery(api.todoApi.list, {}, { token });
await fetchMutation(api.todoApi.create, { text }, { token });
```

Legacy patterns that no longer work:

- `stackServerApp.getPartialUser({ from: "convex", ctx })` in Convex functions. Use `getCurrentStackUser(ctx)`.
- Loading a full Stack user in an API route before calling Convex. This creates double authentication.
- Creating `stack/convex.ts` or a Convex-side `StackServerApp` for normal queries/mutations. Use `ctx.auth.getUserIdentity()`.
- `inheritsFrom: stackClientApp` in `StackServerApp`. Use `tokenStore: "nextjs-cookie"`.
- Passing `teamId`, `userId`, or `ownerUserId` from the browser or API route. Derive them in Convex.
- Using Convex actions for the normal auth flow. Use queries/mutations with `getCurrentStackUser(ctx)`.
- Using `ConvexHttpClient` as the API route pattern. Use `fetchQuery` and `fetchMutation` from `convex/nextjs`.
- Helpers that return either a token or `NextResponse`. Prefer `string | null` and let the route return 401.
- Importing `getConvexProvidersConfig` directly from `@stackframe/stack`. Use `@stackframe/stack/convex-auth.config`.

Rules:

- Do not import `stack/server.ts` into Convex.
- Do not import Next.js modules into Convex.
- In a Turborepo, shared packages are fine, but do not move Next.js token-store code into Convex-imported modules.
- Do not write env var fallback chains.
- Do not silently continue when auth is missing.
- Keep authenticated route responses `Cache-Control: private, no-store`.
