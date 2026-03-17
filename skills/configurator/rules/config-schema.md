# Config Schema

`config.yml` is the single source of truth for your Saleor store. Each top-level key maps to an entity type. All sections are optional — omit a section to leave that part of the store unchanged.

## Top-Level Structure

```yaml
shop:          # Global store settings (singleton)
channels:      # Sales channels
taxClasses:    # Tax classifications
productTypes:  # Product templates
pageTypes:     # CMS page templates
attributes:    # Reusable attribute definitions
categories:    # Product category hierarchy
collections:   # Product groupings
warehouses:    # Fulfillment locations
shippingZones: # Geographic shipping rules
products:      # Product catalog
menus:         # Navigation menus
models:        # Custom data models
```

## Key Rules

- **Deploy is additive only** — Configurator does not delete entities; it only creates and updates
- **Omitted and empty sections are both ignored** — neither causes any remote changes
- **Cross-references use the target entity's identifier** — products reference `productType` by name, `category` by slug (see `identity-strategies`)
- **Validation runs before any mutations** — duplicate slugs/names or broken references abort the deploy

## Minimal Example

```yaml
shop:
  name: "My Store"
  defaultCurrency: USD
  defaultCountry: US

productTypes:
  - name: "Physical Product"
    kind: NORMAL

categories:
  - name: "Electronics"
    slug: "electronics"
```

## Full Schema Reference

The authoritative schema is defined as Zod in the Configurator source:

- **Schema source**: [`src/modules/config/schema/schema.ts`](https://github.com/saleor/saleor-configurator/blob/main/src/modules/config/schema/schema.ts)
- **Entity reference**: [`docs/ENTITY_REFERENCE.md`](https://github.com/saleor/saleor-configurator/blob/main/docs/ENTITY_REFERENCE.md)

## Anti-Patterns

❌ Don't mix up cross-reference identifier types — see `identity-strategies` for which entity uses slug vs name
