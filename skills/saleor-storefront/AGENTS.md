# Saleor Storefront

**Version 1.0.0**  
Saleor  
February 2026

> **Note:**  
> This document is mainly for agents and LLMs to follow when building,  
> maintaining, or debugging Saleor-powered storefronts. Humans may also  
> find it useful, but guidance here is optimized for AI-assisted workflows.

---

## Abstract

Universal guide for AI agents building storefronts on the Saleor e-commerce platform. Covers the Saleor GraphQL API data model, permission system, checkout lifecycle, channel architecture, and product/variant patterns. Framework-agnostic — these rules apply whether you're using Next.js, Remix, Nuxt, or a custom setup. Contains 7 rules across 4 categories.

---

## Table of Contents

1. [API](#1-api) — **CRITICAL**
   - 1.1 [Data Model](#11-data-model)
   - 1.2 [Permissions](#12-permissions)
   - 1.3 [GraphQL Patterns](#13-graphql-patterns)
   - 1.4 [API Investigation](#14-api-investigation)
2. [Products](#2-products) — **HIGH**
   - 2.1 [Products & Variants](#21-products--variants)
3. [Checkout](#3-checkout) — **HIGH**
   - 3.1 [Checkout Lifecycle](#31-checkout-lifecycle)
4. [Channels](#4-channels) — **MEDIUM**
   - 4.1 [Channels & Purchasability](#41-channels--purchasability)

---

## 1. API

**Impact: CRITICAL**

The API layer controls data fetching, type safety, and access control. Getting this wrong causes permission errors, missing data, or broken queries.

### 1.1 Data Model

Understanding Saleor's data model quirks — nullable fields, pricing structure, and automatic storefront filtering — prevents bugs and wasted debugging time.

> **Source**: [Saleor API Reference](https://docs.saleor.io/api-reference)

## Nullable Fields

Saleor's GraphQL schema has many nullable fields. Handle nulls intentionally — use optional chaining with a fallback for display values, but guard or throw when null signals a real problem:

```typescript
// Display value with fallback
const name = product.category?.name ?? "Uncategorized";

// Guard when null means something is wrong
if (!product.defaultVariant) {
	throw new Error(`Product ${product.slug} has no default variant`);
}
```

Common nullable fields that catch people off guard:

| Field                    | Nullable? | Why                                  |
| ------------------------ | --------- | ------------------------------------ |
| `product.category`       | Yes       | Products can be uncategorized        |
| `product.defaultVariant` | Yes       | Products might have no variants      |
| `variant.pricing`        | Yes       | No price set for the channel         |
| `product.thumbnail`      | Yes       | No images uploaded                   |
| `product.description`    | Yes       | Rich text JSON, can be empty or null |
| `checkout.shippingPrice` | Yes       | No shipping method selected yet      |

## Pricing Structure

Every priced entity in Saleor uses the `TaxedMoney` pattern:

```graphql
type TaxedMoney {
	gross: Money!   # Price including tax
	net: Money!     # Price excluding tax
	tax: Money!     # Tax amount
	currency: String!
}
```

Pricing is always **per-channel**. A product variant can have different prices in different channels.

### Variant Pricing

```graphql
variant {
	pricing {
		price { gross { amount currency } }              # Current price (after discounts)
		priceUndiscounted { gross { amount currency } }  # Original price
	}
}
```

A variant has a discount when `priceUndiscounted.gross.amount > price.gross.amount`.

### Discount Percentage

```typescript
const discount = Math.round(
	((undiscounted - price) / undiscounted) * 100
);
```

## Automatic Storefront Filtering

Saleor automatically filters data based on authentication token. You typically do NOT need to:

- Fetch the `visibleInStorefront` field
- Filter data client-side by visibility

The API already returns only what's meant to be shown:

| Token Type              | What's Returned                         |
| ----------------------- | --------------------------------------- |
| None (anonymous)        | Only public, `visibleInStorefront` data |
| Customer JWT            | Same as anonymous + user's own data     |
| App with specific perms | Based on granted permissions            |
| App with `MANAGE_*`     | ALL data for that domain                |

This applies to attributes, products, collections, and other entities with "visible in storefront" flags.

## Anti-patterns

❌ **Don't assume fields are non-null** — Check the schema and handle nulls explicitly  
❌ **Don't blindly use `?.` everywhere** — When null means something is broken, throw or guard  
❌ **Don't filter `visibleInStorefront` client-side** — The API already does it based on your token  
❌ **Don't mix up `gross` and `net`** — Storefronts typically display `gross` (tax-inclusive)

### 1.2 Permissions

Understanding how Saleor's permission model works prevents "Permission denied" errors and ensures your storefront queries only request data they're authorized to access.

> **Source**: [Saleor Docs - Permissions](https://docs.saleor.io/developer/permissions)

## Token Types

| Token Type            | How to Get                    | Typical Use              |
| --------------------- | ----------------------------- | ------------------------ |
| **Anonymous**         | No token                      | Product browsing         |
| **Customer JWT**      | `tokenCreate` mutation        | User account, checkout   |
| **App token**         | Saleor Dashboard → Extensions | Server-side operations   |
| **Staff JWT**         | Dashboard login               | Admin operations (never use in storefront) |

## Permission Error

When you see:

```
"To access this path, you need one of the following permissions: MANAGE_..."
```

The field requires admin permissions and isn't available to anonymous/customer tokens. Your options:

1. **Remove the field** from the storefront query (most common fix)
2. **Fetch server-side** with an app token that has the required permission

## Common Permission Boundaries

### Public (no token needed)

- Product details, listings, search
- Categories, collections
- Menu/navigation structure
- Page content

### Customer JWT required

- `me` query (current user)
- `checkoutLinesAdd`, `checkoutComplete`
- Order history (`me.orders`)
- Address book (`me.addresses`)

### App token required

- `channels` query (requires `AUTHENTICATED_APP`)
- Any field annotated with `MANAGE_*` permissions

## Channel Listing

The `channels` query requires an authenticated app token. No specific permission beyond `AUTHENTICATED_APP` is needed to list channels:

1. Create an app in **Saleor Dashboard → Extensions → Add extension**
2. No special permissions required for channel listing
3. Use the token server-side only via `SALEOR_APP_TOKEN` env var

**Never use a staff token in a storefront.** App tokens are the correct approach for server-side data that requires authentication.

## Two-Tier Query Pattern

Most Saleor storefronts use two query helpers:

```typescript
// Public queries — no auth, only publicly visible data
// Safe for caching, SSG, ISR
const products = await publicQuery(ProductListDocument, { variables });

// Authenticated queries — requires user session or app token
// NOT safe for caching (user-specific)
const checkout = await authenticatedQuery(CheckoutDocument, { variables });
```

This separation ensures:
- Only publicly visible products are fetched on display pages
- No user cookies leak into cached responses
- No "Signature has expired" errors on public pages

## Anti-patterns

❌ **Don't request `MANAGE_*` fields in storefront queries** — They'll fail for customers  
❌ **Don't use staff tokens in production storefronts** — Use app tokens instead  
❌ **Don't cache authenticated queries** — User-specific data must be fresh  
❌ **Don't mix public and authenticated data in a single cached response**

### 1.3 GraphQL Patterns

Common patterns and gotchas when writing GraphQL queries against the Saleor API.

> **Source**: [Saleor API Reference](https://docs.saleor.io/api-reference)

## Channel-Scoped Queries

Most storefront queries require a `channel` argument:

```graphql
query ProductDetails($slug: String!, $channel: String!) {
	product(slug: $slug, channel: $channel) {
		id
		name
		pricing { priceRange { start { gross { amount currency } } } }
	}
}
```

Without `channel`, pricing and availability data won't be returned.

## Variant Attribute Queries

Saleor distinguishes between two types of variant attributes via the `variantSelection` filter:

```graphql
product {
	# Attributes that IDENTIFY which variant (color, size) — interactive pickers
	selectionAttributes: attributes(variantSelection: VARIANT_SELECTION) {
		attribute { name slug }
		values { name slug }
	}

	# Attributes that DESCRIBE the variant (material, brand) — display only
	nonSelectionAttributes: attributes(variantSelection: NOT_VARIANT_SELECTION) {
		attribute { name slug }
		values { name slug }
	}
}
```

| Type              | `variantSelection` value    | Purpose                              | UI Pattern          |
| ----------------- | --------------------------- | ------------------------------------ | ------------------- |
| **Selection**     | `VARIANT_SELECTION`         | Identify which variant (color, size) | Interactive picker  |
| **Non-Selection** | `NOT_VARIANT_SELECTION`     | Describe the variant (material)      | Display-only badges |

**Key insight**: Neither type is "passed" to checkout. You only pass the `variantId`. All attributes are already stored on the variant in Saleor.

## Product Filtering

Use `ProductFilterInput` for server-side filtering:

```graphql
query Products($channel: String!, $filter: ProductFilterInput) {
	products(channel: $channel, filter: $filter, first: 20) {
		edges {
			node { id name slug }
		}
	}
}
```

Common filter fields:

| Filter             | Type                | Notes                                        |
| ------------------ | ------------------- | -------------------------------------------- |
| `categories`       | `[ID!]`             | Requires category IDs (not slugs)            |
| `collections`      | `[ID!]`             | Requires collection IDs                      |
| `price`            | `PriceRangeInput`   | `{ gte: 10, lte: 50 }`                      |
| `search`           | `String`            | Full-text search                             |
| `stockAvailability`| `StockAvailability`  | `IN_STOCK` or `OUT_OF_STOCK`                |
| `attributes`       | `[AttributeInput!]` | Filter by attribute values                   |

**Gotcha**: Categories and collections require **IDs**, not slugs. You need a slug-to-ID resolution step if your URLs use slugs.

## Pagination

Saleor uses cursor-based pagination:

```graphql
products(first: 20, after: $cursor) {
	edges { node { id name } cursor }
	pageInfo { hasNextPage endCursor }
}
```

## GraphQL Codegen

Most Saleor storefronts use `graphql-codegen` with API introspection:

```typescript
// .graphqlrc.ts or codegen.ts
const config = {
	schema: process.env.NEXT_PUBLIC_SALEOR_API_URL,
	documents: "src/graphql/*.graphql",
	generates: {
		"src/gql/": { preset: "client" },
	},
};
```

**Critical**: Always regenerate types after changing `.graphql` files. The generated files contain the full schema and typed document nodes.

## Anti-patterns

❌ **Don't forget the `channel` argument** — Most queries require it for pricing/availability  
❌ **Don't use category slugs in filters** — Resolve to IDs first  
❌ **Don't edit generated type files** — Regenerate instead  
❌ **Don't make `variantSelection` attributes interactive** when they're `NOT_VARIANT_SELECTION` — They're display-only

### 1.4 API Investigation

How to investigate Saleor API behavior when documentation is unclear or you need to understand exact data models, permission checks, or filtering logic.

## Step 1: Check the Generated Types (Schema)

If you're using `graphql-codegen` with API introspection (recommended), your generated types file contains the **full Saleor schema** as TypeScript types — all types, enums, inputs, not just the ones used in declared queries.

```bash
# Example: generated types file (path depends on your setup)
grep -A 20 "^export type Product " src/gql/graphql.ts
grep -A 10 "^export enum StockAvailability" src/gql/graphql.ts
grep -A 30 "^export type ProductFilterInput" src/gql/graphql.ts
```

This file is generated via API introspection, so it always matches your exact Saleor version. If the types confirm what you need, stop here.

## Step 2: Check Saleor Documentation

- [Saleor API Reference](https://docs.saleor.io/api-reference) — GraphQL schema with field descriptions
- [Saleor Developer Docs](https://docs.saleor.io/developer) — Guides and concepts

## Step 3: Check Saleor Source for Behavior

Types tell you *what* fields exist. For *how* they behave (permission checks, filtering logic, side effects), check the Saleor core source.

If the Saleor core repo is available locally (e.g. `../saleor/`), or clone it:

```bash
cd /tmp && git clone --depth 1 https://github.com/saleor/saleor.git saleor-core
```

### Quick Investigation Patterns

```bash
# Find where a field is resolved
grep -r "def resolve_" saleor/graphql/product/

# Find permission checks
grep -r "has_perm" saleor/graphql/product/resolvers.py

# Find mutation logic
grep -r "def perform_mutation" saleor/graphql/checkout/
```

## Examples

### Does `product.attributes` filter by `visibleInStorefront`?

Check `saleor/graphql/product/resolvers.py`:

```python
def resolve_product_attributes(root, info, *, limit):
    if requestor_has_access_to_all_attributes(info.context):
        dataloader = AttributesByProductIdAndLimitLoader        # Admin: ALL
    else:
        dataloader = AttributesVisibleToCustomerByProductIdAndLimitLoader  # Customer: FILTERED
```

**Answer**: Yes. Storefront users only see `visibleInStorefront=True` attributes automatically.

### Token-Based Data Filtering

Saleor filters data based on authentication:

| Token                      | `product.attributes` returns    |
| -------------------------- | ------------------------------- |
| None (anonymous)           | Only `visibleInStorefront=True` |
| Customer JWT               | Only `visibleInStorefront=True` |
| App with `MANAGE_PRODUCTS` | ALL attributes                  |

This pattern applies across many entities in Saleor.

## Key Insights

### Storefront Auto-Filtering

When building storefront features, you typically don't need to:

- Fetch `visibleInStorefront` field
- Filter data client-side

The API already returns only what's meant to be shown based on your token.

### Product Types Control Variant Attributes

Which attributes appear on variants is configured at the **ProductType** level:

Dashboard → Configuration → Product Types → [Type] → Variant Attributes

If an attribute doesn't appear in `variant.attributes`, check the ProductType configuration — not the query.

## Anti-patterns

❌ **Don't guess API behavior** — Check the source  
❌ **Don't filter `visibleInStorefront` client-side** — API does it  
❌ **Don't assume attribute presence** — Check ProductType config in Dashboard

#### Appendix: Saleor Core Key Directories

Reference for navigating the Saleor source code when investigating API **behavior** (permission checks, filtering logic, side effects).

> **For schema types**, prefer your generated types file first. It contains the full Saleor schema as TypeScript types, generated via API introspection. Only check the Saleor core source when you need to understand resolver behavior.

## Clone Command

```bash
cd /tmp && git clone --depth 1 https://github.com/saleor/saleor.git saleor-core
```

## Directory Structure

| Path                        | Purpose                                |
| --------------------------- | -------------------------------------- |
| `saleor/graphql/`           | Resolvers, type definitions, mutations |
| `saleor/graphql/product/`   | Product queries/mutations              |
| `saleor/graphql/attribute/` | Attribute handling                     |
| `saleor/graphql/checkout/`  | Checkout operations                    |
| `saleor/graphql/order/`     | Order processing                       |
| `saleor/graphql/account/`   | User/authentication                    |
| `saleor/product/models.py`  | Product Django models                  |
| `saleor/attribute/models/`  | Attribute Django models                |
| `saleor/checkout/models.py` | Checkout Django models                 |
| `saleor/order/models.py`    | Order Django models                    |

## Key Files by Feature

### Products

- `saleor/graphql/product/types/products.py` — Product type definition
- `saleor/graphql/product/resolvers.py` — Product query resolvers
- `saleor/graphql/product/mutations/` — Product mutations

### Attributes

- `saleor/graphql/attribute/types.py` — Attribute types
- `saleor/graphql/attribute/resolvers.py` — Attribute resolvers
- `saleor/attribute/models/base.py` — Core attribute model

### Checkout

- `saleor/graphql/checkout/types.py` — Checkout type
- `saleor/graphql/checkout/mutations/` — Checkout mutations
- `saleor/checkout/calculations.py` — Price calculations

### Permissions

- `saleor/graphql/core/enums.py` — Permission enum definitions
- `saleor/permission/enums.py` — Backend permission enums

---

## 2. Products

**Impact: HIGH**

Products and variants are the core commerce data model. Understanding how Saleor structures them is essential for building product pages and cart operations.

### 2.1 Products & Variants

Understanding Saleor's product-variant model is essential for building product detail pages, cart operations, and filtering.

> **Source**: [Saleor Docs - Products](https://docs.saleor.io/developer/products)

## Core Concept: You Add Variants to Cart, Not Products

Each variant is a specific attribute combination:

| Product | Attributes     | Variant ID |
| ------- | -------------- | ---------- |
| T-Shirt | Black + Medium | `abc123`   |
| T-Shirt | Black + Large  | `def456`   |
| T-Shirt | White + Medium | `ghi789`   |

The `checkoutLinesAdd` mutation requires a specific `variantId`. Without selecting ALL required attributes, there's no variant to add.

## Two Types of Variant Attributes

Saleor distinguishes between two types via the ProductType configuration:

| Type              | GraphQL Filter              | Purpose                              | UI Pattern          |
| ----------------- | --------------------------- | ------------------------------------ | ------------------- |
| **Selection**     | `VARIANT_SELECTION`         | Identify which variant (color, size) | Interactive picker  |
| **Non-Selection** | `NOT_VARIANT_SELECTION`     | Describe the variant (material)      | Display-only badges |

```graphql
product {
	selectionAttributes: attributes(variantSelection: VARIANT_SELECTION) {
		attribute { name slug }
		values { name slug }
	}
	nonSelectionAttributes: attributes(variantSelection: NOT_VARIANT_SELECTION) {
		attribute { name slug }
		values { name slug }
	}
}
```

**Key insight**: Neither type is "passed" to checkout. You only pass the `variantId`. All attributes are already stored on the variant in Saleor.

## Variant Pricing

Each variant has per-channel pricing:

```graphql
variant {
	pricing {
		price { gross { amount currency } }              # Current price (after discounts)
		priceUndiscounted { gross { amount currency } }  # Original price
	}
}
```

A variant has a discount when `priceUndiscounted > price`.

## Variant Availability

Availability depends on stock:

```graphql
variant {
	quantityAvailable  # Stock count (may require permission)
}
```

For storefront use, check `product.isAvailable` or variant-level stock. Remember the fulfillment triangle (see [Channels & Purchasability](#41-channels--purchasability)) — stock exists in warehouses, not channels.

## Variant Media

Variants can have their own images, separate from the product's main gallery:

```graphql
variant {
	media { url alt }  # Variant-specific images
}
product {
	media { url alt }  # Product-level images (fallback)
}
```

Common pattern: show variant images when selected, fall back to product images.

## Product Types Control Everything

Which attributes appear on variants is configured at the **ProductType** level in Saleor Dashboard:

**Dashboard → Configuration → Product Types → [Type] → Variant Attributes**

If an attribute doesn't appear in `variant.attributes`, check the ProductType configuration — it's not a query issue.

## Variant Selection UX Pattern

The recommended pattern for variant selection on storefronts:

1. **URL is source of truth** — `?color=black&size=m&variant=abc123`
2. **"Add to Cart" disabled** until ALL selection attributes are chosen
3. **Incompatible options are clickable** — they clear conflicting selections (don't block)
4. **Out-of-stock options are visible** but not clickable (strikethrough)
5. **Auto-adjustment** — selecting an incompatible option clears other selections rather than blocking

This ensures users can always explore all options without getting "stuck."

## Anti-patterns

❌ **Don't enable "Add to Cart" without full variant selection** — Needs a specific variant ID  
❌ **Don't block incompatible options** — Let users click, clear conflicting selections  
❌ **Don't assume single attribute** — Products can have multiple selection attributes  
❌ **Don't make non-selection attributes interactive** — They're display-only  
❌ **Don't use `0` in boolean checks for prices** — `0` is a valid price, use `typeof === "number"`

---

## 3. Checkout

**Impact: HIGH**

Checkout handles payment and order completion. Bugs here directly cause lost revenue and poor user experience.

### 3.1 Checkout Lifecycle

Understanding how Saleor checkout sessions work — creation, persistence, completion, and common failure modes — prevents payment errors and lost orders.

> **Source**: [Saleor Docs - Checkout](https://docs.saleor.io/developer/checkout/overview)

## Core Concept

A Saleor Checkout is a mutable object that tracks:
- Line items (variants + quantities)
- Shipping/billing addresses
- Shipping method
- Payment transactions
- Channel (set at creation, immutable)

When payment succeeds, `checkoutComplete` converts it to an immutable **Order**.

## Checkout ID

Checkout IDs are base64-encoded Saleor global IDs:

```javascript
atob("Q2hlY2tvdXQ6YThjN2Y4YjgtZmU0NS00ZTRkLThhZmItZDdjYWI2YTM5MTdm");
// → "Checkout:a8c7f8b8-fe45-4e4d-8afb-d7cab6a3917f"
```

Common storage strategies:
- **Cookie**: `checkoutId-{channel}` (survives page refreshes)
- **URL query param**: `/checkout?checkout=Q2hlY2...` (shareable)
- **localStorage**: Alternative to cookies

## Lifecycle

```
1. User adds first item → checkoutCreate mutation
2. Checkout ID stored in cookie/URL
3. User adds more items → checkoutLinesAdd
4. User sets address → checkoutShippingAddressUpdate
5. User picks shipping → checkoutDeliveryMethodUpdate
6. Payment initiated → transactionInitialize
7. Payment authorized → transactionProcess
8. checkoutComplete → Checkout becomes Order
9. Clear stored checkout ID → ready for new cart
```

### Creation

A new checkout is created when:
- User adds first item to an empty cart
- No valid checkout ID exists in storage
- Existing checkout is not found in Saleor (expired/deleted)

### Completion

When `checkoutComplete` succeeds:
- Checkout is converted to an Order
- The checkout ID becomes invalid
- A new checkout must be created for future purchases

## Common Errors

### "CHECKOUT_NOT_FULLY_PAID"

```
The authorized amount doesn't cover the checkout's total amount.
```

**Causes**:
1. Payment app is down — transaction created but authorization failed
2. Stale checkout — previous partial transactions accumulated
3. Amount mismatch — checkout total changed after transaction init (e.g., shipping added)

**Debug steps**:
1. Query the checkout's `transactions` to see their status
2. Check if `transactionEvent.type` is `AUTHORIZATION_FAILURE`
3. If so, check if the payment app is healthy

### "AUTHORIZATION_FAILURE"

```json
{
	"transactionEvent": {
		"message": "Failed to delivery request.",
		"type": "AUTHORIZATION_FAILURE"
	}
}
```

**Cause**: The payment app (Stripe, Adyen, Dummy Gateway) is not responding.

**Fix**: Check Saleor Dashboard → Apps for payment app health.

### Stale Checkout with Failed Transactions

If payment fails multiple times, the checkout accumulates partial transactions. Fresh checkout is needed:
- Clear the checkout cookie
- Navigate to checkout without the `?checkout=` URL param
- Test in incognito mode

## Debugging

### Inspect Checkout State

```graphql
query {
	checkout(id: "Q2hlY2tvdXQ6...") {
		id
		totalPrice { gross { amount currency } }
		lines { variant { name } quantity }
		shippingMethod { name }
		transactions {
			id
			chargedAmount { amount }
			authorizedAmount { amount }
		}
	}
}
```

### Check Cookie in Browser

```javascript
document.cookie.split(";").find(c => c.includes("checkoutId"));
```

## Best Practices

1. **Always use live checkout data for payment amounts** — Never use cached prices
2. **Handle "checkout not found" gracefully** — Create a new checkout
3. **Clear checkout after completion** — Stale IDs cause confusing errors
4. **Test with fresh checkouts** when debugging payment issues
5. **Check payment app health** when transactions fail with `AUTHORIZATION_FAILURE`

## Anti-patterns

❌ **Don't cache checkout data** — It's transactional, always fetch fresh  
❌ **Don't reuse checkout IDs after completion** — They become invalid  
❌ **Don't ignore failed transactions** — They accumulate and block checkout completion  
❌ **Don't display cached prices at payment time** — Use live checkout totals

---

## 4. Channels

**Impact: MEDIUM**

Channels control multi-storefront, multi-currency setups. Understanding the fulfillment model prevents confusing "product not available" issues.

### 4.1 Channels & Purchasability

Understanding Saleor's channel model and the "fulfillment triangle" is essential for debugging why products appear unavailable or can't be purchased.

> **Source**: [Saleor Docs - Stock Overview](https://docs.saleor.io/developer/stock/overview)

## How Channels Work

A Saleor channel represents a sales storefront with its own:
- **Currency** (one currency per channel)
- **Product visibility** (published/unpublished per channel)
- **Pricing** (variant prices set per channel)
- **Countries** (which countries can be served)

## The Fulfillment Triangle

Product purchasability depends on three connected entities:

```
        CHANNEL                 SHIPPING ZONE              WAREHOUSE
     (sales storefront)       (delivery region)         (inventory location)
            │                        │                        │
            ├────── assigned to ─────┤                        │
            │                        ├──── fulfills from ─────┤
            ├──────────── assigned to ────────────────────────┤
```

**All three connections must exist for a product to be purchasable.**

## 7-Point Purchasability Checklist

When debugging why a product can't be purchased in a channel, verify all conditions:

1. Product is **published** in the channel
2. Product is **available for purchase** in the channel
3. At least one variant has a **price** in the channel
4. Channel has at least one active **shipping zone**
5. That shipping zone has at least one **warehouse**
6. That warehouse has **stock** for the variant
7. That warehouse is also **assigned to the channel**

## Unreachable Stock

A warehouse assigned to a channel but **not** to any shipping zone for that channel results in "unreachable" stock — it exists in the system but customers cannot buy it.

This is the most common cause of confusing `isAvailable: false` on products that appear to have inventory.

## Why Products Differ Across Channels

The same product can be purchasable in one channel and not another because:

- **Different warehouses** are assigned to each channel
- **Different shipping zones** cover different countries per channel
- **Stock levels** vary per warehouse
- **Pricing** may only be set in certain channels

## Channel-Scoped Queries

Always pass the `channel` argument to get correct pricing and availability:

```graphql
query ProductDetails($slug: String!, $channel: String!) {
	product(slug: $slug, channel: $channel) {
		name
		isAvailable       # Availability for THIS channel
		pricing {          # Pricing in THIS channel's currency
			priceRange { start { gross { amount currency } } }
		}
	}
}
```

Without `channel`, pricing and availability fields return null.

## Listing Channels

The `channels` query requires an authenticated app token:

```graphql
query {
	channels {
		id
		slug
		name
		currencyCode
	}
}
```

Create an app in **Saleor Dashboard → Extensions → Add extension**. No special permissions beyond `AUTHENTICATED_APP` are needed to list channels. Use the token server-side only.

## Anti-patterns

❌ **Don't assume stock means purchasable** — Warehouse must be in both the channel AND a shipping zone  
❌ **Don't debug availability client-side only** — Check the 7-point checklist in Saleor Dashboard  
❌ **Don't forget the `channel` argument** — Pricing and availability require it  
❌ **Don't hardcode channel slugs** — Fetch from API or use a configuration fallback

---

## References

1. [Saleor API Reference](https://docs.saleor.io/api-reference)
2. [Saleor Developer Docs](https://docs.saleor.io/developer)
3. [Agent Skills Specification](https://agentskills.io)
