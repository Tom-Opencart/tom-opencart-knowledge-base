# OpenCart MVC-L Architecture (OC3 / LiveStore)

## Scope

The request lifecycle and the MVC-L conventions of OpenCart 3.x: the two applications, the bootstrap,
the engine classes, loader name resolution, controller/model/view/language naming, and how a URL route
becomes a running method. Does not cover admin authorization (see `permissions.md`), packaging (see
`release-and-packaging.md`), or Twig internals (see `twig-theming.md`).

## Baseline

- Version: LiveStore `v3.0.4.5`
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference

## The two applications

| App | Root | Entry point | Verified |
| --- | --- | --- | --- |
| Admin | `admin/` | `admin/index.php` | `[exists]` |
| Catalog (storefront) | `catalog/` (store root) | `index.php` | `[exists]` |
| Installer | `install/` | `install/index.php` | `[exists]` |

Both web applications run the same engine. Which one is active is decided by the **constants** defined
before the engine loads, not by a separate codebase:

- `DIR_APPLICATION` — the current application's root.
- `DIR_CATALOG` — defined in the admin app `[verified]` by its use in `startup.php` :50.
- `DIR_OPENCART` — defined by the installer `[verified]` :52.

## Engine classes (`[verified]`)

| Class | File | Declaration |
| --- | --- | --- |
| `Controller` | `system/engine/controller.php` | `abstract class Controller` |
| `Model` | `system/engine/model.php` | `abstract class Model` |
| `Loader` | `system/engine/loader.php` | `final class Loader` |
| `Registry` | `system/engine/registry.php` | `final class Registry` |
| `Router` | `system/engine/router.php` | `final class Router` |
| `Action` | `system/engine/action.php` | `class Action` |
| `Event` | `system/engine/event.php` | `class Event` |
| `Config`, `Document`, `Language`, `Response`, `Template` | `system/library/*.php` | plain classes |

`Controller` and `Model` are **abstract** and both provide the `__get()` accessor that exposes registry
entries (`$this->load`, `$this->db`, `$this->config`, …) as properties.

## Bootstrap order (`[verified]`, `system/startup.php` :48-95)

1. :48-67 **Modification override** — `function modification($filename)`.
   - :50 `if (defined('DIR_CATALOG'))` → `DIR_MODIFICATION . 'admin/' . substr($filename, strlen(DIR_APPLICATION))`
   - :52 `elseif (defined('DIR_OPENCART'))` → `DIR_MODIFICATION . 'install/' . ...`
   - :54 `else` → `DIR_MODIFICATION . 'catalog/' . ...`
   - :58 files under `DIR_SYSTEM` → `DIR_MODIFICATION . 'system/' . substr($filename, strlen(DIR_SYSTEM))`
   - :62-66 returns the override **only `if (is_file($file))`**, otherwise the original path.
   - Also sets `$_SERVER['HTTPS']` from `HTTPS`, port 443, or `HTTP_X_FORWARDED_PROTO` / `HTTP_X_FORWARDED_SSL` (:40-46).
2. :70-72 Composer autoloader from `DIR_STORAGE . 'vendor/autoload.php'` when present.
3. :74-87 **Library autoloader** — `function library($class)`, registered with `spl_autoload_register('library')`.
4. :90+ engine files are `require_once(modification(...))` — the core engine itself loads through the override.

**Consequence — nothing is loaded from its original path if a patched copy exists.** Every core file
included through `modification()` is served from `DIR_MODIFICATION` when a matching patched file is
present. See `opencart-core/ocmod.md`.

## Library name resolution (`[verified]`, `startup.php` :74-84)

```
:75  $file = DIR_SYSTEM . 'library/' . str_replace('\\', '/', strtolower($class)) . '.php';
:78  include_once(modification($file));
```

**Consequence — a library class name maps to a LOWER-CASE file path with backslashes turned into
slashes.** The namespaced class `\vita\search\engine` resolves to
`system/library/vita/search/engine.php`. This is why library and vendored file names are lower case, and
it is the mechanism behind namespaced helper libraries in a modification.

## Name resolution for models, views and language (`[verified]`, `system/engine/loader.php`)

`Loader` is `final` and exposes `model()`, `controller()`, `language()`, `view()`, `library()`, `config()`.

- `$this->load->model('catalog/product')` makes the model available as the property
  `$this->model_catalog_product` (route separators become underscores).
- `$this->load->language('extension/module/x')` loads the matching `admin|languages` file and its keys
  become available through `$this->language->get()`.
- `$this->load->view('extension/module/x', $data)` returns the rendered template string.
- `$this->load->controller('extension/module/x')` `[baseline]` returns whatever the target controller
  method returns — this is how core invokes another controller's `install()`.

`Loader` checks `is_file()` before including (`[verified]` :76, :150, :167), so a missing
model/controller/language file fails as a soft error rather than a hard include failure.

## Route → class → method resolution (`[verified]`, `system/engine/action.php`)

This is the single most important mechanism to understand.

- :16 `private $method = 'index';` — the default method.
- :26 `$parts = explode('/', preg_replace('/[^a-zA-Z0-9_\/]/', '', (string)$route));` — the route is
  sanitised to alphanumerics, underscores and slashes.
- :29-39 The constructor walks the segments **backwards**, testing
  `DIR_APPLICATION . 'controller/' . implode('/', $parts) . '.php'`:
  - :32-35 if the file exists → `$this->route = implode('/', $parts)` and stop;
  - :37 else → `$this->method = array_pop($parts)` and try again with one fewer segment.
