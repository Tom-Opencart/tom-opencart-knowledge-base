# OpenCart Coding Standards (OC3 / LiveStore)

## Scope

The formatting, naming and structural rules for PHP, Twig, JavaScript, CSS and language files in an
OpenCart 3.x codebase, **plus** what the LiveStore baseline actually does in practice where the two
disagree. Does not cover architecture (see `mvc-l-architecture.md`) or security (see `security.md`).

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Official documented standard: `https://github.com/opencart/opencart/wiki/Coding-Standards`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[official]`: a rule quoted from the OpenCart wiki/standard document
- `[baseline]`: confirmed by a measured census over baseline files
- `[needs-verification]`: inference

---

## 1. The official documented rules (`[official]`)

Quoted from the OpenCart coding standard document (retrieved 2026-09-13):

**File types & encoding**
- PHP files use the `.php` extension; **all view/template files use `.twig`**.
- Line feeds: the repo is LF-managed, converted to the native environment on clone.

**PHP tags**
- Short opening tags and ASP tags are not supported. Use lowercase `<?php`.
- The document states closing tags were required before 2.0 and that **from 2.0 on, PHP files no longer
  have a closing tag**.

**Indentation**
- **PHP files must be indented using the TAB character. 4-space tabs are not supported.**
- **HTML in `.twig` template files must be indented using 2 spaces**, not 4 spaces and not TABs.
- **JavaScript must be indented using the TAB character.**

**Spacing**
- `if`, `while`, `for` get a space before the bracket: `if () {` — never `if(){`.
- `else` gets spacing around the braces: `} else {` — never `}else{`.
- Type casting has no space before the variable: `(int)$var` — never `(int) $var`.
- Assignment has a space before and after `=`: `$var = 1;` — never `$var=1;`.

**Whitespace**
- No trailing whitespace after code or on empty lines.
- After a closing PHP tag, remove all whitespace.

**New lines / braces**
- Opening curly braces never go on their own line — 1TBS. `if (...) {`, `class X extends Y {`,
  `public function f() {`.

**Naming**
- Files: lower case, words separated by underscores.
- Classes and methods: camelCase (`class ModelExampleExample`, `public function addExample()`).
- **"A method scope should always be cast"** — every method declares `public`/`protected`/`private`.
- Helper functions: lower case with underscores.
- PHP variables: lower case with underscores.
- User-defined constants: upper case (`define('MY_VAR', ...)`).
- Language constants `true`/`false`/`null`: lower case.
- HTML class names and ids: hyphenated, not underscored (`my-class`, not `my_class`).

**Tooling the standard references** (`[official]`, tools not verified present in this baseline):
`php tools/php-cs-fixer.phar fix -v` and `php tools/phpstan.phar`.

---

## 2. What the baseline actually does (`[baseline]`, measured)

Census over 84 PHP files and 12 `.twig` files sampled from LiveStore `v3.0.4.5`:

| Rule | Official | Measured in baseline | Verdict |
| --- | --- | --- | --- |
| PHP indentation | TAB | **84 / 84 files use TAB** | followed |
| PHP closing tag | none from 2.0 | **0 files contain `?>`** | followed |
| Short echo `<?=` | not supported | **0 files** | followed |
| Method naming | camelCase | **213 camelCase, 0 snake_case** | followed |
| Method scope always declared | required | **4 methods without a modifier** | violated in-baseline |
| PHP 4-space indentation | not supported | **9 files contain 4-space-indented lines** | violated in-baseline |
| Twig indentation | 2 spaces | **871 lines TAB vs 71 lines 2-space vs 101 lines 4-space** | **NOT followed** |

### Twig indentation — the important contradiction (`[baseline]`)

Measured per file (lines beginning with each indent):

| File | 2 spaces | TAB | 4 spaces |
| --- | --- | --- | --- |
| `admin/view/template/blog/article_list.twig` | 0 | 201 | 0 |
| `admin/view/template/blog/review_list.twig` | 0 | 172 | 0 |
| `admin/view/template/blog/article_form.twig` | 17 | 135 | 11 |
| `admin/view/template/blog/category_form.twig` | 15 | 27 | 8 |
| `catalog/view/theme/default/template/account/address_form.twig` | 3 | 109 | 12 |
| `catalog/view/theme/default/template/account/account.twig` | 6 | 0 | 12 |
| `catalog/view/theme/default/template/account/download.twig` | 3 | 0 | 12 |

**Consequence — the documented "2 spaces in .twig" rule is NOT the practice.** TAB is the majority
convention, 4 spaces is common in catalog templates, and 2 spaces is the minority. Files are internally
mixed.

**Consequence — never mass-reformat a template to satisfy the documented rule.** Two concrete reasons:

1. **It collides with OCMOD.** A modification's `<search>` block matches EXACT text in the target file
   (see `opencart-core/ocmod.md`). Changing the leading whitespace of a block that a shipped modifier
   searches for breaks that modifier silently — it stops applying, or worse, applies at the wrong offset.
2. **It destroys reviewability.** Reformatting a 400-line template produces a diff of the entire file and
   hides the real change.

**The practical rule: match the indentation of the file you are editing, and keep new files consistent
with their own directory.** When a file is already mixed, follow the dominant indent of that file.

---

## 3. Practical rules for new code

- **PHP**: TAB indentation, no closing `?>`, no short tags, 1TBS braces, `(int)$var`, spaces around `=`,
  `} else {`, no trailing whitespace, LF line endings.
- **Methods**: always declare a scope keyword; camelCase names. `[official]` + `[baseline]` (4 baseline
  violations show this is a real, recurring slip).
- **Variables**: snake_case, lower case.
- **Files**: lower case with underscores; `.php` for code, `.twig` for templates.
- **Twig**: match the file; do not reformat; keep `<script>`/`<style>` bodies JS/CSS-valid.
- **Language files**: `$_['key']` lowercase snake_case keys, one assignment per line.

## 4. ENFORCEABLE CHECKS

Machine-checkable (a checker can decide these deterministically):

1. Every changed `.php` passes `php -l`.
2. No `?>` at end of file; no `<?=` anywhere.
3. No line with trailing whitespace.
4. No file with mixed CRLF/LF (the shipped artifact must be LF).
5. Every `function` declaration has an explicit `public`/`private`/`protected`.
6. Method names match `[a-z][A-Za-z0-9]*` (camelCase); variable assignment names match `[a-z_][a-z0-9_]*`.
7. Type casts written as `(int)$var` without an intervening space.
8. Twig tag balance for every changed `.twig`.
9. File names lower case with underscores.

Review-only (a checker cannot decide these):

- Whether the brace style is 1TBS in every construct.
- Whether a reformat is justified vs. merely churn.
- Whether a new file's indentation matches its directory's convention.

## 5. Common violations an agent introduces

- Writing PHP bodies with 4 spaces or 2 spaces instead of TAB. (Recurring failure: indentation is easy to
  get wrong in generated code — verify the actual bytes, not the intent.)
- Adding a closing `?>`.
- Omitting the method scope keyword.
- Reformatting a whole template to "2 spaces" and breaking OCMOD searches.
- Trailing whitespace on edited lines.
- Switching a file's line endings to CRLF on Windows.
- Renaming HTML classes from `my_class` to `my-class` in a template that existing CSS or JS targets.

## Verification Notes

- Official rules `[official]`: retrieved from the OpenCart coding-standards document on 2026-09-13
  (299 lines). They are documentation claims, not baseline-verified behavior.
- Baseline census `[baseline]`: measured over 84 PHP files and 12 `.twig` files downloaded from
  LiveStore `v3.0.4.5`. The PHP sample is skewed toward controllers/models/services; view and language
  files are under-represented, so per-directory exceptions may exist that this census did not see.
- The Twig conclusion rests on 12 files out of 404 `.twig` files in the baseline (`[baseline]`). It is
  strong enough to reject "2 spaces is the practice", but a per-directory breakdown was NOT produced.
- `php-cs-fixer` / `phpstan` configs referenced by the standard were **not** verified to exist in this
  baseline — `[needs-verification]`.
- JavaScript and CSS indentation rules were **not** measured; only the documented claim is recorded.
- Not verified: whether LiveStore ships a `.editorconfig` or CI style gate.
