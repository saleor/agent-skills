---
name: saleor-core
description: >
  Saleor backend internals and behavior reference. Covers discount precedence,
  order-level vs line-level discount stacking, manual/voucher/promotion interactions,
  and denormalized field semantics. Use when working with Saleor discounts, building
  Dashboard discount UI, or debugging discount application order.
license: MIT
metadata:
  author: saleor
  version: "1.0.0"
---

# Saleor Core

Backend behavior reference derived from the Saleor core source code. Covers
internal mechanics that aren't fully documented in the public API reference —
discount precedence, stacking rules, denormalized fields, and known Dashboard
gotchas.

## When to Apply

- Building or debugging discount/promotion UI in the Dashboard
- Investigating why a voucher or promotion isn't applying
- Understanding order-level vs line-level discount precedence
- Working with `OrderDiscount` / `OrderLineDiscount` objects
- Debugging `unit_discount_value` on `OrderLine`
- Deciding whether discounts stack or suppress each other

## Rule Categories

| Priority | Category   | Impact   | Prefix       |
| -------- | ---------- | -------- | ------------ |
| 1        | Discounts  | CRITICAL | `discount-`  |

## Quick Reference

### 1. Discounts (CRITICAL)

- `discount-precedence` — Full precedence hierarchy, stacking rules, manual vs voucher vs promotion interactions, denormalized field semantics, known Dashboard bug

## How to Use

Read individual rule files for detailed explanations and source-level evidence:

```
rules/discount-precedence.md
```

Each rule file contains:

- Precedence hierarchy with stacking matrix
- Source code references from `saleor/` with function names and file paths
- Anti-patterns and known bugs
