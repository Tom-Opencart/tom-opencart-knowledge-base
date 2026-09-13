# OpenCart Security Standards (OC3 / LiveStore)

## Scope

The baseline's actual defences for SQL injection, XSS, request authenticity (CSRF), credentials, session
and file uploads — and the practical rules that follow for extension code. Authorization is covered by
`permissions.md`; do not duplicate it here.

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference

## 1. SQL injection (`[verified]`)

- The DB layer exposes **only** `query(string $sql)` — there is **no prepared-statement or parameter-binding
  API**. See `system/library/db/mysqli.php` :24 and the `DB` wrapper `system/library/db.php` :44.
- `escape($value)` is `real_escape_string()` and maps `null` to `''` (`mysqli.php` :53-55).
- Core's own pattern, from `admin/model/setting/setting.php` :6:
  `"SELECT * FROM " . DB_PREFIX . "setting WHERE store_id = '" . (int)$store_id . "' AND \`code\` = '" . $this->db->escape($code) . "'"`

**Consequence — SQL safety is entirely the caller's responsibility.** There is no framework layer to save a
mistake: every string interpolated into SQL must be escaped, and every numeric id must be cast. A missing
`escape()` is a direct injection.

**Consequence — `(int)` casting is a security control, not a style choice.** For ids and numeric filters it
is the only correct form; `escape()` on a value used unquoted in SQL is not sufficient protection.

**Consequence — `escape(null)` yields `''`, not `NULL`.** Escaping a nullable value silently converts it,
so a column meant to store `NULL` receives an empty string.

**Consequence — a query error throws an Exception containing the full SQL** (`mysqli.php` :49). In a store
with display errors on, that leaks query text and structure to the browser. Error output must not be
displayed in production.

## 2. XSS and output escaping (`[verified]` — the most important finding)

### Twig autoescape is OFF

`system/library/template/twig.php` :23-28 sets the environment explicitly:

```
'autoescape'  => false,
'debug'       => false,
'auto_reload' => true,
'cache'       => DIR_CACHE . 'template/'
```

**Consequence — `{{ variable }}` in a Twig template does NOT escape output.** In this baseline, printing
untrusted data with `{{ }}` renders raw HTML and is a direct XSS vector. Escaping must be explicit:
`{{ value|escape }}` (or `|e`). This contradicts the default expectation carried over from standard Twig,
where autoescape is on unless disabled.

**Consequence — a reviewer must treat every `{{ }}` of user-controlled data as a defect until the escape
filter is present.** Store-review fields, customer names, addresses, search terms, and any admin-entered
HTML are all in scope.

### Input is HTML-escaped on the way IN

`system/library/request.php` :39-51 — `clean()` recurses through each superglobal and applies
`htmlspecialchars($data, ENT_COMPAT, 'UTF-8')` to every scalar, **including array keys and `$_SERVER`**
(:44 cleans the key as well as the value). All six superglobals pass through it in the constructor (:25-30).

**Consequence — `$this->request->get/post/…` values are already HTML-escaped.** Two consequences follow:

1. Escaping again on output **double-encodes** (`&amp;lt;`), so blindly adding `|escape` to a value that
   came from the request can corrupt it. Decide once where escaping happens.
2. `htmlspecialchars` is **not** SQL escaping and provides **no** SQL-injection protection. The two
   defences are independent and both are required.

**Consequence — route/query handling sees encoded text.** This is the origin of the QUERY_STRING encoding
behaviour documented in `opencart-core/request-response.md`; read that pack before parsing raw
`QUERY_STRING`.

## 3. Request authenticity / CSRF (`[verified]`)

`admin/controller/startup/login.php` enforces the token **centrally**, for every admin request:

- :13 `$this->registry->set('user', new Cart\User($this->registry));`
- :15-17 if the user is not logged in and the route is not in the first ignore list, the request is
  redirected to `common/login` — `return new Action('common/login')`.
- :19-31 the second ignore list is `common/login`, `common/logout`, `common/forgotten`, `common/reset`,
  `error/not_found`, `error/permission`.
- :29 the check:
  `if (!in_array($route, $ignore) && (!isset($this->request->get['user_token']) || !isset($this->session->data['user_token']) || ($this->request->get['user_token'] != $this->session->data['user_token']))) {`
  → redirect to `common/login`.
- :33-35 when no `route` is present at all, the token is still required.

