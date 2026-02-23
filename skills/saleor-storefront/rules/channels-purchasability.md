# Saleor Channels & Purchasability

Understanding Saleor's channel model and the "fulfillment triangle" is essential for debugging why products appear unavailable or can't be purchased.

> **Source**: [Saleor Docs - Stock Overview](https://docs.saleor.io/developer/stock/overview)

## How Channels Work

A Saleor channel represents a sales storefront with its own:
- **Currency** (one currency per channel)
- **Product visibility** (published/unpublished per channel)
- **Pricing** (variant prices set per channel)
- **Countries** (which countries can be served)

Typical multi-channel setup:

```
/us/products/...  →  Channel "us" (USD)
/uk/products/...  →  Channel "uk" (GBP)
/de/products/...  →  Channel "de" (EUR)
```

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
