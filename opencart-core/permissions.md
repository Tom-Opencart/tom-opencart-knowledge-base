# OpenCart Admin Permissions & ACL (OC3 / LiveStore)

## Scope

Admin-side authorization: how an admin route is turned into a permission key, the `access` / `modify`
split, the core extension install/uninstall permission flow, and the escalation patterns that let a
low-privilege (e.g. demo) user reach a write path. Does not cover storefront authorization, customer
accounts, or the API token model.

## Baseline

- Version: LiveStore `v3.0.4.4` — re-checked against `v3.0.4.5` (identical files, see Verification Notes)
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/{tag}/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `verified`: the file was read in full or the exact cited block was read directly from baseline source
- `baseline`: a targeted grep or method listing confirmed the claim, body not read
- `exists`: existence-checked only
- `needs-verification`: informed inference, or behavior that may differ on a live store

## Main Routes

- `admin/index.php?route=extension/extension/module` — the module list; owns `install` / `uninstall`
- `admin/index.php?route=extension/extension/theme` — the theme list; same shape
- `admin/index.php?route=extension/module/<ext>` — a specific module controller (all its methods)
- `admin/index.php?route=extension/theme/<ext>` — a specific theme controller
- `admin/index.php?route=user/user_group` — permission editing UI
- `admin/index.php?route=error/permission` — the denial target; in the `$ignore` list

## Core Files

### Controllers

- `admin/controller/startup/permission.php` — the ONLY central authorization check (`access`, `[verified]`, read in full, 55 lines)
- `admin/controller/extension/extension/module.php` — module list; `install()` / `uninstall()` / `validate()` (`[verified]` lines 14-40 + 206-207)
- `admin/controller/extension/extension/theme.php` — theme list; same shape (`[verified]` by targeted grep: 13, 23-24, 35, 122-123)

### Models

- `admin/model/user/user_group.php` — `addPermission($user_group_id, $type, $route)` at :62, `removePermission(...)` at :74; `getUserGroup()` decodes the `permission` JSON at :22 (`[verified]`, exact lines read)

## Data Flow

1. Request arrives at `route=<r>`.
2. `startup/permission.php` builds the permission key from `explode('/', $r)` (`[verified]`).
3. It checks `access` for that key only; on failure it returns `new Action('error/permission')` (`[verified]` :50-52).
4. The controller runs. Any further authorization is the controller's own responsibility (`modify`).

## Database

### Tables

- `oc_user_group` — one row per admin user group; the `permission` column holds JSON (`[verified]`)

### Important Columns

- `oc_user_group.permission` — JSON shaped as `{"access":["route",...],"modify":["route",...]}`.
  `addPermission()` reads the row, decodes it, appends the route to the given `type` array and writes it
  back with `json_encode` (`[verified]` user_group.php:62-71).

## Route → permission key resolution (`[verified]`)

Source: `admin/controller/startup/permission.php` (read in full).

- `$part = explode('/', $route)` (:7); the key becomes `$part[0] . '/' . $part[1]` (:9-15).
- A third segment is appended ONLY when the two-segment prefix is in the `$extension` list (:18-37):
  `extension/advertise`, `extension/dashboard`, `extension/analytics`, `extension/captcha`,
  `extension/currency`, `extension/extension`, `extension/feed`, `extension/fraud`, `extension/module`,
  `extension/payment`, `extension/shipping`, `extension/theme`, `extension/total`, `extension/report`.
- `$ignore` (:40-48) skips the check entirely for `common/dashboard`, `common/login`, `common/logout`,
  `common/forgotten`, `common/reset`, `error/not_found`, `error/permission`.
- :50 — `if (!in_array($route, $ignore) && !$this->user->hasPermission('access', $route)) return new Action('error/permission');`

**Consequence — the method name is never part of the key.** `extension/module/vita_theme/install`
resolves to the key `extension/module/vita_theme`; the `install` segment is discarded. Every method of a
controller shares ONE permission row, so a route-level check can never distinguish a read method from a
destructive one. A single unprotected write method in a controller is therefore reachable by anyone who
may merely open that controller's page.

**Consequence — the router checks `access` only.** There is no `modify` check anywhere in
`permission.php`. `access` means "may open the page". `modify` is never enforced centrally: each write
path must check it itself.

## The two-tier model

- `access` — may open the page. Enforced centrally by the router.
- `modify` — may change data. NEVER enforced centrally. The controller must check it.

**Consequence — a demo / limited group typically HAS `access` and must NOT have `modify`.** Granting
`access` is how such a group sees anything at all. Therefore any write reachable through a GET page,
or reachable from a write method that trusts the router, is a privilege-escalation hole. Auditing a
demo grant means auditing every write path for its own `modify` check — not auditing the route list.

## Core extension install/uninstall flow (`[verified]`)

Source: `admin/controller/extension/extension/module.php` (lines 14-40 read; validate() at 206-207).

- `install()` :14 → `if ($this->validate())` :20 → `validate()` :206-207 requires
  `hasPermission('modify', 'extension/extension/module')`.
