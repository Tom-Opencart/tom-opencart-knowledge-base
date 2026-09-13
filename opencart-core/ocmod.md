# OCMOD — install.xml Modification System (OC3 / LiveStore)

## Scope

How a `.ocmod.zip` / `install.xml` modification is registered, stored, applied to store files, and
loaded at runtime. Covers the XML element set the baseline actually parses, the modification cache, and
the failure modes that break a live store. Does not cover general packaging strategy (see
`opencart-core/release-and-packaging.md`).

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: targeted grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference

**Baseline-specific warning.** The LiveStore `admin/controller/marketplace/modification.php` is
**customised**, not stock: it is 1163 lines and adds `restore()`, `clearHistory()`, `download()`,
`upload()`, `clear()`, `enable()`, `disable()`, `clearlog()`, plus `parseMetaFromXml()`. Do not compare
its line numbers against stock OpenCart 3 or against another store's build.

## Core Files

| File | Role |
| --- | --- |
| `admin/controller/marketplace/modification.php` | modification manager UI; parses `install.xml`; `refresh()` applies patches |
| `admin/controller/marketplace/install.php` | `.ocmod.zip` upload flow: `install()`, `unzip()`, `move()`, `xml()`, `remove()`, `uninstall()` |
| `admin/model/setting/modification.php` | DB access: `getModificationByCode`, `addModification`, `deleteModification`, `deleteModificationsByExtensionInstallId` |
| `system/startup.php` | `modification($filename)` — runtime path override (`[verified]` :48-62) |
| `system/engine/loader.php` | loads through the override (`[verified]` `is_file` checks at :76, :150, :167) |

## The `<file>` element and its `path` attribute (`[verified]`)

Source: `admin/controller/marketplace/modification.php` :604-609.

- :604 `$dom->getElementsByTagName('modification')->item(0)->getElementsByTagName('file')`
- :607 then `$file->getElementsByTagName('operation')` — operations are children of `<file>`
- :609 `$files = explode('|', str_replace("\\", '/', $file->getAttribute('path')));`

**Consequence — `path` accepts MULTIPLE files separated by `|`, and backslashes are normalised to `/`.**
One `<file>` block can patch several files at once. Paths are relative to the store root
(`admin/...`, `catalog/...`, `system/...`).

## The element set the baseline actually parses (`[verified]`)

Source: `admin/controller/marketplace/modification.php` :675-710 and :794-796.

| Element / attribute | Line | Meaning |
| --- | --- | --- |
| `<operation error="...">` | :675 | error-handling mode for the operation |
| `<ignoreif regex="true">` | :678, :681, :682, :686 | skip the operation when its text is present; with `regex="true"` it is treated as a PCRE |
| `<search regex="true">` | :695 | treat the search text as a regex instead of a literal |
| `<search trim="...">` | :698 | trim behaviour applied to the search text |
| `<search index="...">` | :699 | which occurrence to target when the search text appears multiple times |
| `<search limit="...">` | :795 | occurrence limit in the second (plain-replace) code path |
| `<add trim="...">` | :708 | trim behaviour applied to the added text |
| `<add position="...">` | :709 | where to inject relative to the search match |
| `<add offset="...">` | :710 | numeric offset used together with `position` |

**Consequence — `position` and `offset` belong to `<add>`, never to `<search>`, and `error="..."`
belongs to `<operation>`, never to `<file>`.** These are the three attributes most often attached to the
wrong element by hand-written XML.

## Registration: the `code` identity rule — CRITICAL (`[verified]`)

Source: `admin/controller/marketplace/install.php` `xml()` :237, and specifically :285-289, :322-333.

```
:285  // Check to see if the modification is already installed or not.
:286  $modification_info = $this->model_setting_modification->getModificationByCode($code);
:288  if ($modification_info) {
:289      $this->model_setting_modification->deleteModification($modification_info['modification_id']);
...
:333  $this->model_setting_modification->addModification($modification_data);
```

The installer identifies the existing modification **by `<code>`**, deletes that row, and inserts the new
one.

**Consequence — changing `<code>` in a shipped `install.xml` does NOT update the existing modification.**
The lookup at :286 finds nothing, nothing is deleted at :289, and `addModification()` at :333 inserts a
SECOND row. Both modifications now carry the same `<search>` blocks, both are applied to the same file
during `refresh()`, and the duplicated injection typically breaks the store (fatal error / 500). This is
the single most destructive, easily-triggered mistake in OCMOD packaging. Treat `<code>` as immutable for
the lifetime of the extension; renaming it deliberately requires every user to delete the old
modification in the admin first.

Note the contrast with the **manual** add form, which does NOT replace: `validateForm()` rejects a
duplicate instead — `:1387-1391` rejects a duplicate `<name>`, `:1397-1401` rejects a duplicate `<code>`
via `getModificationByCode`. The delete-and-replace behaviour is specific to the installer's `xml()` path.

