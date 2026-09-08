# OpenCart 3 Cache: TTL semantics and expiry pitfalls

Verified against LiveStore 3.0.4.4 sources (`upload/system/library/cache.php`, `upload/system/library/cache/file.php`) and OpenCart 3.0.x. Base knowledge for any module that caches rendered output.

## Facts

- `Cache::__construct($adaptor, $expire = 3600)` — the TTL is fixed per Cache instance at startup (framework passes it once). It is **not per-call**.
- `Cache::set($key, $value)` takes exactly **two arguments**. A third "ttl" argument is silently ignored — no error, no warning. Code like `$this->cache->set($key, $data, 60)` does **not** create a 60-second cache entry.
- File adaptor `set()` encodes expiry into the **filename**: `cache.<key>.<time() + expire>`.
- File adaptor `get()` performs a prefix glob (`cache.<key>.*`), takes the **first** match, and returns its contents. It **never compares the filename expiry with `time()`** — expired cache files are served as fresh data indefinitely.
- `Cache::delete($key)` is a **prefix glob delete** (`cache.<key>.*`, dots preserved by the sanitizer `[^A-Z0-9\._-]`). So `delete('vita_extra_html')` removes both `cache.vita_extra_html.<md5>…` content entries and `cache.vita_extra_html.inject…`. `delete('*')` purges everything.
- Consequence: a cache entry created via the file adaptor lives until (a) overwritten by `set()` with the same key, (b) removed by `delete()` with a matching prefix, or (c) the whole cache directory is purged. The "expiry" in the filename is decoration unless something checks it.

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

2. Cached data that derives from mutable state (module rows, settings, visibility) needs an invalidation hook on every write path of that state. Stock routes that delete module rows (`marketplace/module/delete`) do **not** clear custom cache prefixes — a module that caches a DB snapshot of its instances must either keep the snapshot short-lived (rule 1) or revalidate against the DB periodically. An immortal snapshot here means a deleted module keeps rendering on the storefront.

3. Cache keys must include every dimension the cached value depends on: store id, language id, currency — and any visibility dimension computed inside the cached content (e.g. customer price visibility). Visibility checks themselves should run **before** the cache lookup so per-request state is never served from cache.
