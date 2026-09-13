# OpenCart Extension Development Standards (OC3 / LiveStore)

## Scope

How an extension is laid out, registered, installed and uninstalled: the extension type taxonomy, the
canonical file set, the `install()`/`uninstall()` lifecycle, the enable/disable lifecycle owned by core,
and the idempotency rules. Packaging/archives: `release-and-packaging.md`. Authorization:
`permissions.md`. OCMOD: `ocmod.md`.

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference

## The extension type taxonomy (`[verified]`)

The baseline ships exactly these 14 directories under `admin/controller/extension/`:

```
advertise  analytics  captcha  currency  dashboard  extension  feed  fraud
module     payment    report   shipping  theme      total
```

**Consequence — this list is the same set used by the authorization layer.** The permission key builder
treats a route whose two-part prefix is in its own `$extension` list as a three-part key, and that list has
14 entries covering these types (see `permissions.md`). Adding a directory here does **not** add it to the
authorization list — a new type needs a permission-layer change.

**Consequence — the type determines where the extension is enabled from**, and `extension/extension` is
itself a type (the enable/disable manager). A "module" is enabled per store layout; a "payment" method is
enabled in its own section.

## The canonical file set (`[verified]` for `module`)

A module is not one file — it is a coordinated set. Verified locations for the shipped `banner` module:

| Part | Path |
| --- | --- |
| Admin controller | `admin/controller/extension/module/banner.php` |
| Admin language | `admin/language/<lang>/extension/module/banner.php` (per language: `en-gb`, `ru-ru`) |
| Admin template | `admin/view/template/extension/module/banner.twig` |
| Catalog controller | `catalog/controller/extension/module/banner.php` |
| Catalog template | `catalog/view/theme/<theme>/template/extension/module/banner.twig` |

**Consequence — an admin-only module is a broken module.** The admin half configures; the catalog half
renders. Shipping the admin side without the catalog controller produces a module that appears installed
and outputs nothing.

**Consequence — the language file must exist for every installed language.** A missing language file
surfaces as raw keys instead of labels, not as a fatal error, so it can ship unnoticed.

## The `install()` / `uninstall()` contract

`install()` and `uninstall()` are **lifecycle hooks invoked by core**, not pages the user opens directly.

For a module, core's manager (`admin/controller/extension/extension/module.php`) does:

1. `install()` :14 → `if ($this->validate())` :20
2. register the extension :21
3. `addPermission(...)` for `access` and `modify` on `extension/module/<ext>` :25-26
4. `$this->load->controller('extension/module/<ext>/install')` :29 — **the extension's own `install()`
   runs only after the permission rows have been written**

`validate()` :206-207 requires `modify` on `extension/extension/module`. For themes the flow is the same
shape: `theme.php` `install()` :13, `addPermission` :23-24, `uninstall()` :35, `validate()` :122-123
requiring `modify` on `extension/extension/theme`.

**Consequence — the permission rows for a new extension are created by core, not by the extension.** An
extension must not write its own permission rows; and if a group already exists without the new route,
core's `install()` is what grants it. A route with no permission row behaves per `permissions.md`.

**Consequence — not every extension implements these hooks.** Verified: the shipped `banner` module
contains only `index()` :5 and `validate()` :135 and has **no** `install()`/`uninstall()` at all, because it
owns no schema. `[verified]`

**Consequence — core tolerates a missing hook.** `$this->load->controller(...)` resolves a missing
controller method as a soft failure, so an extension without `install()` installs fine. The reverse is not
true: a hook that throws leaves the store part-installed.

## Hard rules for the lifecycle hooks

1. **`install()` MUST be idempotent.** It can run more than once (reinstall, refresh, or a call reached
   from more than one path). Running it twice must not duplicate rows, columns, events, settings, or
   permissions, and must not error.
