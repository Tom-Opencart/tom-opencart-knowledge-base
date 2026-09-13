# OpenCart Standards — knowledge base

A verified **standards and knowledge layer for OpenCart 3.x / ocStore / LiveStore**, built against a
documented upstream baseline. It is designed to be pointed at from **any AI agent**: the rules an agent
needs are in the packs, and the entry points below make the repository discoverable automatically.

It is an acceleration layer, **not** a replacement for verifying the real project in front of you.
Project files, theme overrides, XML modifiers, live database schema, and the installed module stack always
take precedence over anything here.

## Start here

**`AGENTS.md`** is the authoritative entry point: non-negotiable rules, the full pack index, the
confidence taxonomy, and the baseline policy. Every other entry file is a pointer to it.

## Use it in any agent

| Agent / tool | Entry point | Setup |
| --- | --- | --- |
| Codex, DSH, Cursor, Copilot, most tools | `AGENTS.md` | Read automatically from the repo root. |
| Claude Code | `CLAUDE.md` | Present as a pointer to `AGENTS.md`. |
| Skill-capable agents | `SKILL.md` | Load the repository as a skill. |
| Cursor (scoped rules) | `.cursor/rules/opencart-standards.mdc` | `alwaysApply: true`, applies to PHP/Twig/XML. |
| GitHub Copilot | `.github/copilot-instructions.md` | Present as a pointer to `AGENTS.md`. |
| Anything else | — | Paste `AGENTS.md`, or tell the agent: *"Read `AGENTS.md` and the packs it names in `https://github.com/Tom-Opencart/tom-opencart-knowledge-base`, then verify every claim against this project."* |

Minimum bootstrap prompt:

> Before writing any OpenCart code, read `AGENTS.md` and the relevant pack in `opencart-core/` from the
> Tom-Opencart knowledge base, then verify every claim against the actual project files. Tag unverified
> statements `[needs-verification]`.

## What is included

### `opencart-core/` — standards

| Pack | Covers |
| --- | --- |
| `coding-standards.md` | formatting, naming, file layout, official style rules |
| `mvc-l-architecture.md` | controllers, models, views, language, loaders, registry, routing internals |
| `permissions.md` | admin ACL, `access` vs `modify`, write-path audits, demo/restricted admin users |
| `security.md` | SQL escaping, XSS, CSRF/`user_token`, sessions, hashing, uploads |
| `database-schema.md` | `DB_PREFIX`, schema ownership, idempotency, core table map |
| `settings-and-config.md` | `setting` table, store scoping, module/theme key conventions, inert settings |
| `extension-development.md` | extension types, file layout, install/uninstall contract |
| `ocmod.md` | `install.xml`, search/add, position/offset, the modification cache, the `code` landmine |
| `release-and-packaging.md` | `.ocmod.zip` layout, installer mechanics, update-vs-reinstall strategy |
| `twig-theming.md` | Twig config, template paths, theme structure, asset registration |
| `seo-and-routing.md` | route resolution, `seo_url`, link generation, URL hygiene |
| `testing-and-verification.md` | static gates, structural audits, positive-testing a gate, evidence discipline |

### `opencart-core/` — behavior maps

`events.md`, `cache.md`, `request-response.md`, `products.md`, `orders.md`, `customers.md`, `mail.md`,
`api.md` — routes, core files, tables and data flow for those domains.

### Overlays

- `themes/` — theme-specific behavior and override maps
- `modules/` — module/extension overlays
- `projects/` — store-specific notes

## Rules that matter most

- **Admin authorization is two-tier.** The route permission key is `parts[0]/parts[1]` (plus `parts[2]`
  only for the `extension/*` list), so the **method name is never part of the key** — all methods of a
  controller share one permission row. The router enforces only `access`; `modify` is never enforced
  centrally, so every write path must check it itself, and the check must dominate the write.
- **A restricted or demo group normally has `access`.** Any write reachable by opening a page — a GET
  `index()` that writes, installs a sibling extension, runs DDL, or grants permissions — is an
  escalation hole.
- **Schema changes belong in `install()`/`uninstall()`**, never on a page load. `CREATE TABLE IF NOT
  EXISTS` does not add columns to an existing table, so a guarded `ALTER` inside `install()` is the only
  repair path for a legacy table.
- **Never change `<code>` in `install.xml`.** The modification row is keyed by `code`; a changed value
  creates a *second* modification with the same search blocks, both apply, and the store can die.
- **Static gates prove syntax and structure, never behavior.**

## Confidence taxonomy

Every factual claim carries or inherits one of: `[verified]` (source read directly), `[baseline]`
(grep/list confirmed only), `[exists]` (existence-checked only), `[needs-verification]` (inference —
never presented as established fact).

## Baseline

- LiveStore `v3.0.4.5`
- Source: `https://github.com/19th19th/LiveStore`
- Raw convention: `https://raw.githubusercontent.com/19th19th/LiveStore/{tag}/upload/{path}`

Each pack states its own `Version`, `Source` and `Verified on`.

## Operating rules and contributing

The repository workflow, the pack authoring standard, the confidence taxonomy, and the re-baselining
procedure are defined in [`WORKFLOW.md`](./WORKFLOW.md).

If you verify a fact that contradicts or extends a pack, update that pack in the same session and state
which baseline source and lines you verified it from. Prefer exact file paths, route names, method names,
table names and important columns, and record collision and override risks.
