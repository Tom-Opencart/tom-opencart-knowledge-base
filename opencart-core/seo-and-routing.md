# OpenCart SEO & Routing Standards (OC3 / LiveStore)

## Scope

How a URL becomes a route and how a route becomes a URL: the pre-action chains, the `Url` class, the
`seo_url` startup controller and its rewrite registration, query-string construction, and the SEO URL
storage model. The route→class→method mechanism itself is in `mvc-l-architecture.md`; do not duplicate it.

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
| `system/library/url.php` | `Url` — builds links (`[verified]`, 69 lines, full) |
| `catalog/controller/startup/seo_url.php` | the SEO URL pre-action (`[verified]` by targeted read) |
| `system/config/catalog.php` | the catalog pre-action chain (`[verified]`, 62 lines) |
| `system/config/admin.php` | the admin pre-action chain (`[verified]`, 49 lines) |
| `system/engine/router.php` | the dispatch loop (`[verified]`, 81 lines, full) |

## 1. The pre-action chains (`[verified]`)

**Catalog** (`system/config/catalog.php` :32-39), in order:

```
startup/session, startup/startup, startup/error, startup/event, startup/maintenance, startup/seo_url
```

**Admin** (`system/config/admin.php` :22-29), in order:

```
startup/startup, startup/error, startup/event, startup/sass, startup/login, startup/permission
```

Also `:32` `$_['action_default'] = 'common/dashboard';` for the admin app.

**Consequence — routing and authorization are pre-actions, and their ORDER is the security model.** In the
admin chain, `startup/login` (session/user + `user_token`) runs before `startup/permission` (route access).
Reordering them, or inserting a handler that ends the chain early, changes who gets to reach which route.
See `permissions.md` and `security.md` §3.

**Consequence — `startup/seo_url` runs in the CATALOG only.** SEO URLs do not affect admin routes, which is
why admin links always carry `index.php?route=…&user_token=…`.

**Consequence — a pre-action can replace the whole request.** `Router::dispatch` :45-53 runs pre-actions
first and, if one returns an `Action`, that action replaces the target and the loop breaks
(`router.php` :48-51). This is the mechanism behind both the permission redirect and the SEO rewrite. A
custom pre-action that returns an `Action` therefore redirects the request; returning nothing continues it.

## 2. `Url::link()` (`[verified]`, `system/library/url.php`)

- :48 `link($route, $args = '', $secure = false)`
- :49-53 the base is `$this->ssl` when an SSL URL is configured **and** `$secure` is true, otherwise
  `$this->url`; the result always starts `index.php?route=<route>`
- :55-61 argument handling — **two different, non-equivalent branches**:
  - an **array** → `$url .= '&amp;' . http_build_query($args)` :57
  - a **string** → `$url .= str_replace('&', '&amp;', '&' . ltrim($args, '&'))` :59
- :63-65 every registered rewrite is applied in registration order
- :35-37 `addRewrite($rewrite)` appends

**Consequence — array arguments produce an inconsistent separator.** `http_build_query()` joins pairs with
a raw `&`, and only the first separator is `&amp;`. The generated URL then mixes `&amp;` and `&`. If the
link is emitted into HTML this is tolerable; if it is used as a redirect target, an API call, or a
canonical/OG tag, it can be wrong. Pass a **string** when the URL is consumed by anything other than a
plain HTML `href`.

**Consequence — rewrites run on the ALREADY-BUILT string.** A rewrite receives a URL that already contains
`index.php?route=…&amp;…`, so a rewrite that assumes unencoded `&` will mis-parse it. `seo_url` handles this
by parsing with `parse_str` after `str_replace`, not by assuming clean input.

**Consequence — `$secure` only has an effect if an SSL URL is configured in the store.** A "secure" link on
a store without one is silently a normal link — not an error.

## 3. The `seo_url` pre-action (`[verified]` by targeted read)

`catalog/controller/startup/seo_url.php`:

- :9 `__construct($registry)`; :15 `index()`
- :17-18 **`if ($this->config->get('config_seo_url')) { $this->url->addRewrite($this); }`** — the controller
  registers **itself** as a `Url` rewrite. SEO URLs are therefore a URL-builder concern, not a router
  concern.
- :37 keyword lookup: `SELECT * FROM …seo_url WHERE keyword = '<keyword>'`
- :40 `$url = explode('=', $query->row['query']);` — the stored `query` value has the form `product_id=42`
- :62-63 if a query exists and its leading key is not one of the special-cased ids (information_id,
  manufacturer_id, …), then `$this->request->get['route'] = $query->row['query']`
