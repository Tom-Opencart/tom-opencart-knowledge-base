# OpenCart Database & Schema Standards (OC3 / LiveStore)

## Scope

Database access conventions, the schema conventions of the core installer SQL, schema ownership and
migration rules for extensions, and the verified result shape of the DB layer. Does not cover the
`setting` table model in depth (see `settings-and-config.md`) or SQL-injection defence (see `security.md`).

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
| `install/opencart.sql` | the core schema shipped by the installer (327 KB) |
| `system/library/db.php` | `DB` wrapper, delegates to an adaptor |
| `system/library/db/mysqli.php` | the MySQLi adaptor (`[verified]`, 86 lines, namespace `DB`) |
| `system/library/db/mpdo.php` | the PDO adaptor (`[exists]`) |
| `admin/model/setting/setting.php` | the `setting` table access layer |

## Schema conventions in `install/opencart.sql` (`[verified]`)

- Every table is emitted as `DROP TABLE IF EXISTS \`oc_<name>\`;` followed by `CREATE TABLE \`oc_<name>\`` —
  the prefix is written **literally as `oc_`** in the file (`:10-14`) and is substituted with the store's
  chosen prefix by the installer. `[verified]`
- Table suffix: `) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_general_ci;` — with some tables using
  `ENGINE=MyISAM DEFAULT CHARSET=utf8;` without an explicit collation. `[verified]`
- Index naming in core is `KEY \`<column>\` (\`<column>\`)` — i.e. the key is commonly named after the
  column, not with an `idx_` prefix. `[verified]` from `oc_extension_install` and the surrounding tables.
- Primary keys are `int(11) NOT NULL AUTO_INCREMENT` with `PRIMARY KEY (\`<x>_id\`)`. `[verified]`
- `datetime NOT NULL` for date columns in the core schema (no default). `[verified]`

**Consequence — the core schema is MyISAM, not InnoDB.** MyISAM has **no transactions** and **no foreign
keys**. A multi-statement `install()` that fails halfway leaves the store in a partially-created state
with no rollback. This is the reason idempotency below is not a style preference but a hard requirement.

**Consequence — `utf8` is not `utf8mb4`.** The schema and the connection charset are both `utf8`. Four-byte
characters (emoji) are not storable unless the store's schema and connection were changed deliberately.

## The DB layer (`[verified]`)

### Wrapper — `system/library/db.php`

`class DB` holds one private `$adaptor` and delegates: `query($sql)` :44, `escape($value)` :55,
`countAffected()` :64, `getLastId()` :73, `connected()` :82. The adaptor is instantiated from
`'DB\\' . $adaptor` :28 and an unknown adaptor throws :33.

### Adaptor — `system/library/db/mysqli.php`

- :17 `set_charset('utf8')`.
- :18 `SET SESSION sql_mode = 'NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION'` — the session sql_mode is forced.
  Consequence: `NO_ZERO_DATE` is **not** set, and `STRICT_TRANS_TABLES` is **not** set, so MySQL accepts
  truncated/zero values that a modern default configuration would reject. A schema relying on strict mode
  would behave differently on another host. `[verified]`
- :24-51 `query($sql)`:
  - takes a **plain SQL string** — there is **no parameter binding and no prepared-statement API** in this
    layer;
  - on a result set returns a `stdClass` with `->num_rows` :36, `->row` (first row, or `[]`) :37 and
    `->rows` (all rows) :38;
  - for a non-result statement returns `true` :46;
  - on error throws an `\Exception` whose message contains the **full SQL** :49.
- :53-55 `escape($value)` = `real_escape_string($value === null ? '' : $value)`.
  **Consequence: `escape(null)` returns an empty string, not `NULL`.** Escaping a nullable value converts
  it to `''`, which matters for columns that must store `NULL`.
- :57-59 `countAffected()` = `affected_rows`; :61-63 `getLastId()` = `insert_id`.
  **Consequence: there is no `lastInsertId` on the wrapper — `getLastId()` reads the adaptor's
  `insert_id` and must be called after the INSERT, before any other statement on that connection.**
- :79-85 `__destruct()` closes the connection.

## Result-shape contract (`[verified]`)

| Call | Returns |
| --- | --- |
| `$this->db->query($sql)` on a SELECT | object with `->num_rows`, `->row`, `->rows` |
| `$this->db->query($sql)` on INSERT/UPDATE/DELETE/DDL | `true` |
| `$this->db->getLastId()` | `insert_id` of the last INSERT |
| `$this->db->countAffected()` | `affected_rows` of the last statement |
| `$this->db->escape($value)` | escaped string; `null` becomes `''` |

