# OpenCart Standards

This repository is the OpenCart standards and knowledge layer. The full entry point — non-negotiable
rules, pack index, confidence taxonomy and baseline policy — is in **`AGENTS.md`**.

Read `AGENTS.md` before making any claim about OpenCart core behavior or writing OpenCart code, then open
only the packs related to the task.

Quick rules:

- The admin route permission key excludes the method name, and the router enforces only `access`; every
  write path must check `modify` itself.
- Schema changes belong in `install()`/`uninstall()`, never on a page load.
- Never change `<code>` in `install.xml`.
- Static gates prove syntax and structure, never behavior.
- Tag every factual claim `[verified]`, `[baseline]`, `[exists]`, or `[needs-verification]`, and verify
  against the real project before relying on anything here.

See `AGENTS.md` for the pack index and `SKILL.md` for task routing.