- :94 `rewrite($link)` — the reverse direction, route → SEO URL
- :105 `parse_str($url_info['query'], $data)`
- :109 calls a **`seo_pro`** component: `$this->seo_pro->baseRewrite(...)` — LiveStore ships a `seo_pro`
  integration, and `:168` reads its `config_seopro_addslash` setting
- :116, :127 further `seo_url` lookups by `query` (including a `category_id=` form), appending
  `'/' . $query->row['keyword']` (:119, :130)
- :146-154 the leftover query string is rebuilt with `rawurlencode()` per pair (:150) and then
  `str_replace('&', '&amp;', trim($query, '&'))` (:154)
- :173 assembles `scheme://host(:port)…`

**Consequence — `seo_url` is the single point where request parsing and URL building meet, and it is
config-gated.** With `config_seo_url` off, the rewrite is never registered and plain
`index.php?route=…` links are produced — so any code that *depends on* SEO-path parsing must tolerate both
forms.

**Consequence — the `seo_url` table stores a raw query string per keyword.** Lookups happen in both
directions by `keyword` and by `query`; an entry whose `query` is malformed (no `=`) makes `explode()`
return a single element, so the leading-key test at :62 degrades. Data integrity in this table is
functionally load-bearing.

**Consequence — `seo_pro` behaviour is not core.** Parameter handling, trailing-slash behaviour
(`config_seopro_addslash` :168) and the produced path shape come from LiveStore's own component. Treat any
SEO claim as `[needs-verification]` on a store where that component differs.

## 4. Where a route comes from, and how a missing one behaves

- The route resolution (walking segments backwards until a controller file is found, extra segments
  becoming the method name) is `[verified]` in `mvc-l-architecture.md` — refer to it rather than repeating.
- A route with no matching controller produces an `Action` whose `execute()` returns an Exception
  (`action.php` :73, :81), which `Router::execute` (:73-79) converts into the configured error action.

**Consequence — a broken route is a redirect to the error route, not a crash**, so a mistyped route fails
quietly unless the error page is inspected.

## 5. Practical rules

- Always build links with `$this->url->link()`. Never hardcode `index.php?route=` in a template or
  controller: the rewrite chain, the SSL choice and the store URL are all bypassed.
- Pass **string** args when the URL is not plain HTML.
- Admin links must carry `user_token` (security.md §3).
- Do not register a rewrite that assumes a clean, unencoded query string.
- Treat `seo_url` table rows as load-bearing data; a keyword edit changes live URLs.
- Remember `startup/seo_url` is catalog-only; do not expect admin routes to be rewritten.

## ENFORCEABLE CHECKS

1. No hardcoded `index.php?route=` in new templates/controllers — links go through `Url::link()`.
2. Array arguments to `link()` are only used for HTML output contexts.
3. Any new pre-action is added to the correct chain (`system/config/*.php`) **after** the existing
   security pre-actions in admin.
4. A custom rewrite parses the query string safely (`str_replace` + `parse_str`, not raw `&` assumptions).
5. SEO-dependent code handles both the rewritten and the plain-`index.php` form.
6. No duplicate array key in a config's `action_event` (see `twig-theming.md`).

## Verification Notes

- Read directly and in full: `system/library/url.php` (69 lines). Read in full:
  `system/config/catalog.php` (62 lines), `system/config/admin.php` (49 lines).
- `catalog/controller/startup/seo_url.php` was inspected by **targeted line read** (the lines cited above),
  **not** read in full — the surrounding logic of `index()`/`rewrite()` is `[baseline]`. The `seo_pro`
  component's own file was not read at all — `[needs-verification]`.
- The `seo_url` **table definition** was not read from `install/opencart.sql`; the column names `keyword`
  and `query` are `[verified]` from the controller's SQL, not from the schema.
- The special-cased id list at :62 was seen partially (information_id, manufacturer_id); the complete list
  is `[needs-verification]`.
- `.htaccess` / web-server rewrite rules and the store's `config_seo_url` value were **not** inspected.
  Whether pretty URLs work at all depends on server configuration this pack does not cover.
- Not verified: canonical/hreflang handling, sitemap generation, and any LiveStore-specific SEO module
  beyond `seo_url` and `seo_pro`.