# Saleor Agent Skills

Universal [agent skills](https://agentskills.io) for building e-commerce applications with [Saleor](https://saleor.io).

## Available Skills

Coming soon.

## Installation

```shell
npx skills add saleor/agent-skills --skill saleor-storefront
```

This installs the skill into `.agents/skills/` in your project, where AI agents can auto-discover it.

## What Are Agent Skills?

Agent skills are structured instruction sets that help AI coding assistants (like Cursor, Claude, Copilot) understand domain-specific patterns. They follow the [Agent Skills Specification](https://agentskills.io).

Each skill contains:
- **SKILL.md** — Overview and quick reference (agents read this first)
- **AGENTS.md** — Full compiled document with all rules expanded
- **rules/** — Individual rule files with detailed examples
- **references/** — Supporting deep-dive documentation

## For Saleor Storefront Developers

These skills are **framework-agnostic** — they cover Saleor API patterns that apply whether you're using Next.js, Remix, Nuxt, or a custom setup.

For **framework-specific** patterns (caching strategies, component architecture, file conventions), see project-level skills like [`saleor-paper-storefront`](https://github.com/saleor/paper) in specific storefront implementations.

## License

MIT
