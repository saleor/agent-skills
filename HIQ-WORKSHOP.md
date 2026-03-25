# HIQ Workshop — Saleor Storefront

**Your Saleor instance:** `https://_____.saleor.cloud/graphql/`
**Token (if needed):** `_______________`

---

## 1. Create a repo

Create a directory for your storefront.

---

## 2. Install the skill

```bash
npx skills add saleor/agent-skills
```

Select all the skills besides the "saleor-app" skill.

Make sure to toggle your IDE/AI harness of choice for proper skill formatting.

---

## 3. Run the builder

Run the skill like this in your IDE/CLI:

```
/storefront-builder 1
```

Follow the prompts for each step. When done with a step, move to the next:

```
/storefront-builder 2
/storefront-builder 3
```

**Step 1** — scaffold the project, install GraphQL deps, configure the Saleor API client
**Step 2** — pick your visual style, accent color, and typography
**Step 3** — product list page, product detail page, variant selector

---

## Tips

- The AI will ask for your Saleor API URL and channel slug in Step 1 — use the values above.
- Default channel slug is `default-channel` unless told otherwise.
- Each step stops and waits — never chains automatically.
