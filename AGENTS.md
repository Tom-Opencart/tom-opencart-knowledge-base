# OpenCart Standards — agent instructions

This repository is a **standards and knowledge layer for OpenCart 3.x / ocStore / LiveStore work**, built
against a verified upstream baseline. It is designed to be pointed at from **any** AI agent: read this
file first, then open only the packs you need.

It is an acceleration layer, **not** a substitute for verifying the actual project in front of you.

---

## 1. How to use this repository from an agent

| Your agent reads | What to do |
| --- | --- |
| `AGENTS.md` (Codex, Cursor, Copilot, DSH, and most tools) | Nothing — this file is the entry point. |
| `CLAUDE.md` (Claude Code) | Already present as a pointer to this file. |
| `SKILL.md` (skill-capable agents) | Load the repo as a skill. |
| `.cursor/rules/` | Already present as a pointer. |
| `.github/copilot-instructions.md` | Already present as a pointer. |
| Anything else | Paste this file, or tell the agent: "Read `AGENTS.md` and the packs it names in `https://github.com/Tom-Opencart/tom-opencart-knowledge-base`." |

**Minimum bootstrap prompt for any agent:**

> Before writing any OpenCart code, read `AGENTS.md` and the relevant pack in
> `opencart-core/` from the Tom-Opencart knowledge base, then verify every claim against the
> actual project files. Tag unverified statements.

---

## 2. Non-negotiable rules

These apply to every OpenCart task. They are the condensed result of the packs below; the packs carry
the evidence and the line references.

**Authorization**
- Admin authorization is two-tier, and the router enforces only the first tier. The route permission key
  is `parts[0]/parts[1]` (plus `parts[2]` only for the `extension/*` list), so the **method name is never
  part of the key** — all methods of a controller share one permission row.
- `access` = may open the page (enforced centrally). `modify` = may change data (**never** enforced
  centrally). Every write path must check `modify` itself, and the check must **dominate** the write.
- A route-level `access` grant is **not** write protection. A demo or restricted group normally has
  `access` and must not have `modify`.
- `install()` / `uninstall()` must check `modify` on the core list key
  (`extension/extension/module`, `extension/extension/theme`) — core's own check runs only inside the
  list controller, and a direct route call bypasses it.
- Escalation holes to hunt: a GET page (`index()`/`form()`) that writes, installs a sibling extension,
  runs DDL, or calls `addPermission()`; an `ensure*Access()` helper reachable from a page; a
  `validate*()` method that is defined but never called.

**Schema and data**
- Schema changes run from `install()`/`uninstall()` — **never** from a page load. Read paths must not
  execute DDL.
- `CREATE TABLE IF NOT EXISTS` does not add columns to an existing table; a guarded `ALTER` inside
  `install()` is the only mechanism that repairs a legacy table.
- Installers must be idempotent; `uninstall()` must not destroy user data indiscriminately.
- If a read path truly needs a lifecycle-skew check, make it once-per-request with `static $checked`,
  never a query per call.
- Every raw query uses `DB_PREFIX`; strings go through `$this->db->escape()`; ids are cast with `(int)`.

**Packaging**
- **Never change `<code>` in `install.xml`.** The modification row is identified by `code`; a changed
  value creates a *second* modification with the same search blocks, both apply, and the store can die.
- The installer copies but never deletes store files, so a partial archive cannot remove a file.
- Every release ships a full archive as the base; a diff archive is update-only and must never be
  uninstalled.

**Frontend**
- Theme includes/extends use the full template path with an explicit `.twig` extension.

**Honesty**
- Static gates prove syntax and structure, **never** behavior. Never claim a live-store outcome you did
  not observe.

---

## 3. Pack index — open only what the task needs

### Standards (how to write correct OpenCart code)

| Pack | Use when |
| --- | --- |
| `opencart-core/coding-standards.md` | formatting, naming, file layout, official style rules |
| `opencart-core/mvc-l-architecture.md` | controllers/models/views/language, loaders, registry, routing internals |
| `opencart-core/permissions.md` | admin ACL, `access` vs `modify`, write-path audits, demo/restricted users |
| `opencart-core/security.md` | SQL escaping, XSS, CSRF/`user_token`, sessions, hashing, uploads |
| `opencart-core/database-schema.md` | `DB_PREFIX`, schema ownership, idempotency, core table map |
| `opencart-core/settings-and-config.md` | `setting` table, store scoping, module/theme key conventions, inert settings |
| `opencart-core/extension-development.md` | extension types, file layout, install/uninstall contract |
| `opencart-core/ocmod.md` | `install.xml`, search/add, position/offset, the `code` landmine |
| `opencart-core/release-and-packaging.md` | `.ocmod.zip` layout, installer mechanics, update strategy |
| `opencart-core/twig-theming.md` | Twig config, template paths, theme structure, asset registration |
| `opencart-core/seo-and-routing.md` | route resolution, `seo_url`, link generation, URL hygiene |
| `opencart-core/testing-and-verification.md` | static gates, structural audits, positive-testing a gate, evidence discipline |

### Behavior maps (how core actually behaves)

| Pack | Domain |
| --- | --- |
| `opencart-core/events.md` | event dispatch, orphan events, module lifecycle contract |
| `opencart-core/cache.md` | cache TTL semantics and expiry pitfalls |
| `opencart-core/request-response.md` | QUERY_STRING handling, redirect and URL hygiene |
| `opencart-core/products.md`, `orders.md`, `customers.md`, `mail.md`, `api.md` | core domain maps: routes, files, tables, data flow |

### Overlays

- `themes/` — theme-specific behavior and override maps
- `modules/` — module/extension overlays
- `projects/` — store-specific notes

---

## 4. Confidence taxonomy — every claim carries one

- `[verified]` — the source file was read in full, or the exact cited block was read directly.
- `[baseline]` — a targeted grep, method list, or directory listing confirmed it; the body was not read.
- `[exists]` — existence-checked only (also for negative existence checks).
- `[needs-verification]` — informed inference. **Never present this as an established fact.**

Never guess routes, methods, signatures, columns, events, Twig variables, XML anchors, or override
behavior. If it was not verified from source, schema, grep, or an existence check, keep it explicitly
unverified.

---

## 5. Baseline policy

- Default upstream baseline: `https://github.com/19th19th/LiveStore` — use the latest release tag.
- Raw source convention: `https://raw.githubusercontent.com/19th19th/LiveStore/{tag}/upload/{path}`
- Every pack states `Version`, `Source` and `Verified on`.
- Current project files, theme overrides, XML modifiers, database schema, and the installed module stack
  **always** take precedence over anything here.

---

## 6. Repository maintenance

When you discover a verified fact that contradicts or extends a pack, update the pack in the same
session — do not leave the discovery implicit.

- Prefer exact file paths, route names, method names, table names and important columns.
- Record collision hotspots and override risks.
- Keep `Verification Notes` stricter than the prose above them.
- Downgrade a claim's confidence tag when the evidence is weaker than the sentence sounds.
- Repository structure, the pack authoring standard, and the re-baselining procedure live in
  `WORKFLOW.md`.

### OpenCart Hard Mode

This repository is an acceleration layer, not permission to guess. Hard Mode also means: a clean static
run is not evidence of correct behavior, and a gate that has never failed has not been shown to work.
