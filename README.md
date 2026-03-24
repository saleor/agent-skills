# Saleor Agent Skills

Universal [agent skills](https://agentskills.io) for building e-commerce applications with [Saleor](https://saleor.io).

## Available Skills

| Skill                                            | Description                                                                                          |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| [`saleor-core`](skills/saleor-core/)             | Saleor backend internals — discount precedence, stacking rules, denormalized fields, Dashboard bugs  |
| [`saleor-storefront`](skills/saleor-storefront/) | Saleor API patterns for building storefronts — data model, permissions, checkout, channels, variants |
| [`saleor-app`](skills/saleor-app/)               | Patterns for building Saleor apps — protocol manifest, dashboard appbridge, webhooks, permissions    |

## Installation

Install a specific skill with `npx skills`:

```shell
npx skills add saleor/agent-skills --skill <skill-name>
```

Where `<skill-name>` is one of: `saleor-core`, `saleor-storefront`, `saleor-app`. See each skill's README for its install command.

## What Are Agent Skills?

Agent skills are structured instruction sets that help AI coding assistants (like Cursor, Claude, Copilot) understand domain-specific patterns. They follow the [Agent Skills Specification](https://agentskills.io).

Each skill contains:

- **SKILL.md** — Overview and quick reference (agents read this first)
- **AGENTS.md** — Full compiled document with all rules expanded
- **rules/** — Individual rule files with detailed examples
- **references/** — Supporting deep-dive documentation

## License

MIT
