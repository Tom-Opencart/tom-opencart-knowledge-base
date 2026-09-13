# OpenCart Settings & Config (OC3 / LiveStore)

## Scope

How settings are stored, scoped, read and written: the `setting` table and its model, the `Config` class,
the key conventions for core/module/theme settings, and the defect class where a setting is exposed in the
UI but consumed by nothing. Does not cover authorization (see `permissions.md`).

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
| `admin/model/setting/setting.php` | the `setting` table access layer (`[verified]`, 54 lines, full) |
| `system/library/config.php` | `Config` — the runtime key/value store (`[verified]`, 67 lines, full) |
| `system/framework.php` | loads settings into `Config` at bootstrap (`[baseline]`) |

## The `setting` table (`[verified]`)

Columns, read from the model's own SQL: `store_id`, `code`, `key`, `value`, `serialized`.

A row is therefore identified by the triple **(`store_id`, `code`, `key`)**.

### `getSetting($code, $store_id = 0)` — :3-17

Selects all rows for that code and store, then builds an array keyed by `key`:

- `:9-10` when `serialized` is falsy, the raw `value` is returned;
- `:11-13` when `serialized` is truthy, `value` is passed through `json_decode($value, true)`.

**Consequence — `serialized` is a JSON flag, not PHP `serialize()`.** A value written as a PHP-serialized
string will not decode; only JSON round-trips. Arrays must be written by the writer's array branch.

### `editSetting($code, $data, $store_id = 0)` — :19-31 — DESTRUCTIVE

```
:20  DELETE FROM `…setting` WHERE store_id = '<id>' AND `code` = '<code>'
:22  foreach ($data as $key => $value) {
:23      if (substr($key, 0, strlen($code)) == $code) {
:24          if (!is_array($value)) { INSERT … value = escape($value) }
:26          else { INSERT … value = json_encode($value), serialized = '1' }
```

Two consequences, both easy to trigger:

**Consequence — `editSetting()` is a full replace, not a merge.** It deletes **every** row for that
`code`/`store_id` and re-inserts only what is in `$data`. Any existing key for that code that is absent
from the submitted array is **permanently lost**. A settings form that posts only part of its fields wipes
the rest.

**Consequence — keys that do not start with `$code` are silently dropped.** The `:23` filter keeps only
keys whose **string prefix equals the code** (note: `==`, not strict). A module saving with
`code = 'module_x'` can only persist keys beginning `module_x`. A key named differently — a typo, a
mismatched prefix, a nested key — is discarded **without any error**. The save appears to succeed and the
value simply never exists.

### Other methods

| Method | Line | Behaviour |
| --- | --- | --- |
| `deleteSetting($code, $store_id = 0)` | :33-35 | deletes all rows for that code/store |
| `getSettingValue($key, $store_id = 0)` | :37-45 | selects by **`key` only** — no `code`; returns `value` or `null` |
| `editSettingValue($code, $key, $value, $store_id = 0)` | :47-53 | `UPDATE` by code+key+store; sets `serialized` to `'1'` for arrays and `'0'` otherwise |

**Consequence — `getSettingValue()` ignores the code**, so it reads the first row matching the key across
codes. Two settings sharing a key name collide; prefer `getSetting($code)`.

## Store scoping

- `store_id = 0` is the default store used by every method's default argument (`[verified]`).
- Multi-store values are separate rows with a non-zero `store_id`.
- `Config::get($key)` (`system/library/config.php` :23-25) returns `null` for an unknown key — it never
  throws. Code must therefore treat a missing setting as `null` and decide its own default.

**Consequence — reading a setting through `$this->config->get()` cannot distinguish "set to empty" from
"never set".** Both may be falsy. Use `Config::has($key)` (:44-46) when the distinction matters.

## The `Config` class (`[verified]`, `system/library/config.php`)

- `get($key)` :23 — value or `null`.
- `set($key, $value)` :33.
- `has($key)` :44 — `isset()`.
- `load($filename)` :53-66 — includes `DIR_CONFIG . $filename . '.php'`, which must define an `$_` array,
  and merges it into the data (:57-61). A missing config file triggers an error and `exit()` (:62-65).

