---
name: hexclave-convex
description: Use when wiring Hexclave auth to Convex in any Next.js app. Covers Convex auth setup, Hexclave-to-Convex token passing, getCurrentHexclaveUser, API route fetchQuery/fetchMutation usage, which Hexclave calls hit the network and how to avoid slow sequential ones, and obsolete anti-patterns.
---

# Hexclave Convex

Use for Hexclave + Convex integration in Next.js apps, including Turborepos. Paths may move, but the boundary must stay the same:

- Next.js API routes get the Convex token and pass `{ token }`.
- Convex functions authorize with `getCurrentHexclaveUser(ctx)`.
- Browser/API callers pass business args only; Convex derives user/team ownership from auth.

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

Create `hexclave/client.ts` for the browser app:

```ts
// hexclave/client.ts
import { HexclaveClientApp } from "@hexclave/next";

export const hexclaveClientApp = new HexclaveClientApp({
  tokenStore: "nextjs-cookie",
  urls: {
    default: {
      type: "hosted",
    },
  },
});
```

Use `tokenStore: "nextjs-cookie"` in Next.js, `"cookie"` for other web frontends, and `null` for backend environments.

Create `hexclave/server.ts` that inherits from the client app and exposes one helper for API routes:

```ts
// hexclave/server.ts
import "server-only";
import { HexclaveServerApp } from "@hexclave/next";
import { NextRequest } from "next/server";
import { hexclaveClientApp } from "./client";

export const hexclaveServerApp = new HexclaveServerApp({
  inheritsFrom: hexclaveClientApp,
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
import type { MutationCtx, QueryCtx } from "../_generated/server";

type AuthCtx = MutationCtx | QueryCtx;

export type HexclaveUser = {
  id: string;
  email: string;
  name: string;
  selectedTeamId: string;
  isAnonymous: boolean;
  isRestricted: boolean;
};

export async function getCurrentHexclaveUser(
  ctx: AuthCtx,
): Promise<HexclaveUser | null> {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity || identity.role !== "authenticated") return null;

  const { subject: id, email, name, selected_team_id: teamId } = identity;

  if (!id || !email || !name || typeof teamId !== "string" || !teamId) {
    return null;
  }

  return {
    id,
    email,
    name,
    selectedTeamId: teamId,
    isAnonymous: Boolean(identity.is_anonymous),
    isRestricted: Boolean(identity.is_restricted),
  };
}
```

## Usage

Convex queries and mutations take business args only. Do not accept `userId`, `teamId`, or `ownerUserId`; derive them from the Hexclave user.

```ts
const user = await getCurrentHexclaveUser(ctx);
if (!user) return { ok: false, error: "Unauthenticated." } as const;

// Query by auth-owned scope.
const rows = await ctx.db
  .query("todos")
  .withIndex("by_team", (q) => q.eq("teamId", user.selectedTeamId))
  .collect();

// Write auth-owned scope.
await ctx.db.insert("todos", {
  completed: false,
  ownerUserId: user.id,
  teamId: user.selectedTeamId,
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

## Performance

Every Hexclave SDK call that is not read from the token is one HTTPS round trip to the Hexclave API. Sequential calls add up, so know which is which.

No network, read from the JWT:

- `ctx.auth.getUserIdentity()` in Convex. This is why `getCurrentHexclaveUser(ctx)` is free.
- `getPartialUser({ from: "token" })` and `usePartialUser({ from: "token" })` in Next.js.

The JWT carries `sub`, `name`, `email`, `email_verified`, `selected_team_id`, `is_anonymous`, `is_restricted`. The token partial user exposes less than that: `id`, `displayName`, `primaryEmail`, `primaryEmailVerified`, and the anonymous/restricted/MFA flags. It has no `selectedTeamId` and no `profileImageUrl`. When the team is needed, read it in Convex or call `getUser()`; do not decode the cookie by hand.

`getPartialUser({ from: "token" })` returns `null` once the access token expires (10 minutes by default) even though the user is still signed in, because it only decodes the token and never refreshes it. Never use it alone as an auth gate or redirect condition:

```ts
const user =
  (await hexclaveServerApp.getPartialUser({ from: "token" })) ??
  (await hexclaveServerApp.getUser());
```

One round trip each, never in the token: `getUser()`, permissions, team members, invitations, API keys.

- `user.selectedTeam` from the server `getUser()` is typed and documented as the client `Team`. It is enough for `id`, `displayName`, and permission checks. Server-only data (member emails, `lastActiveAt`, invitations) needs `hexclaveServerApp.getTeam(id)`, which is one more round trip. Do not cast `selectedTeam` to `ServerTeam` to skip it.
- Only call `getTeam(id)` when that server-only data is needed; never just to read the team name or id.
- Run independent calls together with `Promise.all`, e.g. `team.listUsers()` and `team.listInvitations()`.
- Client hooks (`useUser`, `usePermission`, `team.useApiKeys`) do not run during SSR and suspend on first use, one round trip each, in sequence when chained through one component tree. After that they are served from the SDK's in-memory cache.
- `usePermission(team, id)` reads the single `usePermissions(team)` list, so every permission check for a team shares one fetch.

## Anti-Patterns

Do not double authenticate in API routes. API routes pass `{ token }`; Convex calls `getCurrentHexclaveUser(ctx)`.

Legacy patterns that no longer work:

- `hexclaveServerApp.getPartialUser({ from: "convex", ctx })` in Convex functions. Use `getCurrentHexclaveUser(ctx)`.
- Loading a full Hexclave user in an API route before calling Convex. This creates double authentication.
- Creating `hexclave/convex.ts` or a Convex-side `HexclaveServerApp` for normal queries/mutations. Use `ctx.auth.getUserIdentity()`.
- Reconstructing project keys, `tokenStore`, or `urls` on `HexclaveServerApp`. Use `inheritsFrom: hexclaveClientApp`.
- Passing `teamId`, `userId`, or `ownerUserId` from the browser or API route. Derive them in Convex.
- Using Convex actions for normal auth flow. Use queries/mutations with `getCurrentHexclaveUser(ctx)`.
- Using `ConvexHttpClient` as the API route pattern. Use `fetchQuery` and `fetchMutation` from `convex/nextjs`.
- Helpers that return either a token or `NextResponse`. Prefer `string | null`; let the route return 401.
- Importing `getConvexProvidersConfig` from `@hexclave/next`. Use `@hexclave/next/convex-auth.config`.

Rules:

- Do not import `hexclave/server.ts` into Convex.
- Do not import Next.js modules into Convex.
- In Turborepos, shared packages are fine, but keep Next.js token-store code out of Convex-imported modules.
- Do not write env var fallback chains.
- Do not silently continue when auth is missing.
- Keep authenticated route responses `Cache-Control: private, no-store`.