**Consequence — never call `->num_rows` on the result of an INSERT/UPDATE/DELETE.** It returns `true`, and
reading a property of a boolean is a fatal error in modern PHP.

## Schema ownership and migration rules

This is the most important standard in this pack.

1. **A table an extension owns is created by that extension's `install()`.** Schema belongs to the
   lifecycle hook, not to a page.
2. **`uninstall()` must not destroy user data indiscriminately.** A blanket `DROP TABLE` on uninstall
   discards content the merchant may consider theirs; core's own extensions are conservative here.
3. **Installers must be 100% idempotent** — re-running `install()` must not duplicate rows, columns,
   settings, events or crash. Required because of the MyISAM no-transaction consequence above.
4. **`CREATE TABLE IF NOT EXISTS` does NOT add columns to an existing table.** For a store that already has
   the table from an older version, this statement is a silent no-op, so a new column never appears. A
   **guarded `ALTER`** is the only mechanism that repairs a legacy table.
5. **Run schema DDL from `install()`/`uninstall()` — never from a page load.** `index()`, `form()`,
   `getList()` and read methods must not execute DDL.
6. **If a read path genuinely needs a lifecycle-skew check** (code can run before `install()`), make it
   once per request with a `static $checked` flag rather than a query per call.

The guarded-column pattern:

```php
$columns = array();

foreach ($this->db->query("SHOW COLUMNS FROM `" . DB_PREFIX . "my_table`")->rows as $column) {
	$columns[] = $column['Field'];
}

if (!in_array('my_new_column', $columns, true)) {
	$this->db->query("ALTER TABLE `" . DB_PREFIX . "my_table` ADD `my_new_column` VARCHAR(255) NOT NULL DEFAULT ''");
}
```

**Consequence — `SHOW COLUMNS` is a metadata roundtrip.** Placing an unguarded column check inside a read
method runs it on every call in a request; that is a measurable performance defect, not just untidiness.
A once-per-request static guard, or a schema-version marker in the `setting` table, removes it.

**Recommended companion practice: declare the new column in the `CREATE TABLE` as well.** Then a clean
install performs zero `ALTER` statements, and the guarded `ALTER` only exists to repair legacy tables.
Both paths then agree, and a fresh install is observable as "no DDL beyond table creation".

## Query conventions (`[verified]` from core code)

- Always `DB_PREFIX` in a raw query — no table name is ever written literally.
- `$this->db->escape()` for every string value.
- `(int)` cast for every numeric id before interpolation.
- Core's own `setting` model shows the house pattern:
  `"... WHERE store_id = '" . (int)$store_id . "' AND \`code\` = '" . $this->db->escape($code) . "'"`.

## ENFORCEABLE CHECKS

1. Every new table is created in `install()`, and `install()` can be run twice without error.
2. No `ALTER`/`CREATE`/`DROP` in any method reachable from a page load. Search the controllers for DDL and
   confirm each site is inside `install()`/`uninstall()` — **including cross-file calls** (a controller's
   `index()` calling a model's `upgradeSchema()` is the classic miss).
3. Every new column appears in the `CREATE TABLE` **and** in a guarded `ALTER` for legacy tables.
4. A read path with a schema check uses a `static $checked` guard, not a per-call query.
5. Every raw query contains `DB_PREFIX`; every string value is escaped; every id is cast.
6. `->num_rows` / `->row` / `->rows` are only read from SELECT results.
7. `uninstall()` does not drop tables holding merchant data without an explicit decision.

## Verification Notes

- Read directly: `system/library/db.php` (85 lines, full), `system/library/db/mysqli.php` (86 lines,
  full), `admin/model/setting/setting.php` (54 lines, full), and targeted sections of
  `install/opencart.sql` (`:10-14` table emission, the `ENGINE=` lines, `:1337` `oc_extension_install`).
- The `ENGINE=MyISAM` / `utf8` claim comes from the first several `ENGINE=` occurrences in the SQL file
  (`[verified]` for those tables). Whether **every** table in the file uses MyISAM was **not** confirmed
  across all ~200 tables — treat "all core tables are MyISAM" as `[baseline]`.
- The `oc_extension_install` column list is `[verified]`; other core table definitions were only
  spot-checked.
- `system/library/db/mpdo.php` was `[exists]`-checked only; its behaviour was not read, so adaptor
  differences (especially `escape()` and error handling) are `[needs-verification]`.
- The core `setting` table definition itself was not located in the SQL by name in this pass —
  its columns are `[verified]` from the model's queries, not from the `CREATE TABLE`.
- Not verified: a store's actual table engine after migration, and whether LiveStore ships alterations to
  the core schema.