## Metadata parsing (`[verified]`)

Source: `admin/controller/marketplace/modification.php` `xml()` :258 and `parseMetaFromXml()` :1426+.

- :293 `<name>`, :311 `<code>`, :329 `<author>`, :337 `<version>`, :345 `<link>`.
- `parseMetaFromXml()` :1427 initialises `name`, `code`, `author`, `version`, `link` and reads
  `:1433` name, `:1439` code, `:1445` author, `:1451` version, `:1457` link.
- `validateForm()` :1384-1406 requires a non-empty name, code and version.

**Consequence — `<code>` is mandatory and must be stable; `<version>` is mandatory.** A missing `<code>`
is rejected outright (`:1394-1395`).

## How patches are applied — `refresh()` (`[verified]`)

Source: `admin/controller/marketplace/modification.php` :488 onward.

- :554 also loads `DIR_SYSTEM . 'modification.xml'`.
- :557 also reads legacy `glob(DIR_SYSTEM . '*.ocmod.xml')` files.
- :568-569 iterates the DB modification rows and applies **only those whose `status` is truthy**.
- :641-642 files under `DIR_SYSTEM` are keyed as `system/<relative>`.
- :667 content line endings are normalised with `preg_replace('~\r?\n~', "\n", $content)`.
- :682 / :686 `<ignoreif>` skips the operation when the text is present (or matches, with `regex="true"`).
- :586 each applied modification is written to the refresh log.

**Consequence — a DISABLED modification is not applied.** After `refresh()`, only enabled rows take effect.
Both the DB and the legacy file-based sources feed the same patch pass.

## How patched files are loaded at runtime (`[verified]`)

Source: `system/startup.php` :48-62, used from :77-78 and :90 (`include_once(modification($file))`).

- :49 `function modification($filename)`
- :51 admin: `DIR_MODIFICATION . 'admin/' . substr($filename, strlen(DIR_APPLICATION))`
- :53 install: `DIR_MODIFICATION . 'install/' . ...`
- :55 catalog: `DIR_MODIFICATION . 'catalog/' . ...`
- :59 system: `DIR_MODIFICATION . 'system/' . substr($filename, strlen(DIR_SYSTEM))`
- :62 the overridden path is returned only `if (is_file($file))`

**Consequence — the runtime reads the PATCHED COPY from `DIR_MODIFICATION`, not the original file.** The
original on disk is left untouched by `refresh()`. This is why a stale cache silently serves old patched
code, and why adding or changing an `install.xml` requires the admin **Refresh** action to rebuild it.
Deleting a modification does not automatically rebuild the cache either.

## ENFORCEABLE CHECKS

Run these on every `install.xml` change:

1. **`<code>` unchanged** compared with the previously shipped version. This is the highest-priority
   check; a build should FAIL on any change. `[verified]` mechanism above.
2. `<name>`, `<code>`, `<version>` all present and non-empty (`[verified]` :1384-1406).
3. `<file path="...">` uses forward slashes and targets real files under the store root; `|` may be used
   to list several (`[verified]` :609).
4. Every `<operation>` carries `error="skip"` when its search text is optional.
5. `position` / `offset` appear on `<add>` only; `error` on `<operation>` only.
6. `<search>` blocks are single-line literal text, and the exact text exists in the current file.
7. No two `<operation>` blocks target the same region of the same file (collision / double-injection risk).
8. `<file>` blocks are ordered admin → catalog for predictability.
9. XML is well-formed (DOM validation) before packaging.

## Verification Notes

- Read directly: `admin/controller/marketplace/install.php` (`xml()` :237-350, function list),
  `admin/controller/marketplace/modification.php` (`add()` :80-118, `validateForm()` :1369-1416,
  `refresh()` internals :540-710, XML parsing :604-710 and :794-796, metadata :258-356 and :1426-1457),
  `system/startup.php` :48-62 and :70-90.
- The file-path override targets (`DIR_MODIFICATION` + app prefix) are `[verified]` from `startup.php`;
  the constant's actual disk location is defined in `config.php` and was **not** read here —
  `[needs-verification]`.
- `DIR_MODIFICATION` write mechanics inside `refresh()` (directory creation, file writing) were only
  partially observed (`[baseline]`); the exact write call was not read line-by-line.
- The `<search limit>` attribute is `[verified]` for the second code path (:795) only; whether it also
  applies to the regex path was not confirmed.
- Behaviour of `modification.xml` / `*.ocmod.xml` legacy sources is `[verified]` as a source of patches;
  their precedence relative to DB rows was **not** determined.
- Not verified: OpenCart 4.x OCMOD/VQMod differences, and any store-specific modification to these
  controllers.
