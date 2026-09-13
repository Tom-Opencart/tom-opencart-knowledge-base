# OpenCart Packaging & Release Standards (OC3 / LiveStore)

## Scope

What an installable archive is, how the OpenCart installer treats it, why a manifest's identity is
load-bearing, what a partial (diff) archive can and cannot do, and the release hygiene that follows.
OCMOD manifest internals: `ocmod.md`. Verification gates: `testing-and-verification.md`.

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference

## 1. Archive shape

An OpenCart 3.x extension archive (`.ocmod.zip`) contains:

```
install.xml                 ← at the ZIP ROOT
upload/…                    ← mirrored onto the store root
```

Rules that follow directly from how the installer consumes it:

- Entry separators must be `/`. A `\` separator produces paths the installer does not resolve as intended.
- `install.xml` must sit at the archive root — not inside `upload/`.
- The `upload/` prefix is a **packaging convention**: it is stripped when files are moved onto the store
  root. A file at `upload/admin/controller/x.php` lands at `admin/controller/x.php`.

**Consequence — the archive layout is the deployment contract.** A file placed at the wrong depth installs
successfully and is never loaded, which then looks like a code bug rather than a packaging bug.

**Consequence — build determinism matters if archives are committed.** Fixed entry timestamps, entries
sorted by path, and normalised attributes make a rebuild byte-identical. Without that, every build produces
a diff even when nothing changed, and a CI rebuild-on-push would commit noise. `[baseline]` as a practice.

## 2. What the installer does (`[verified]`)

`admin/controller/marketplace/install.php`:

| Method | Line | Role |
| --- | --- | --- |
| `install()` | :3 | entry point |
| `unzip()` | :35 | extracts the archive to a temporary location |
| `move()` | :82 | recursively copies into the store root |
| `xml()` | :237 | registers/replaces the manifest and records the file list |
| `remove()` | :352 | temp cleanup |
| `uninstall()` | :417 | removes the files tracked for one install id |

**Consequence — files are COPIED, never DELETED, on install/update.** `unlink`/`rmdir` occur only in
`remove()` (temporary cleanup) and in the explicit `uninstall()`. Therefore:

- installing a newer version over an older one **leaves removed files behind**;
- a partial archive can never delete anything;
- a file deleted in your source still exists on an updated store and is still loaded.

**Consequence — every upload is tracked under its own install id.** `oc_extension_install` (`[verified]`
in `install/opencart.sql` :1337) has `extension_install_id` (PK), `extension_download_id`, `filename`,
`date_added` and a key on `extension_download_id`. Each upload becomes a separate row with its own tracked
file list, and `uninstall()` removes the files tracked for that id.

**Consequence — a partial archive is UPDATE-ONLY.** Because every upload has its own file list, uninstalling
a partial archive can delete files that are shared with the full archive and that other parts of the
installation still need. Rule: never uninstall a diff/partial archive; roll back with the full archive or a
backup.

## 3. Manifest identity — the `<code>` landmine (`[verified]`)

`xml()` :286 calls `getModificationByCode($code)`, and when a row is found :289 `deleteModification(...)`,
then :333 `addModification(...)`.

**Consequence — the manifest is identified by its `<code>`, and a changed `<code>` does not replace the old
registration.** The old row is not found, so it is not deleted; the new one is added. Two registered
modifications then carry the **same `<search>` blocks** and both apply to the same files — duplicate
injections and, in practice, a fatal breakage of the store.

**Rule: `<code>` is immutable for the lifetime of the extension.** Renaming it is a deliberate migration
that requires every store to first delete the old modification in the admin. See `ocmod.md`.

**Consequence — validation also rejects duplicates.** `validateForm()` :1370 requires `modify` on
`marketplace/modification` and rejects a duplicate **name** (:1387-1391) and a duplicate **code**
(:1397-1401), so a re-upload with an unchanged code is accepted (it replaces) while a colliding new name is
refused.

## 4. The Refresh step, and why it is not optional (`[verified]`)

The installer registers the manifest but the patched copies are produced later. `refresh()` :488 rebuilds
the modification cache; the sources include `DIR_SYSTEM . 'modification.xml'` :554 and
`glob(DIR_SYSTEM . '*.ocmod.xml')` :557, and **only rows whose `status` is truthy are applied** :568-569.
`system/startup.php` `modification()` then serves the patched copy whenever `DIR_MODIFICATION` contains one
(see `ocmod.md`).

**Consequence — shipping a new `install.xml` forces the operator to press "Refresh".** Until they do, the
old patched files are still served and the change appears not to have taken effect.

**Consequence — Refresh re-applies ALL cached modifications, not just yours.** If another installed
modification was broken, stale, or conflicting, a Refresh can surface that failure and it will look like
your release caused it. Warn the operator before they press it.

**Consequence — a disabled modification is silently not applied** (:568-569 status gate). "It installed but
nothing changed" is first a status question, then a Refresh question.

## 5. Release hygiene

1. **Edit source only.** Archives are build output. Never hand-edit a ZIP or a file inside it, and never
   treat an extracted copy as source.
2. **Rebuild the archive in the SAME commit as the source change** when archives are committed — otherwise
   the shipped artifact and the source disagree.
3. **Bump the version in every place it is recorded, in one commit** — the manifest `<version>`, any build
   constant in code, and the changelog. A version recorded in three places drifts.
4. **Ship a full archive as the base for every release**, plus optionally a diff for incremental updaters.
   Users who skipped versions take the full archive.
5. **A diff cannot**: delete a file, run a schema change, or be uninstalled. See §2.
6. **State the required post-install step** in the release notes: Refresh, or none.
7. **Verify on a live store before publishing.** Static gates prove structure, not behavior
   (`testing-and-verification.md`).

## 6. When a partial (diff) archive is safe

| Change | Partial safe? | Required |
| --- | --- | --- |
| File content edits (php/twig/js/css) | yes | ship the changed files |
| Manifest changed | conditional | ship the new `install.xml` and the operator presses Refresh |
| New table or column | **no** | the lifecycle hook must run → ship the full archive |
| File deletion | **no** | the installer never deletes → full archive or manual cleanup |
| New files referenced by a manifest | conditional | ship the files **and** the manifest |

The diff's contents are derived from version control, not hand-picked:
`git diff --name-only <prev_tag> <head_tag> -- upload install.xml`.

## 7. Landmines

1. The installer copies but never deletes — a partial update cannot remove a file.
2. Shipping a new manifest forces a Refresh that re-applies every cached modification.
3. **Lifecycle skew:** a refreshed manifest can run before the extension's `install()`; injected code must
   be a safe no-op until the extension is installed and enabled (`extension-development.md`).
4. Version skipping: always ship a full archive as a base.
5. Changing `<code>` creates a duplicate registration and can kill the store.
6. Diff archives are update-only; uninstalling one can delete shared files.

## ENFORCEABLE CHECKS

1. The archive contains `install.xml` at the root and `upload/…` entries, all with `/` separators.
2. `<code>` is byte-identical to the previous release — enforce this in the builder as a hard failure.
3. Every version site (manifest, build constant, changelog) carries the same value.
4. A partial archive contains only paths that changed between the two tags.
5. A release touching schema or deleting files ships a full archive.
6. A committed archive is reproducible: rebuilding identical source yields an identical file.
7. The release notes name the required post-install step.

## Verification Notes

- Read directly in this session: `admin/controller/marketplace/install.php` (`install()` :3, `unzip()` :35,
  `move()` :82, `xml()` :237, `getModificationByCode` call :286, `deleteModification` :289,
  `addModification` :333, `remove()` :352, `uninstall()` :417) and
  `admin/controller/marketplace/modification.php` (`refresh()` :488, sources :554/:557, status gate
  :568-569, `validateForm()` :1370, duplicate checks :1387-1391/:1397-1401).
- `install/opencart.sql` :1337 `oc_extension_install` column list — `[verified]`.
- The MyISAM/no-transaction consequence behind "a failed install leaves partial state" comes from
  `database-schema.md` (`[verified]` for the observed tables).
- Build determinism is recorded as `[baseline]` practice, corroborated by the release repository's own
  documented rules; it is not an OpenCart core behavior.
- `[needs-verification]`: whether `uninstall()` deletes only that install id's tracked files (the code path
  was read at method level, not line-by-line), and the exact behaviour when two registered modifications
  overlap.
- Not verified: marketplace/download-id semantics beyond the table columns, and how a store's
  `modification` cache is invalidated on a multi-store setup.