**Consequence — config files are plain PHP arrays assigned to `$_`.** They are merged, so a later
`load()` overrides earlier keys.

## Key conventions (`[baseline]`)

| Domain | Convention | Example |
| --- | --- | --- |
| Core store settings | `config_*` | `config_seo_url`, `config_file_ext_allowed` |
| Module settings | `module_<name>_*`, stored with `code = 'module_<name>'` | `module_banner_*` |
| Theme settings | `theme_<name>_*`, stored with `code = 'theme_<name>'` | `theme_default_*` |

The `code` used with `editSetting()` must match the key prefix, per the `:23` filter above. This is the
single most common cause of "the setting will not save".

## The inert-setting defect class

A setting can be **fully wired on the write and display side and consumed by nothing**:

- a field is rendered in an admin template;
- the controller holds a default value for it;
- language strings exist for its label and help text;
- **no code anywhere reads the key.**

The admin's choice then appears to save and silently does nothing. This is invisible to a syntax check, a
linter, and a static style gate, because every individual file is valid.

**Detection (mechanical):** grep the setting key across the whole tree and classify the hits. If the only
hits are (a) the admin template, (b) the language file, and (c) the defaults array in the controller, then
nothing consumes the setting — it is inert. A real consumer is a read at runtime, typically
`$this->config->get('…')` or `$settings['…']` in a catalog controller.

**Why it matters beyond cosmetics:** an inert setting is not merely useless — if it gates a security- or
performance-relevant behaviour (a rate limit, a permission, an output filter), the store is unprotected
while the admin believes otherwise.

## Form-default vs persisted-default

A defaults array in a controller is often used **only to pre-fill the settings form** when a key is not yet
saved. In that pattern:

```
foreach ($defaults as $key => $default) {
	if (isset($this->request->post[$key]))      { $data[$key] = $this->request->post[$key]; }
	elseif (isset($setting_info[$key]))         { $data[$key] = $setting_info[$key]; }
	else                                        { $data[$key] = $default; }
}
```

**Consequence — the key does not exist in the database until the admin saves the page.** Runtime code that
reads it must handle absence, and a "default" shown in the UI is not a stored value. Any consumer that
assumes the key exists behaves differently on a freshly installed store than on one whose settings page was
once saved.

**Detection:** check whether the defaults array is written through `editSetting()`/`addSetting()` at
install time, or only merged into the form payload. If it is only merged into the form, the value is a form
default, not a persisted one.

## ENFORCEABLE CHECKS

1. Every setting key written by a form begins with the `code` passed to `editSetting()`, or it will be
   dropped (:23).
2. Settings forms submit the **complete** set of keys for that code — `editSetting()` deletes first (:20).
3. Every setting exposed in an admin template has a verified runtime consumer (grep the key).
4. Array values are written through the array branch so `serialized` is set to `1`.
5. Runtime reads of a setting tolerate absence (`null`), and use `has()` when "unset" differs from "empty".
6. `getSettingValue()` is not relied on to disambiguate codes.

## Verification Notes

- Read directly and in full: `admin/model/setting/setting.php` (54 lines) and
  `system/library/config.php` (67 lines). The `setting` table columns are `[verified]` from these models'
  SQL, not from a `CREATE TABLE` in `install/opencart.sql`.
- The key conventions are `[baseline]` (observed across core code), not a documented rule.
- The inert-setting and form-default classes are stated as **methodology** with mechanical detection
  procedures; the worked pattern above is generic rather than tied to one verified baseline file, so the
  class itself is `[needs-verification]` as a claim about core, and `[verified]` as a detection technique.
- How `Config` is populated at bootstrap (which settings are loaded globally versus per page) was
  `[baseline]` — `system/framework.php` was read for request flow, not for the settings load path.
- Not verified: whether LiveStore adds settings outside the `setting` table, and the exact default values
  of `config_file_ext_allowed`.
