# OpenCart Knowledge Base Workflow

## Purpose

What this repository is for:
- a reusable OpenCart domain knowledge base built against a verified upstream baseline
- faster orientation for future tasks
- safer change planning through pre-mapped domain packs

What this repository is NOT:
- not a replacement for project-level verification
- not a guarantee that a live project matches the baseline
- not a source for inventing missing routes, methods, tables, or extension behavior

## Repository Structure

### Core Packs

- `opencart-core/` — stable domain packs for core OpenCart behavior.
- Current STANDARDS packs (rules for writing correct OpenCart code):
  - `coding-standards.md` — formatting, naming, file layout, official style rules
  - `mvc-l-architecture.md` — controllers/models/views/language, loaders, registry, routing internals
  - `permissions.md` — admin ACL: route → permission key resolution, the `access` / `modify` split,
    the core extension install/uninstall flow, escalation patterns, and the write-path audit recipe
  - `security.md` — SQL escaping, XSS, CSRF/`user_token`, sessions, hashing, uploads
  - `database-schema.md` — `DB_PREFIX`, schema ownership, idempotency, core table map
  - `settings-and-config.md` — the `setting` table, store scoping, key conventions, inert settings
  - `extension-development.md` — extension types, file layout, install/uninstall contract
  - `ocmod.md` — `install.xml`, search/add, position/offset, the modification cache, the `code` landmine
  - `release-and-packaging.md` — `.ocmod.zip` layout, installer mechanics, update-vs-reinstall strategy
  - `twig-theming.md` — Twig config, template paths, theme structure, asset registration
  - `seo-and-routing.md` — route resolution, `seo_url`, link generation, URL hygiene
  - `testing-and-verification.md` — static gates, structural audits, positive-testing a gate, evidence discipline
- Current DOMAIN MAPS (how core behaves):
  - `orders.md`
  - `mail.md`
  - `products.md`
  - `customers.md`
  - `api.md`
  - `cache.md`
  - `events.md`
  - `request-response.md`

### Overlays

- `themes/`
- `modules/`
- `projects/`

Purpose of overlays:
- store- or extension-specific behavior
- theme override maps
- module-specific deviations from core
- project-specific collision notes

### Templates

- `templates/domain-pack-template.md`

## Confidence Taxonomy

Every factual claim must carry or inherit one of these confidence levels.

### `[verified]`

Use only when:
- the source file was read in full, or
- the exact cited block was read directly from source

Examples:
- full model method behavior
- exact SQL columns from `CREATE TABLE`
- exact auth flow from a controller body

### `[baseline]`

Use only when:
- a targeted grep, function list, event row, or directory listing confirmed the claim
- but the full file/body was not read

Examples:
- controller method lists
- event presence
- guard-count observations
- directory contents

### `[exists]`

Use only when:
- the path was existence-checked
- content was not read

Also use for negative existence checks:
- file absent
- HTTP HEAD 404
- missing template/controller path

### `[needs-verification]`

Use when:
- the statement is informed inference
- standard OpenCart behavior is assumed but not confirmed against the current baseline
- live-project behavior may differ

Rule:
- never present `[needs-verification]` as an established fact

## Baseline Policy

Default upstream baseline:
- `https://github.com/19th19th/LiveStore`
- always use the latest release tag unless a pack explicitly documents another baseline

Current baseline used by the core packs:
- `v3.0.4.4` for the domain maps written before 2026-09-13 (`orders.md`, `mail.md`, `products.md`,
  `customers.md`, `api.md`, `cache.md`, `events.md`, `request-response.md`)
- `v3.0.4.5` for the standards packs authored on 2026-09-13 (`coding-standards.md`,
  `mvc-l-architecture.md`, `permissions.md`, `security.md`, `database-schema.md`,
  `settings-and-config.md`, `extension-development.md`, `ocmod.md`, `release-and-packaging.md`,
  `twig-theming.md`, `seo-and-routing.md`, `testing-and-verification.md`)

Raw source convention:
- `https://raw.githubusercontent.com/19th19th/LiveStore/{tag}/upload/{path}`

Important rule — every pack must state:
- `Version`
- `Source`
- `Verified on`

## Baseline Drift Log

### `v3.0.4.4` → `v3.0.4.5`

Measured on 2026-09-13 via the GitHub compare API: **39 commits, 49 changed files.**

