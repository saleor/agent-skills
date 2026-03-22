---
name: saleor-app
description: >
  Universal Saleor app development patterns. Covers the app protocol (manifest, registration,
  webhooks, authentication), SDK abstractions, settings persistence, and Dashboard integration.
  Framework-agnostic with Next.js examples.
license: MIT
metadata:
  author: saleor
  version: "1.0.0"
---

# Saleor App

Guide for building apps that extend Saleor via webhooks and the GraphQL API.
Framework-agnostic protocol documentation with Next.js examples using `@saleor/app-sdk`.

## When to Apply

- Defining an app manifest or registering webhooks
- Handling async/sync webhook events from Saleor
- Authenticating requests (registration handshake, JWT, signature verification)
- Storing app settings in Saleor metadata
- Building Dashboard UI inside the iframe
- Making GraphQL calls with app tokens
- Debugging webhook failures, auth errors, or permission issues
- Deciding who can view the app and what they should see (user vs app scope)

## Rule Categories by Priority

| Priority | Category        | Impact   | Prefix          |
| -------- | --------------- | -------- | --------------- |
| 1        | Protocol        | CRITICAL | `protocol-`     |
| 2        | Permissions     | CRITICAL | `permissions-`  |
| 3        | Webhooks        | HIGH     | `webhook-`      |
| 4        | Data & Settings | HIGH     | `data-`         |
| 5        | Dashboard UI    | MEDIUM   | `dashboard-`    |
| 6        | Development     | MEDIUM   | `dev-`          |

## Quick Reference

### 1. Protocol (CRITICAL)

- `protocol-manifest` — App manifest, required endpoints, permissions, extensions
- `protocol-auth` — Registration handshake, APL, token scopes, JWT/signature verification

### 2. Permissions (CRITICAL)

- `permissions-access-scopes` — User scope vs app scope, client-side permission checks, JWT middleware patterns

### 3. Webhooks (HIGH)

- `webhook-async` — Async event handling, payload typing, retry policy, signature verification
- `webhook-sync` — Sync event handling, response schemas, performance constraints
- `webhook-external` — Receiving webhooks from external services, multi-tenant routing

### 4. Data & Settings (HIGH)

- `data-graphql` — GraphQL from apps: client setup, auth headers, codegen, app vs user tokens
- `data-settings` — MetadataManager, EncryptedMetadataManager, domain-scoped persistence

### 5. Dashboard UI (MEDIUM)

- `dashboard-appbridge` — AppBridge iframe protocol, actions, events, theme/locale sync

### 6. Development (MEDIUM)

- `dev-debug` — Common errors, webhook dry runs, tunnel setup, debugging checklist
