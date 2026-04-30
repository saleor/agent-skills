---
name: saleor-core
description: >
  Saleor backend internals and behavior reference. Covers discount precedence,
  stock availability modes (legacy vs direct, 3.23+), Dashboard UI rules,
  webhook trigger conditions, denormalized field semantics, and migration
  footguns. Use when working with Saleor discounts or stock availability,
  building Dashboard UI, or debugging backend behavior.
license: MIT
metadata:
  author: saleor
  version: "1.1.0"
---

# Saleor Core

Backend behavior reference derived from the Saleor core source code. Covers
internal mechanics that aren't fully documented in the public API reference —
discount precedence, stock availability modes, denormalized fields, and known
Dashboard gotchas.

## When to Apply

- Building or debugging discount/promotion UI in the Dashboard
- Investigating why a voucher or promotion isn't applying
- Understanding order-level vs line-level discount precedence
- Working with `OrderDiscount` / `OrderLineDiscount` objects
- Debugging `unit_discount_value` on `OrderLine`
- Deciding whether discounts stack or suppress each other
- Working on stock availability, shipping zones, warehouses, or `Shop.useLegacyShippingZoneStockAvailability`
- Debugging why `PRODUCT_VARIANT_*_IN_CHANNEL` webhooks aren't firing
- Building diagnostic / "doctor" tooling that explains availability problems
- Auditing Dashboard copy that uses "available" / "in stock" / "purchasable"

## Rule Categories

| Priority | Category   | Impact   | Prefix       |
| -------- | ---------- | -------- | ------------ |
| 1        | Discounts  | CRITICAL | `discount-`  |
| 2        | Stock      | CRITICAL | `stock-`     |

## Quick Reference

### 1. Discounts (CRITICAL)

- `discount-precedence` — Full precedence hierarchy, stacking rules, manual vs voucher vs promotion interactions, denormalized field semantics, known Dashboard bug

### 2. Stock (CRITICAL)

- `stock-availability-modes` — `Shop.useLegacyShippingZoneStockAvailability` (3.23+), purchasability vs shippability, server queryset dispatch, mode-conditional webhooks, Dashboard UI / copy / severity rules, migration footguns, test fixtures

## How to Use

Read individual rule files for detailed explanations and source-level evidence:

```
rules/discount-precedence.md
rules/stock-availability-modes.md
```

Each rule file contains:

- Precedence hierarchy or computation rules
- Source code references from `saleor/` with function names and file paths
- Anti-patterns and known bugs
