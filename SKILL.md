---
name: opencart-standards
description: OpenCart 3.x / ocStore / LiveStore development standards and verified core behavior. Use when writing, reviewing, auditing, or shipping any OpenCart code or package — PHP controllers/models, Twig themes and templates, admin modules and extensions, install.xml / OCMOD modifications, database schema and migrations, admin permissions (access vs modify), security (SQL escaping, XSS, user_token), settings and config, SEO routes, or .ocmod.zip release packaging. Also use for release-readiness checks, permission audits for restricted or demo admin users, and any question about how OpenCart core actually behaves.
---

# OpenCart Standards

A verified standards and knowledge layer for OpenCart 3.x, built against the LiveStore upstream
baseline. Read the entry point first, then open only the packs the task needs.

## Entry point

Read **`AGENTS.md`** in this repository. It carries the non-negotiable rules, the full pack index, the
confidence taxonomy, and the baseline policy. Do not work from this file alone.

## How to route a task

| Task | Open |
| --- | --- |
| Writing or reviewing PHP/Twig/JS style | `opencart-core/coding-standards.md` |
| Controller/model/view/language structure, loaders, routing internals | `opencart-core/mvc-l-architecture.md` |
| Admin authorization, `access` vs `modify`, permission audits, demo or restricted users | `opencart-core/permissions.md` |
| SQL escaping, XSS, CSRF/`user_token`, sessions, hashing, file uploads | `opencart-core/security.md` |
| Database work, schema ownership, migrations, idempotency | `opencart-core/database-schema.md` |
| Settings, config, store scoping, module/theme setting keys | `opencart-core/settings-and-config.md` |
| Building or changing an extension/module/theme | `opencart-core/extension-development.md` |
| `install.xml`, OCMOD search/replace, modification cache | `opencart-core/ocmod.md` |
| Building, updating, or shipping a package | `opencart-core/release-and-packaging.md` |
| Twig, templates, theme structure, assets | `opencart-core/twig-theming.md` |
| Routes, SEO URLs, link generation | `opencart-core/seo-and-routing.md` |
| Auditing, gating, or reporting on a change | `opencart-core/testing-and-verification.md` |
| Events, cache, request/response | `opencart-core/events.md`, `cache.md`, `request-response.md` |
| Products, orders, customers, mail, API domains | `opencart-core/products.md`, `orders.md`, `customers.md`, `mail.md`, `api.md` |

## The rules that matter most

- The admin route permission key excludes the method name, and the router enforces only `access`. Every
  write path must check `modify` itself. A restricted group usually has `access` and must not have
  `modify` — so any write reachable by opening a page is an escalation hole.
- Schema changes belong in `install()`/`uninstall()`, never on a page load.
- Never change `<code>` in `install.xml` — it creates a duplicate modification instead of replacing the
  existing one.
- Static gates prove syntax and structure, never behavior.

## Verification rules

- Every factual claim carries a confidence tag: `[verified]`, `[baseline]`, `[exists]`, or
  `[needs-verification]`. Never present `[needs-verification]` as established fact.
- Always verify against the actual project in front of you. Project files, theme overrides, XML
  modifiers, live schema, and the installed module stack take precedence over this repository.
- Never guess routes, methods, signatures, columns, events, Twig variables, or XML anchors.

## Maintenance

If you verify a fact that contradicts or extends a pack, update that pack in the same session and state
which baseline source and lines you verified it from. Structure and authoring rules: `WORKFLOW.md`.
