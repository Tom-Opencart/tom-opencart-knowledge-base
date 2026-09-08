# OpenCart Request / Response (OC3 / LiveStore)

## QUERY_STRING is htmlspecialchars-encoded (verified LiveStore v3.0.4.4)

Source: `upload/system/library/request.php` (`Request::clean()`), `upload/system/library/url.php`, `upload/catalog/controller/startup/seo_url.php`, `upload/system/library/response.php`.

- `Request::__construct()` runs `clean()` over `$_GET`, `$_POST`, `$_REQUEST`, `$_SERVER`, `$_FILES`, `$_COOKIE`. `clean()` applies `htmlspecialchars($data, ENT_COMPAT, 'UTF-8')` to every scalar value — including `$_SERVER['QUERY_STRING']`.
- Therefore `$this->request->server['QUERY_STRING']` is NOT the raw query string: every `&` inside it arrives as `&amp;`.
- `parse_str()` splits on the literal `&`, so keys after the first parameter come back as `amp;param` instead of `param`.

**Consequence:** building a redirect from `parse_str($this->request->server['QUERY_STRING'], $params)` produces parameter keys like `amp;filter_manufacturers`. When that array reaches `http_build_query()` and a `seo_url::rewrite()`, the bogus keys are rawurlencoded into `amp%3Bparam` and the `Location` header lands on an unfiltered page (e.g. `/search/?amp%3Bfilter_manufacturers=1`). Read parsed request data instead — `$this->request->get` already contains clean keys. The stock round-trip is safe on its own: `Url::link()` emits `&amp;` separators, `seo_url::rewrite()` strips them before `parse_str`, and `Response::redirect()` decodes `&amp;` again before sending the header, so none of these layers needs a manual workaround — the only corrupting step is re-parsing the escaped QUERY_STRING.

## Redirect and URL hygiene

- `Response::redirect($url, $status = 302)` sends `header('Location: ' . str_replace(array('&amp;', "\n", "\r"), array('&', '', ''), $url), true, $status)` and exits — it normalizes single-level `&amp;` back to `&`.
- `Url::link($route, $args, $secure)` HTML-escapes `&` into `&amp;` in string args before registered rewrites run; the incoming SEO redirect (`startup/seo_url`) rebuilds non-SEO route URLs from `$this->request->get`, which is why bare-route chains (`index.php?route=...&a=1&b=2`) preserve parameters cleanly while a controller that re-parses the escaped QUERY_STRING does not.
- When a legacy route must permanently move, prefer the stock pattern: read `$this->request->get`, drop `route`, and redirect via `$this->url->link(...)` with an explicit `http_build_query($params, '', '&')` separator so a non-default `arg_separator.output` cannot corrupt the target.