Re-checked and found **byte-identical** between the two tags, so claims about them hold on either:
- `upload/admin/controller/startup/permission.php`
- `upload/admin/controller/extension/extension/module.php`
- `upload/admin/controller/extension/extension/theme.php`

Notable changes that touch existing pack domains (these claims were verified on `v3.0.4.4` and have NOT
been re-verified against `v3.0.4.5` — treat the affected statements as `[needs-verification]` until then):
- `upload/admin/controller/setting/setting.php`, `admin/view/template/setting/setting.twig` — settings domain
- `upload/admin/controller/common/filemanager.php` — security / upload domain
- `upload/admin/view/template/user/user_group_form.twig` — permission UI (the ACL mechanism itself is unchanged)
- `upload/admin/controller/catalog/product.php`, `admin/model/catalog/product.php`,
  `catalog/controller/product/product.php`, `catalog/model/catalog/product.php`,
  `catalog/controller/product/manufacturer.php` — products domain
- `upload/catalog/controller/account/login.php`, `catalog/controller/checkout/login.php` — customers domain
- `upload/catalog/view/theme/default/stylesheet/stylesheet.css` — theme assets

Removed in `v3.0.4.5` (a real example of the installer's no-delete asymmetry):
- `upload/catalog/language/en-gb/extension/module/google_hangouts.php`
- `upload/catalog/language/ru-ru/extension/module/google_hangouts.php`
- `upload/catalog/view/theme/default/template/extension/module/google_hangouts.twig`

A pack written against `v3.0.4.4` whose cited files are in the list above must be re-checked and its
`Version` updated before its claims are relied on.

## Re-Baselining Procedure

When a new LiveStore tag appears:

1. Get the newest tag
   - `gh api repos/19th19th/LiveStore/tags --jq '.[0].name'`

2. Compare the new tag against the current documented baseline.

3. Re-check each core pack only where drift matters:
   - routes
   - method lists
   - SQL schema
   - event rows
   - known LiveStore extras
   - API directory layout

4. Update:
   - `Version`
   - `Source`
   - `Verified on`
   - any changed facts
   - any changed confidence tags if verification depth changed

5. If drift is partial:
   - do not silently keep old claims
   - downgrade uncertain claims to `[needs-verification]` until re-checked

## Pack Authoring Standard

Each domain pack should follow this section order:

1. `Scope`
2. `Baseline`
3. `Confidence Levels`
4. `Main Routes`
5. `Core Files`
6. `Data Flow`
7. `Database`
8. `Events`
9. `API Links`
10. `Mail Links`
11. `OCMOD Hotspots`
12. `Theme Override Risks`
13. `Common Extension Collision Zones`
14. `Verification Notes`

## Writing Rules

### What to do
- separate confirmed facts from inference
- explicitly call out LiveStore-specific extras
- cross-link related packs where behavior crosses domains
- keep `Verification Notes` stricter than the prose above them
- downgrade confidence when evidence is weaker than the sentence sounds

### What not to do
- do not guess controller bodies from route names
- do not guess method signatures from memory
- do not treat existence as content verification
- do not treat one LiveStore build extra as universal OpenCart behavior
- do not collapse project-specific behavior into core packs

## Core vs Overlay Rule

Put facts into `opencart-core/` only when they describe:
- standard OpenCart behavior, or
- confirmed LiveStore baseline behavior that belongs to the upstream build itself and is reusable across projects

Use overlays when behavior depends on:
- a specific theme
- a specific module
- a specific store project
- a custom OCMOD/VQMod layer
- local schema drift not present in the upstream baseline

## LiveStore-Specific Extras Rule

If a fact is present in LiveStore but not vanilla OpenCart:
- state it explicitly
- mark it as build-specific
- add a re-verification warning for non-LiveStore baselines

Current known examples (as documented in the core packs at `v3.0.4.4`):
- order tables `oc_order_shipment` + `oc_shipping_courier` (and `sale/order::shipping`) — see `orders.md`
- `product.noindex` — see `products.md`
- `product_description.meta_h1` — see `products.md`
- `product_to_category.main_category` — see `products.md`
- product relation extras `product_related_article` / `product_related_wb` / `product_related_mn` — see `products.md`
- `config_seo_pro` / `seopro` cache handling — see `products.md`
- consolidated `api/payment` + `api/shipping` controllers instead of the stock 3.0.x
  `*_address` / `*_method` split — see `api.md`

## Project-Level Verification Rule

This repository speeds up analysis, but does NOT remove the need to verify the actual target project.

Before implementation in a real project, always re-check:
- active theme overrides
- extra database columns/tables
- OCMOD/VQMod/event collisions
- module-specific route/controller replacements
- baseline drift from the current LiveStore tag

## OpenCart Hard Mode

Use this repository under a strict no-guessing rule.

- Do not guess routes, methods, models, Twig variables, XML anchors, database tables, columns, events, or config keys.
- Read the real target files before treating a technical point as confirmed.
- Check the full affected chain: route, controller, model, language, view, OCMOD or install XML, events, and theme override.
- Treat OCMOD as fragile and verify exact single-line search anchors against the real source before editing.
- If a point is not verified against source, baseline, schema, or existence check, keep it unverified rather than inventing it.
- Before implementation, check collision zones: OCMOD, theme overrides, events, and third-party extensions.
- Before completion, verify that the final logic is not duplicated, overridden, or injected elsewhere.
- In OpenCart work, plausibility is not correctness.

Rule of interpretation:
- this knowledge base is a verified acceleration layer
- it is not a license to skip project-specific verification
- confidence tags remain part of the fact itself, not optional decoration

## Update Policy

Update a pack when:
- a claim becomes newly verified
- a baseline changes
- a real project reveals a stable reusable fact that was then verified against the upstream baseline or intentionally classified into the correct overlay
- a previously confident statement turns out to be weaker than claimed

Do not update a pack when:
- the fact is only project-local and belongs in an overlay
- the statement is still inference without source support
- the evidence level is unclear

## Self-Correction and Gap-Fill

The knowledge base is expected to improve during real work. Two obligations apply.

### Fix errors immediately

If, while verifying against a real source (LiveStore baseline, local project files, live DB), you
find a pack claim that is wrong, outdated, or misleading, fix it in the same session.

Qualifying errors:
- method signature does not match the real source
- table/column name wrong or missing
- event mapping incorrect or incomplete
- confidence tag too high for the actual evidence
- a LiveStore build extra documented as vanilla OpenCart, or vice versa
- a path that does not exist at the referenced baseline

Fix protocol:
1. Confirm the correct fact against the real source first — never replace one guess with another.
2. Edit the pack.
3. Adjust the confidence tag to reflect the new verification depth.
4. If the fix is significant (wrong signature, wrong schema, missing critical event), state it in
   the commit message / task output so the user is aware.

### Fill gaps on discovery

If during a task you had to read something the pack should already have documented (a commonly
needed method, a schema detail, an event mapping, a config default, a controller→model→view chain,
an OCMOD hotspot, a collision zone), add it back.

Fill protocol:
1. Finish the current task first — do not context-switch mid-implementation.
2. Add the missing fact with the correct confidence tag (prefer `[verified]` when read from source
   during the task).
3. Commit the update so future sessions benefit.

### Guardrail

Self-correction must never lower accuracy. Any edit to the base must be backed by a source read,
grep, or existence check — the same evidence bar as the original claim. Do not "correct" a pack
from memory or from unverified inference.

## Commit Policy

Recommended commit boundaries:
- one commit per completed pack
- one commit for workflow/docs
- one commit per meaningful overlay pack or overlay batch

Reason:
- keeps verification history auditable
- makes confidence regressions easier to trace

## Suggested Future Expansion

Next likely overlays:
- `themes/`
- `modules/`
- `projects/`

Good first overlay candidates:
- a major theme actually used in work
- a frequently touched module family
- a real project with persistent deviations from core

## Short Operating Principle

Use this knowledge base to get oriented faster.
Do not use it as permission to skip source verification in the real target project.
Confidence tags are part of the data, not decoration.

## Mandatory Engineering & Self-Review Loop (Автономный цикл разработки и ревью)

При создании или модификации решений OpenCart:
1. **Grounded Verification & Native Modules Alignment:** Никаких фантазий и предположений — обязательная сверка архитектуры, данных, методов и таблиц с аналогичными штатными модулями/расширениями из поставки той же версии OpenCart и апстрима `LiveStore v3.0.4.4`.
2. **Lifecycle & Migrations:** Поддержка `install`, `upgradeSchema` (`ALTER TABLE`), `migration_map` для предотвращения коллизий ID, идемпотентность.
3. **Autonomous Self-Review:** Обязательный запуск мини-код-ревью (P0/P1/P2) с реальным исполнением тестов. Если найдены ошибки — цикл исправлений и повторных проверок продолжается до достижения 0 дефектов перед финальным отчётом.