**Consequence — the `user_token` is the baseline's CSRF protection, and it is enforced centrally.** An
extension does not add its own token field; it must **propagate** the existing one. Every admin link and
every admin form must carry `user_token`, or the request is rejected — and the failure looks like a
mysterious redirect to the login page, not an error message.

**Consequence — the token travels in the query string** (`?route=...&user_token=...`), so it appears in
server logs, referrers and browser history. Treat it as a credential for the session, never hardcode or
log it. Also note the comparison uses loose `!=` (`[verified]` :29).

## 4. Credentials (`[verified]`)

`system/library/cart/user.php` — `class Cart\User` (namespace `Cart`):

- :43 admin login SQL compares the password as
  `password = SHA1(CONCAT(salt, SHA1(CONCAT(salt, SHA1('<pwd>')))))` **OR** `password = '<md5>'`.
- **Consequence — an MD5 fallback is still accepted for admin logins.** Any code that creates or migrates
  an admin user must never write an MD5 password, and a hash-upgrade path is a legitimate security task.
- :17-39 the session user is loaded with `status = '1'`; if the row is gone the library calls `logout()`
  :37. The constructor also writes the client IP into `oc_user.ip` :25 on every request.
- :75-81 `hasPermission($key, $value)` returns `in_array($value, $this->permission[$key])`, and `false`
  when the key is absent. **Consequence — an unknown permission type denies by default**: a group with no
  `modify` array at all fails every `modify` check. Deny-by-default is the behaviour to preserve.

Storefront customer login is in `system/library/cart/customer.php` :50
(`login($email, $password, $override = false)`); its password comparison was **not** read in full —
`[needs-verification]`.

## 5. File uploads (`[baseline]`)

`admin/controller/tool/upload.php`:

- :315 `upload()`; :322 rejects with `error_permission` when the permission check fails;
  :331 `error_filename`; :334-337 reads the **allowed extension list from configuration**
  (`config_file_ext_allowed`, newline-separated) and builds the `$allowed` array.
- :294 the download path checks `file_exists($file) && filesize($file) > 0`.

**Consequence — the upload extension whitelist is configuration-driven**, so its strength depends on the
store's settings, not on the code alone. An extension that accepts uploads must apply its own explicit
whitelist rather than assuming the store's list is strict.

**Consequence — no MIME/content validation was observed in this pass.** Validation appears to be
extension-based; do not claim content-type safety. Tagged `[baseline]` because the full `upload()` body was
not read line-by-line.

## 6. Security defect classes most often introduced

| Defect | Catch |
| --- | --- |
| Unescaped string interpolated into SQL | every query inspected for `escape()` / `(int)` |
| `{{ }}` of untrusted data with autoescape off | search templates for data output lacking `\|escape` |
| Double-escaping request data on output | trace each value's escaping path once |
| Admin link/form missing `user_token` | grep generated links and forms for `user_token` |
| Write path reachable without a `modify` check | see `permissions.md` |
| SQL or query text surfaced in an error page | confirm display_errors is off in production |
| Raw `QUERY_STRING` re-parsed | see `request-response.md` |
| Upload accepted on extension alone | explicit whitelist in the extension itself |

## ENFORCEABLE CHECKS

1. Every string in a raw query is wrapped in `$this->db->escape()`; every id is `(int)`-cast.
2. Every admin URL and form carries `user_token`.
3. Every state-changing admin path is additionally gated by a `modify` permission check (central token alone
   is not authorization).
4. No template outputs untrusted data without an explicit escape filter.
5. No new code writes an MD5 password.
6. Uploads validate an explicit extension whitelist; no upload path is reachable without a permission check.
7. No SQL text can reach the browser.

## Verification Notes

- Read directly: `system/library/request.php` (51 lines, full), `system/library/cart/user.php` (97 lines,
  full), `admin/controller/startup/login.php` (37 lines, full), `system/library/template/twig.php` (44
  lines, full), `system/library/db/mysqli.php` (`escape()` :53-55, error path :49).
- `admin/controller/tool/upload.php` was inspected by targeted grep — the `upload()` validation **body was
  not read in full**, so the whitelist mechanics and any size/MIME checks are `[baseline]`.
- `system/library/session.php` and the session adaptor `system/library/session/db.php` were collected but
  **not read** — session cookie flags and lifetime are `[needs-verification]` and deliberately not claimed.
- Customer password hashing (`system/library/cart/customer.php`) — `[needs-verification]`.
- The default values of `config_file_ext_allowed` were not inspected.
- Not verified: whether the store disables `display_errors`, and any LiveStore-specific security hardening
  beyond the files read.
