# saleor-core

Backend behavior reference for the [Saleor](https://saleor.io) e-commerce platform. Covers internal mechanics derived from reading the core source code — discount precedence, stacking rules, denormalized fields, and known Dashboard gotchas.

## Installation

```shell
npx skills add saleor/agent-skills --skill saleor-core
```

## What's Included

### Rules (progressively disclosed)

| Rule | Topic |
|------|-------|
| `discount-precedence` | Full precedence hierarchy, manual vs voucher vs promotion interactions, denormalized field semantics, known Dashboard bug |

## Structure

```
saleor-core/
├── SKILL.md              # Overview and quick reference (agents read this first)
├── README.md             # This file (for humans)
└── rules/                # Individual rule files
    └── discount-precedence.md
```

## What's NOT Included

- Storefront API patterns — see `saleor-storefront`
- App development patterns — see `saleor-app`
- Exhaustive Saleor source walkthrough — rules focus on behavior that matters for building integrations
