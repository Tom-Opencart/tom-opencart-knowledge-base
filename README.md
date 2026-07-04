# OpenCart Knowledge Base

A public OpenCart domain knowledge base built against a verified upstream LiveStore baseline.

## What is included

Core domain packs in `opencart-core/`:

- `orders.md`
- `mail.md`
- `products.md`
- `customers.md`
- `api.md`

These packs document verified or explicitly-scoped baseline behavior for the current upstream reference build.

## Operating Rules

The repository workflow, confidence taxonomy, re-baselining rules, and core-vs-overlay policy are defined in [`WORKFLOW.md`](./WORKFLOW.md).

## Baseline

Current documented baseline:

- LiveStore `v3.0.4.4`
- Source: `https://github.com/19th19th/LiveStore`

## Notes

This repository is meant to speed up analysis and reduce repeated orientation work.

It does **not** replace project-level verification. Real projects must still be checked for:
- theme overrides
- OCMOD/VQMod/event collisions
- extra schema drift
- module- or project-specific behavior

## Future Expansion

Overlay packs will be added as real work requires them, under:
- `themes/`
- `modules/`
- `projects/`
