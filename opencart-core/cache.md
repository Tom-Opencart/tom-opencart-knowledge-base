# OpenCart 3 Cache: TTL semantics and expiry pitfalls

## Baseline

- Version: LiveStore `v3.0.4.5`.
- Source: [Cache wrapper](https://github.com/19th19th/LiveStore/blob/v3.0.4.5/upload/system/library/cache.php), [file adaptor](https://github.com/19th19th/LiveStore/blob/v3.0.4.5/upload/system/library/cache/file.php), and [framework bootstrap](https://github.com/19th19th/LiveStore/blob/v3.0.4.5/upload/system/framework.php#L116).
- Verified on: 2026-10-19. Wrapper and file adaptor read in full; bootstrap cache construction read directly.
- Confidence: `[verified]` for the wrapper signatures, file expiry, and prefix deletion described below. Project-specific examples and unrelated module deletion paths retain their previous verification scope.

Base knowledge for any module that caches rendered output.

## Facts

- `Cache::__construct($adaptor, $expire = 3600)` — the TTL is fixed per Cache instance at startup (framework passes it once). It is **not per-call**.
- `Cache::set($key, $value)` takes exactly **two arguments**. A third "ttl" argument is silently ignored — no error, no warning. Code like `$this->cache->set($key, $data, 60)` does **not** create a 60-second cache entry.
- File adaptor `set()` encodes expiry into the **filename**: `cache.<key>.<time() + expire>`.
- File adaptor `__construct()` scans `DIR_CACHE . 'cache.*'` and removes files whose filename expiry is less than `time()` (`file.php` :9-20). The normal framework creates the configured cache at bootstrap (`framework.php` :116), so file expiry is checked at adaptor construction. `get()` itself performs a prefix glob (`cache.<key>.*`), takes the first match, and does not recheck expiry. A long-running instance can therefore read an entry that expires after its constructor ran; normal request bootstrap must not be described as serving expired files indefinitely.
- `Cache::delete($key)` is a **prefix glob delete** (`cache.<key>.*`, dots preserved by the sanitizer `[^A-Z0-9\._-]`). So `delete('vita_extra_html')` removes both `cache.vita_extra_html.<md5>…` content entries and `cache.vita_extra_html.inject…`. `delete('*')` purges everything.
- Consequence: a file cache entry is removed by expiry cleanup at a later adaptor construction, overwrite via `set()`, matching prefix deletion, or a whole-cache purge. Its filename expiry is functional. A separate `new Cache('file', $ttl)` can set a different TTL for its writes, but its constructor also scans the shared cache directory.

## Practical rules

1. Need a per-entry TTL that is shorter than the global expire? Store the deadline inside the value and enforce it yourself:

```php
$cached = $this->cache->get($key);
if (is_array($cached) && isset($cached['expires'], $cached['content'])) {
    if ($cached['expires'] === 0 || $cached['expires'] > time()) {
        return $cached['content'];
    }
}
// … rebuild …
$this->cache->set($key, array('expires' => time() + $ttl, 'content' => $output));
```

This is exactly the pattern used by `catalog/controller/extension/module/vita_extra_html.php` (`index()`), which also supports `ttl = -1` as "never expires" via `expires = 0`.

2. Cached data that derives from mutable state (module rows, settings, visibility) needs an invalidation hook on every write path of that state. Stock routes that delete module rows (`marketplace/module/delete`) do **not** clear custom cache prefixes — a module that caches a DB snapshot of its instances must either keep the snapshot short-lived (rule 1) or revalidate against the DB periodically. A stale snapshot can keep a deleted module rendering until that snapshot is invalidated or expires.

3. Cache keys must include every dimension the cached value depends on: store id, language id, currency — and any visibility dimension computed inside the cached content (e.g. customer price visibility). Visibility checks themselves should run **before** the cache lookup so per-request state is never served from cache.
