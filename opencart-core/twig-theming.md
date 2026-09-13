# OpenCart Twig & Theming Standards (OC3 / LiveStore)

## Scope

The template engine as the baseline configures it: the renderer, the Twig environment, the loader chain,
the legacy `.tpl` fallback, the template path rules, the view events, and the escaping consequence of
`autoescape => false`. Styling/CSS conventions are not covered.

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference

## Core Files

| File | Role |
| --- | --- |
| `system/library/template/twig.php` | the Twig renderer (`[verified]`, 44 lines, full, namespace `Template`) |
| `system/library/template/template.php` | the legacy `.tpl` renderer (`[verified]`, 25 lines, full, namespace `Template`) |
| `system/config/catalog.php` | engine selection and view events (`[verified]`, 62 lines) |
| `system/config/admin.php` | admin view events (`[verified]`, 49 lines) |

## The renderer (`[verified]`, `system/library/template/twig.php`)

- :2-3 `namespace Template; final class Twig`.
- :6-8 `set($key, $value)` accumulates data.
- :10-20 `render($filename, $code = '')`:
  - `:12` `$file = DIR_TEMPLATE . $filename . '.twig';`
  - `:17` a missing file throws `\Exception('Error: Could not load template …')`.
  - Passing `$code` explicitly skips the filesystem read — used when template source is supplied directly.
- :23-28 the Twig environment is configured **explicitly**:
  ```
  'autoescape'  => false,
  'debug'       => false,
  'auto_reload' => true,
  'cache'       => DIR_CACHE . 'template/'
  ```
- :33-35 the loader is a **ChainLoader** of:
  - `ArrayLoader([$filename . '.twig' => $code])` — the current template's source, and
  - `FilesystemLoader([DIR_TEMPLATE])` — so that `{% include %}` / `{% extends %}` resolve from the
    template root.
- :37-39 `$twig->render($filename . '.twig', $this->data)`.
- :40-43 the catch block triggers an error and `exit()`.

**Consequence — `autoescape` is OFF, so `{{ value }}` does NOT escape.** This is the single most important
fact in theming: unescaped output of user data is an XSS vector, and the default expectation from standard
Twig (autoescape on) is wrong here. Escaping must be explicit — `{{ value|e }}` / `{{ value|escape }}`.
See `security.md` §2.

**Consequence — `auto_reload => true` with a filesystem cache means a template edit is picked up on the
next render without clearing the cache, but the compiled cache in `DIR_CACHE . 'template/'` still exists
and must be considered when diagnosing "my change did not appear".** Since `template_cache` is `true`
(`config/catalog.php` :26), stale compiled templates are a real diagnostic candidate.

**Consequence — includes resolve from `DIR_TEMPLATE`, not from the including file's directory.** A relative
include does not work the way it does in a plain Twig project.

**Consequence — the catch clause is written as `catch (Exception $e)` inside `namespace Template`**, so it
names `Template\Exception`, which is not the Twig exception hierarchy. As written it does not catch Twig
errors; those propagate. `[verified]` as written; the runtime effect is `[needs-verification]`.

## The legacy renderer (`[verified]`, `system/library/template/template.php`)

`:10-25` `render($template)` resolves `DIR_TEMPLATE . $template . '.tpl'`, then `extract($this->data)`,
`ob_start()`, `require($file)`, `ob_get_clean()`. A missing file throws.

**Consequence — plain PHP templates (`.tpl`) are still supported by the engine**, and `extract()` puts
data into local scope. A `.tpl` file is executed PHP, so it is trusted code by definition.

## Engine selection (`[verified]`, `system/config/catalog.php`)

- :24 `$_['template_engine'] = 'twig';`
- :25 `$_['template_directory'] = '';`
- :26 `$_['template_cache'] = true;`

**Consequence — the engine is configuration, not hard-coded.** `template_engine` selects the renderer class
(`Twig` or `Template`), so both are live options and an extension must not assume the class name.

## Template path rules

- The renderer builds the file path as `DIR_TEMPLATE . <name> . '.twig'` (`[verified]` :12).
- `Loader::view()` passes the logical template name, and the theme layer supplies `DIR_TEMPLATE`
  (`[baseline]`).

**Rule for `{% include %}` and `{% extends %}` in this codebase:** use the **full path from the theme
root**, i.e. `{theme}/template/{folder}/{name}.twig` including the explicit `.twig` extension. `[baseline]`

**Consequence — omitting `.twig`, or using a path relative to the current file, fails.** The loader has a
single filesystem root, so there is exactly one correct spelling.

`[needs-verification]`: the exact definition of `DIR_TEMPLATE` (per app, per theme) was **not** read in
this pass. The include-path rule above is recorded as baseline practice corroborated by the project's own
documented rule, not as a verified line of core.

## Themes in the baseline (`[baseline]`)

The shipped tree contains exactly one theme directory: `upload/catalog/view/theme/default`.

**Consequence — a theme package normally OVERLAYS a store that already has `default` and its own theme.**
A theme must not assume it is the only theme present, and template overrides are keyed by
`catalog/view/theme/<theme>/template/…`.

## View events (`[verified]`)

Catalog (`system/config/catalog.php` :42-62):

| Event | Handlers |
| --- | --- |
| `controller/*/before` | `event/language/before` |
| `controller/*/after` | `event/language/after` |
| `view/*/before` | `500 => event/theme`, `998 => event/language` |
| `language/*/after` | `event/translation` |

Admin (`system/config/admin.php` :35-48) declares `view/*/before` **twice** — `:42` with
`999 => event/language, 1000 => event/theme`, and `:46` with a plain `event/language`. In a PHP array
literal the second assignment for the same key **overwrites** the first.

**Consequence — in the admin app as shipped, the `event/theme` handler and the priority ordering are lost
to the duplicate key.** `[verified]` as a property of the config file; whether this is intentional or an
oversight is `[needs-verification]`. Anyone adding a `view/*/before` handler to a config must merge into
the existing array rather than declaring the key again.

## ENFORCEABLE CHECKS

1. Every `{% include %}` / `{% extends %}` uses the full path from the theme root **with** the `.twig`
   extension.
2. No user-controlled value is printed with a bare `{{ }}` — an explicit escape filter is required.
3. Every changed `.twig` has balanced `{% %}` / `{{ }}` / `{# #}` and matched block tags.
4. No config declares the same array key twice.
5. New templates live under `catalog/view/theme/<theme>/template/…` (or the admin template tree) and are
   written in the indentation style of their own directory (see `coding-standards.md`).
6. No assumption that `template_engine` is `twig` — read it if the behavior depends on the engine.

## Verification Notes

- Read directly and in full: `system/library/template/twig.php` (44 lines),
  `system/library/template/template.php` (25 lines), `system/config/catalog.php` (62 lines),
  `system/config/admin.php` (49 lines).
- The Twig **version** is not claimed here; the `\Twig\Loader\*` namespaced classes imply Twig 2/3, but
  `composer.json` was not read, so the exact version is `[needs-verification]`.
- `DIR_TEMPLATE`'s definition was **not** read — the path rule is `[baseline]`. Do not cite it as verified.
- The theme inventory ("only `default` in the LiveStore tree") is `[baseline]` from the repository tree; a
  real store may have additional themes.
- The `event/theme` duplicate-key finding is `[verified]` as file content; its intent is not.
- Asset registration (`addStyle`/`addScript`, the `document` library) was **not** read in this pass —
  `[needs-verification]`.
- Not verified: Twig sandbox/extensions, `template_directory` semantics, and whether the store's
  configuration disables `template_cache`.