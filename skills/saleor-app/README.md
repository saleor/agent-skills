# saleor-app

Universal agent skill for building Saleor apps. Framework-agnostic protocol docs with Next.js examples using `@saleor/app-sdk`.

## Installation

```bash
npx skills add saleor/agent-skills --skill saleor-app
```

## What's Included

### Rules (progressively disclosed)

| Rule | Topic |
|------|-------|
| `protocol-manifest` | App manifest, endpoints, permissions, extensions |
| `protocol-auth` | Registration, APL, token scopes, JWT verification |
| `permissions-access-scopes` | User vs app scope, client-side permission checks, JWT middleware |
| `webhook-async` | Async events, payloads, retries, signatures |
| `webhook-sync` | Sync events, response format, performance |
| `webhook-external` | External service webhooks, multi-tenant routing |
| `data-graphql` | GraphQL client, auth headers, codegen |
| `data-settings` | Metadata persistence, encryption, domain scoping |
| `dashboard-appbridge` | Iframe communication, actions, events |
| `dev-debug` | Error catalog, dry runs, tunnel setup |

### References (loaded on demand)

| Reference | Purpose |
|-----------|---------|
| `apl-implementations` | APL comparison and setup |
| `app-manifest-schema` | Full manifest field reference |

## What's NOT Included

- Specific app implementations (payment, tax, shipping) — these apply the patterns above
- Saleor Dashboard internals
- Deployment/infrastructure
- Exhaustive webhook event lists — see [Saleor Docs](https://docs.saleor.io/developer/extending/webhooks/overview)