2. **`uninstall()` MUST NOT drop tables holding merchant data.** Deleting a merchant's content on
   uninstall is data loss; core's own extensions are conservative.
3. **There is no `migrate()` method.** Schema belongs to `install()` only. Schema repair for an existing
   store is a **guarded** `ALTER` (check the column list first), never a separate migration entry point.
4. **No DDL on a page load.** `index()`, `form()`, `getList()` and read paths must not create or alter
   tables. Cross-file calls count: a controller calling a model's `upgradeSchema()`/`install()` from a page
   is the classic violation.
5. **Guard repeated work with a `static $checked`** (or a schema-version marker) so a check reached many
   times per request performs its query once.
6. **Never drop a column in a hook that can run before the code reading it is updated** — the lifecycle
   skew below.

## The idempotency pattern

```php
public function install() {
	static $checked = false;

	if ($checked) {
		return;
	}

	$checked = true;

	// … checks and DDL, each guarded by an existence test
}
```

The `static` guard is per-request. The **existence guards** are what make the operation idempotent across
requests — the static flag alone is a performance measure, not a correctness one. Both are needed, and they
solve different problems.

## Lifecycle skew (the ordering hazard)

Code shipped or injected by a manifest can execute **before** the extension's `install()` has run.

**Consequence — injected/injected-into code must be a safe no-op until the extension is installed and
enabled.** Concretely: a status-gated read, a `static` guard, and a check that the extension is enabled
before depending on its tables. Without this, an admin pressing a global "Refresh" on a store whose
extension is not yet installed gets a fatal error from the injected code.

See `ocmod.md` for how a manifest injects code and why a Refresh re-applies everything.

## Enable/disable versus install (three distinct states)

| State | Owned by | Effect |
| --- | --- | --- |
| Files present | the installer | code exists; nothing runs |
| Installed | core's `install()` + the extension's hook | permissions granted, schema created |
| Enabled | the store's layout/setting for that extension | output appears in the storefront |

**Consequence — deleting files does not uninstall.** The OpenCart installer copies but never deletes
(`release-and-packaging.md`), so "reinstall to fix it" does not remove anything; the row and any schema
remain.

## ENFORCEABLE CHECKS

1. Every extension type used is one of the 14 known directories, or the permission layer is updated too.
2. A module ships **both** the admin half and the catalog half (controller, plus template where it renders).
3. A language file exists for every language directory the store uses.
4. `install()` is idempotent: run twice, no error, no duplicates. Verify by inspection of every DDL/DML
   statement for an existence guard.
5. `uninstall()` contains no `DROP TABLE` for merchant-data tables.
6. No `migrate()` method exists; schema is created in `install()`.
7. No DDL reachable from a page load — prove this by scanning the call graph, not just the file.
8. Repeated in-request checks are behind a `static` guard.
9. Catalog reads of the extension's tables confirm the extension is enabled.

## Verification Notes

- Read directly: `admin/controller/extension/extension/module.php` (`install()` :14-29, `validate()`
  :206-207), `admin/controller/extension/extension/theme.php` (`install()` :13, `addPermission` :23-24,
  `uninstall()` :35, `validate()` :122-123), and `admin/controller/extension/module/banner.php`
  (method listing only: `index()` :5, `validate()` :135).
- The 14 extension type directories are `[verified]` from the repository tree.
- The canonical file set is `[verified]` for the `module` type (banner); other types
  (`payment`, `shipping`, `total`, `feed`, …) follow the same shape but were **not** individually
  verified — treat their exact file sets as `[baseline]`.
- The claim that `Loader::controller()` fails softly for a missing method is `[baseline]`.
- `[needs-verification]`: the exact behavior of core when an extension's `install()` throws (whether the
  registration is rolled back). MyISAM makes a transactional rollback impossible (`database-schema.md`), so
  a partial state is expected — but this was not observed.
- Not verified: extension type-specific managers (payment/shipping/total enable paths) beyond the two
  verified above.