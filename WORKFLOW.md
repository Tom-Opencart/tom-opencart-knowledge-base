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
- Current packs:
  - `orders.md`
  - `mail.md`
  - `products.md`
  - `customers.md`
  - `api.md`

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
- `v3.0.4.4`

Raw source convention:
- `https://raw.githubusercontent.com/19th19th/LiveStore/{tag}/upload/{path}`

Important rule — every pack must state:
- `Version`
- `Source`
- `Verified on`

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