- On success: :21 registers the extension; :25-26 call
  `addPermission($this->user->getGroupId(), 'access'|'modify', 'extension/module/' . <ext>)`;
  :29 calls `load->controller('extension/module/' . <ext> . '/install')`.
- `theme.php` mirrors this: `install()` :13, `addPermission` :23-24, `uninstall()` :35, `validate()`
  :122-123 requiring `modify` on `extension/extension/theme`.

**Consequence — core grants the permission for you.** A module's own `install()` that grants itself
`access`/`modify` duplicates what core already did at :25-26 (and for a theme, :23-24). Such a
self-grant is redundant.

**Consequence — a direct HTTP call bypasses the wrapper.** Calling
`index.php?route=extension/module/<ext>/install` does NOT go through
`extension/extension/module::install()`. The router checks only `access` on
`extension/module/<ext>` — which a demo group usually holds. The extension's own `install()` /
`uninstall()` then runs with NO `modify` check unless it performs one itself.

**Recommended guard string.** Have `install()` / `uninstall()` check `modify` on the same key core
uses — `extension/extension/module` for modules, `extension/extension/theme` for the theme. Using the
identical key cannot block a legitimate install (core already required it one step earlier) and it
closes the direct-call hole.

**Consequence — core never revokes.** Core's `uninstall()` does not call `removePermission`, so
permission rows for a removed extension persist in `oc_user_group.permission` (`[verified]` for the
absent call; effect on the UI `[needs-verification]`).

## Escalation patterns to audit for

Every item below was observed in real third-party admin code; all share one root cause — a write that
the router does not gate and the controller does not gate either.

1. **Page-load side effects.** An `index()` (a GET page) that installs a sibling extension, runs
   schema DDL, or writes permissions.
2. **Self-granting helpers reachable from a page.** An `ensureXAccess()` called from `index()` that
   calls `addPermission()` for the current group, or for EVERY group that holds `access` on another
   route.
3. **Unconditional grants.** Granting `modify` to all groups that hold `access` — this converts every
   viewer into an editor and silently widens a demo group.
4. **Self-escalation via the caller's own group.** `addPermission($this->user->getGroupId(), ...)`
   reachable from a GET lets a low-privilege user widen its own group.
5. **Orphaned validators.** A `validateX()` method that is defined but never called; the write path it
   was written for runs unchecked.
6. **Guard placement.** A `modify` check that appears after the write, or in a branch that does not
   dominate it (e.g. inside an `if` that the write is not nested in).
7. **Install/uninstall without a guard.** See the direct-call consequence above.

## ENFORCEABLE CHECKS

(Requirement, structural check, and the audit recipe that decides them.)

- **Requirement.** Every admin controller method that performs a write must be protected by a
  dominating `hasPermission('modify', <route>)`, where `<route>` is the controller's own route or the
  core install key. A route-level `access` grant is NOT protection.
- **Structural check.** Parse method bodies by brace matching over a MASKED copy (comments and string
  contents blanked, offsets preserved), then analyse the RAW slice for the permission literal. Masking
  string contents blanks the interior of `'modify'`, so a permission search must run on raw text —
  otherwise every method looks unprotected.
- **Write evidence must not rely on a fixed entity whitelist.** Match mutating verbs on model calls
  (`add|edit|delete|remove|save|update|insert|replace|toggle|clear|install|uninstall|sort|...`), SQL
  keywords, `unlink(`, `file_put_contents`, `move_uploaded_file`, and settings writers. A whitelist of
  known entity names silently misses `deleteThing()`-style writes.
- **Verify every validator has a call site.** Grep the controller for each `validate*()` definition.
- **Positive-test the gate.** A checker is worthless unless it flags, at minimum: no check at all,
  `access` used instead of `modify`, and a check placed after the write. Confirm on synthetic cases
  before trusting a clean run.
- **Best practice for third-party controllers.** Add the `modify` check to every write method even
  where core already checks it, because the core wrapper is bypassable by a direct route call.

## Verification Notes

- Read in full from baseline source: `admin/controller/startup/permission.php` (55 lines).
- Read directly (cited lines): `admin/controller/extension/extension/module.php` lines 1-40 and 206-207;
  targeted grep confirmed `admin/controller/extension/extension/theme.php` lines 13, 23-24, 35, 122-123
  and `admin/model/user/user_group.php` lines 22, 62, 70, 74-82.
- Re-checked against `v3.0.4.5`: `permission.php`, `extension/extension/module.php` and
  `extension/extension/theme.php` are byte-identical between `v3.0.4.4` and `v3.0.4.5`, so the claims
  above hold on either tag (`[verified]`).
- NOT verified here: OpenCart 4.x (different controller/route conventions), the
  `marketplace/install.php` upload flow, `user/user_permission` (does a route exist? `[baseline]` only),
  and whether any specific live store deviates via a custom `startup/*` override.
- The escalation patterns are drawn from third-party admin controllers; they are described generically,
  and any given extension must still be re-checked against its own tree.
- Do not guess routes, methods, columns, events, Twig variables, XML anchors, or override behavior.
- If a point was not verified from source, schema, grep, or existence check, keep it explicitly unverified.
