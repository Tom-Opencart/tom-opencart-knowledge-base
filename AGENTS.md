# Tom OpenCart Knowledge Base Instructions

Apply these instructions when using this repository as a knowledge source for OpenCart work.

## Purpose

This repository is a reusable OpenCart knowledge layer.

It is designed to:
- reduce repeated exploration of recurring OpenCart domains
- preserve verified domain notes across tasks
- provide a stable starting map before reading the current project in detail

It is **not** designed to replace verification against the real current project.

## Repository Layers

- `opencart-core/`
  Stable baseline domain packs for recurring OpenCart logic
- `themes/`
  Theme-specific overlays
- `modules/`
  Module and extension overlays
- `projects/`
  Store-specific overlays
- `templates/`
  Reusable note templates

## Required Workflow for AI Assistants

1. Identify the active domain first.
   - Examples: orders, mail, products, customers, API, checkout
2. Check whether a relevant note already exists in this repository.
3. Use the existing note as the starting map for the task.
4. Verify the real current project before making code, XML, Twig, SQL, or schema claims.
5. If a verified gap is discovered during the task, update the relevant note instead of leaving the discovery implicit.

## Verification Rule

- Treat this repository as an acceleration layer, not as the final source of truth.
- Current project files, current theme overrides, current XML modifiers, current database schema, and current module stack always take precedence.

## Baseline Rule

- When documenting baseline OpenCart core behavior, prefer the latest available LiveStore release if LiveStore is the active baseline.
- Do not hard-pin the repository to one historical LiveStore version unless a task explicitly requires that version.

## Update Standard

When adding or editing a note:
- prefer exact file paths
- prefer exact route names
- prefer exact method names
- prefer exact table names and important columns
- record conflict hotspots and override risks
- clearly distinguish verified facts from placeholders or pending verification
