# OpenCart standards (Tom-Opencart knowledge base)

The authoritative entry point is `AGENTS.md` at the repository root, which carries the non-negotiable
rules, the pack index, the confidence taxonomy and the baseline policy.

This file exists only so tools that look for `.github/copilot-instructions.md` find the pointer.

Before writing or reviewing OpenCart code:

1. Read `AGENTS.md`.
2. Open the relevant pack in `opencart-core/` for the domain you are touching.
3. Verify every claim against the actual project files; the project always takes precedence.

Condensed rules:

- Admin authorization is two-tier: the router enforces only `access`, so every write path must check
  `modify` itself, and the check must dominate the write.
- Schema changes run from `install()`/`uninstall()`, never from a page load.
- Never change `<code>` in `install.xml` — it duplicates the modification instead of replacing it.
- Every raw query uses `DB_PREFIX` and `$this->db->escape()`; ids are cast with `(int)`.
- Static gates prove syntax and structure, never behavior.
- Tag claims `[verified]`, `[baseline]`, `[exists]` or `[needs-verification]`; never guess core behavior.
