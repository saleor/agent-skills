# permissions-access-scopes

> **Source**: [Saleor Docs - App Permissions](https://docs.saleor.io/developer/extending/apps/architecture/app-permissions)

## Opening an App Requires No Permissions

> "A user can open the installed app, but they won't be able to deactivate or delete it."
>
> "Opening the app requires no permissions, so you can not assume the presence of any of them."

`MANAGE_APPS` is only required to **install, deactivate, or delete** an app. Any staff member
can open an installed app in the Dashboard sidebar. The app must handle the case where the
user has no relevant permissions.

## Access Scopes

| Scope | Token | Where | Permissions |
|-------|-------|-------|-------------|
| **User scope** | Short-lived JWT from App Bridge | Client-side and tRPC calls from the iframe | The specific staff user's personal permissions |
| **App scope** | Service token from APL | Server-side only (API routes, webhooks, sync) | App's declared permissions, independent of any user |

### User Scope

The user scope applies to the client-side of the app. The Dashboard user's JWT is
available via App Bridge. Since opening an app requires no permissions, you must check
what the user has and render accordingly:

```tsx
import { useAppBridge } from "@saleor/app-sdk/app-bridge";

const AnalyticsDashboard = () => {
  const { appBridgeState } = useAppBridge();
  const permissions = appBridgeState?.user?.permissions ?? [];

  const canViewOrders = permissions.includes("MANAGE_ORDERS");
  const canManageApp = permissions.includes("MANAGE_APPS");

  return (
    <>
      {canViewOrders ? <RevenueCharts /> : <NoAccessMessage />}
      {canManageApp && <SyncControls />}
    </>
  );
};
```

For a reusable hook (pattern from the Stripe app in [saleor/apps](https://github.com/saleor/apps)):

```tsx
export const useHasPermissions = (required: string[]): boolean => {
  const { appBridgeState } = useAppBridge();
  if (!appBridgeState?.user) return false;
  return required.every((p) => appBridgeState.user.permissions.includes(p));
};
```

On the server side (tRPC), verify the JWT is authentic but don't hardcode a permission
gate — let each route declare its own requirements:

```tsx
await verifyJWT({
  appId: ctx.appId,
  token: ctx.token,
  saleorApiUrl: ctx.saleorApiUrl,
  requiredPermissions: meta?.requiredClientPermissions, // per-route, not global
});
```

With per-route defaults:

```tsx
const t = initTRPC.context<Context>().meta<Meta>().create({
  defaultMeta: {
    requiredClientPermissions: [], // no permissions by default
  },
});

// Then specific routes opt into permission requirements:
triggerSync: protectedClientProcedure
  .meta({ requiredClientPermissions: ["MANAGE_APPS"] })
  .mutation(async ({ ctx }) => { /* admin-only action */ }),
```

### App Scope

The app scope applies to the server-side. The app's service token (stored in the APL)
has the permissions declared in the manifest, regardless of which user is logged in.
Use it for operations that run without a user present or that need permissions no
individual user may have:

```tsx
const authData = await apl.get(saleorApiUrl);
const client = createGraphQLClient({
  saleorApiUrl: authData.saleorApiUrl,
  token: authData.token, // app token — never expose to client
});

const data = await client.query(ordersQuery);
```

All users who can render the component will see the result, regardless of their
personal permissions — the app token provides the access.

### When to Use Each Scope

| Operation | Scope | Why |
|-----------|-------|-----|
| Sync data from Saleor API | App scope | Needs app permissions, runs without a user present |
| Process incoming webhooks | App scope | Saleor sends to the app, not to a user |
| Read pre-aggregated analytics | User scope (no permission gate) | Data is already in local DB, not fetched from Saleor |
| Trigger manual sync | User scope (gate on `MANAGE_APPS`) | Admin action |
| View customer PII | User scope (gate on `MANAGE_USERS`) | Sensitive data |

## Anti-Pattern: Hardcoding MANAGE_APPS in JWT Middleware

```tsx
// BAD — blocks any user without MANAGE_APPS from using the app,
// even though Saleor allows all staff to open it.
await verifyJWT({
  appId: ctx.appId,
  token: ctx.token,
  saleorApiUrl: ctx.saleorApiUrl,
  requiredPermissions: ["MANAGE_APPS"],
});
```

This pattern exists in the official apps at [saleor/apps](https://github.com/saleor/apps)
(Stripe, Avatax, SMTP, etc.) because those apps are configuration-heavy (payment gateways,
email providers). It is NOT a Saleor requirement — it's a deliberate choice that makes
sense for admin-only tools but is wrong for apps with broad audiences like analytics
dashboards. The same pattern appears in the example apps at
[saleor/examples](https://github.com/saleor/examples).

## App Extensions and Permissions

App extensions (buttons/widgets mounted in Dashboard sections) declare their own
permissions in the manifest. Saleor automatically hides extensions from users who
lack the required permissions:

```json
{
  "extensions": [
    {
      "label": "Revenue Analytics",
      "mount": "NAVIGATION_ORDERS",
      "target": "APP_PAGE",
      "permissions": ["MANAGE_ORDERS"],
      "url": "/extensions/revenue"
    }
  ]
}
```

This is separate from the app's own `permissions` field (which controls what the
app's service token can do).

## Anti-patterns

- **Don't hardcode `MANAGE_APPS` in JWT middleware** — It blocks users who can legitimately open the app
- **Don't assume any permissions on app open** — Check client-side and adapt the UI
- **Don't expose the app token to the client** — Use it server-side only
- **Don't use user scope for background operations** — Webhooks and sync need the app token
- **Don't confuse app `permissions` with extension `permissions`** — They control different things
