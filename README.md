# Tom OpenCart Knowledge Base

Public knowledge base for OpenCart domain mapping, baseline flows, theme overlays, module overlays, and project-specific notes.

## Purpose

This repository is meant to reduce repeated exploration work across OpenCart tasks.

It is split into two layers:

1. `opencart-core/`
   Stable baseline knowledge about OpenCart itself.
2. `themes/`, `modules/`, `projects/`
   On-demand overlays for specific themes, extensions, and real stores.

The base is intended to accelerate investigation, not replace verification against the current project.

## Structure

```text
opencart-core/
  orders.md
  mail.md
  products.md
  customers.md
  api.md
themes/
  README.md
modules/
  README.md
projects/
  README.md
templates/
  domain-pack-template.md
```

## Working Rules

- `opencart-core/` should describe stable domain flows and known integration points.
- `themes/` should capture theme-specific overrides, bundled modules, and rendering risks.
- `modules/` should capture module-specific controllers, models, views, XML, SQL, and event hooks.
- `projects/` should capture store-specific overlays only when a real task requires them.
- Every note should prefer exact paths, routes, method names, table names, and conflict hotspots.
- Baseline notes should record the source version they were checked against.

## How AI Assistants Should Use This Repository

This repository is meant to be used by Codex, Claude Code, OpenCode, Cursor, Gemini CLI, and other LLM-based coding assistants as a reusable OpenCart knowledge layer.

Recommended workflow:

1. Identify the active OpenCart domain first.
   - Example: orders, mail, products, customers, API, checkout
2. Check whether a relevant note already exists here.
3. Use that note as the starting map for the task.
4. Verify the real current project before making code, XML, SQL, Twig, or schema claims.
5. If the task reveals a verified gap, update the relevant note instead of leaving the discovery implicit.

Important rule:

- This repository accelerates investigation.
- It does **not** replace targeted verification against the current project.

## Suggested Growth Order

1. Orders
2. Mail
3. Products
4. Customers
5. API
6. Checkout
7. Events
8. Modifiers

## Source Strategy

Core notes should be built from a verified upstream baseline such as:

- OpenCart core
- the latest available LiveStore release when LiveStore is the active working baseline

Theme and module notes should be added only when a real task touches them.

## License

MIT