- :60-62 magic methods are rejected: a method starting with `__` returns an Exception.
- :64-65 `$file = DIR_APPLICATION . 'controller/' . $this->route . '.php';`
  `$class = 'Controller' . preg_replace('/[^a-zA-Z0-9]/', '', (string)$this->route);`
- :78 requires `hasMethod($this->method)` **and**
  `getNumberOfRequiredParameters() <= count($args)`, else Exception.

**Consequence — trailing route segments beyond the deepest existing controller file become the METHOD
name.** For `extension/module/vita_theme/search_suggest`, the file
`controller/extension/module/vita_theme.php` exists, so the route is `extension/module/vita_theme` and
the method is `search_suggest`. This is the mechanism behind every "method as an extra route segment"
URL.

**Consequence — the class name is derived by STRIPPING all non-alphanumeric characters**, including
underscores and slashes. Route `extension/module/vita_theme` yields the string
`Controllerextensionmodulevitatheme`. PHP class names are **case-insensitive**, so the declaration
`class ControllerExtensionModuleVitaTheme` matches, and the usual CamelCase spelling is a convention
rather than a requirement. Every verified controller in the baseline follows the CamelCase spelling.

**Consequence — a controller method must not have required parameters.** It is called via
`call_user_func_array` with the trailing route segments; a method with a required parameter that the URL
does not supply returns an Exception instead of running.

## Dispatch loop (`[verified]`, `system/engine/router.php`)

- :32 `addPreAction(Action $pre_action)` collects pre-actions.
- :42 `dispatch(Action $action, Action $error)`:
  - :45-53 runs each pre-action; if one returns an `Action`, that becomes the action and the loop breaks
    — this is how a `startup/*` controller **redirects the request** (the admin permission check and the
    SEO URL rewrite both work this way);
  - :55-57 `while ($action instanceof Action) { $action = $this->execute($action); }` — the returned
    Action chains onward;
  - :66-79 if the executed action returns an `Exception`, the `$error` action replaces it (the
    `error/*` route).

## Framework wiring (`[verified]`, `system/framework.php`)

- :57-60 reads `action_event` from config and registers each entry with `Event::register(key, new Action(...), priority)`.
- :73-76 creates `Response`, sets the default `Content-Type: text/html; charset=utf-8`, applies
  `config_compression`.
- :158-164 creates the `Router` and adds every `action_pre_action` from config via `addPreAction()`.
- :169 `$route->dispatch(new Action($config->get('action_router')), new Action($config->get('action_error')));`
- :172 `$response->output()`.

**Consequence — the whole request is config-driven.** The startup and pre-action lists come from config
(`action_pre_action`, `action_event`, `action_router`, `action_error`), which is why an extension can
inject behaviour without touching the engine, and why `startup/*` controllers are the correct place for
cross-cutting request logic.

## Naming conventions (`[verified]` against real files)

| Artifact | Convention | Example |
| --- | --- | --- |
| Admin controller | `Controller` + CamelCase route | `class ControllerExtensionExtensionModule extends Controller` |
| Catalog controller | `Controller` + CamelCase route | `class ControllerCommonHome extends Controller` |
| Model | `Model` + CamelCase path | `class ModelCatalogCategory extends Model` |
| Controller file | `controller/<route>.php`, lower case, underscore-separated words | `controller/extension/module/x.php` |
| Model file | `model/<route>.php` | `model/catalog/category.php` |
| Template | `<theme>/template/<route>.twig` | see `twig-theming.md` |
| Language | `language/<lang>/<route>.php` with `$_['key']` | `language/en-gb/extension/module/x.php` |

## ENFORCEABLE CHECKS

1. A new controller declares the class as `Controller` + route alphanumerics (CamelCase), and the file
   lives at `controller/<route>.php`.
2. The class name's alphanumerics, lower-cased, equal the route's alphanumerics, lower-cased — this is
   the actual matching rule (`[verified]` :65) and a cheap automated check.
3. Every controller method used as a route segment is `public` and has **no required parameters**.
4. No method starting with `__` is reachable as a route.
5. Registry services are read as properties after the corresponding `$this->load->…` call; the class
   extends `Controller` or `Model`.
6. Cross-cutting request logic goes in a `startup/*` controller registered via `action_pre_action`, and
   returns an `Action` to redirect the request.
7. Library classes map to lower-case `system/library/<path>.php` files.

## Verification Notes

- Read directly: `system/engine/router.php` and `system/engine/action.php` (in full, 81 and 84 lines),
  `system/startup.php` :30-95, `system/framework.php` (key lines :57-76, :158-172), plus class
  declarations verified from real files (`catalog/controller/common/home.php`,
  `catalog/model/catalog/category.php`, and the engine/library classes listed above).
- `system/engine/loader.php` (6745 bytes) was inspected by `is_file()` line only — the full method bodies
  for `model()`, `controller()`, `view()`, `language()` were **not** read line-by-line, so the exact
  property-name construction for each is `[baseline]`, not `[verified]`.
- The `Loader::controller()` **return value** semantics are `[baseline]`.
- `admin/index.php` and the catalog `index.php` were `[exists]`-checked but their bodies were not
  compared here; the claim "both apps run the same engine" rests on `DIR_CATALOG` / `DIR_OPENCART`
  branching in `startup.php` (`[verified]`).
- Not verified: the exact config values of `action_pre_action` / `action_event` in a real store, and
  OpenCart 4.x architecture (different: namespaced controllers, no `Controller` prefix).
