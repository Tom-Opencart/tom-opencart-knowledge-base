# OpenCart Events (OC3 / LiveStore)

## Event dispatch and missing controllers (verified LiveStore v3.0.4.4)

Source: `upload/system/engine/event.php` + `upload/system/engine/action.php`.

- `Event::trigger($event, $args)` iterates registered actions and calls `$value['action']->execute($registry, $args)`.
- `Action::execute()` checks `is_file(DIR_APPLICATION . 'controller/' . $route . '.php')`. If the file is missing it returns `new \Exception('Error: Could not call ...')` — it does NOT include the file, does NOT fatal, does NOT log a PHP error.
- `Event::trigger()` only propagates a non-null, non-Exception result: `if (!is_null($result) && !($result instanceof Exception)) return $result;`. Exceptions are silently ignored.

**Consequence:** an `oc_event` row pointing to a deleted controller/module is a silent safe no-op at runtime. A theme or package upgrade that removes a module's files does not break stores that forgot to run the module's `uninstall()` first — orphan events just do nothing (minor wasted dispatch only). Schema tables and settings left behind by such a store are likewise inert as long as no remaining code reads them.

## Module lifecycle contract

- `install()` should create schema/settings and register events via `model_setting_event->addEvent($code, $trigger, 'path/controller/method')`.
- `uninstall()` must call `deleteEventByCode($code)` and drop its own schema — this is the clean removal path; the no-op behavior above is only a safety net, not a substitute.
- Trigger patterns support `*` and `?` wildcards (converted to `.*` / `.` over `preg_quote`).

## Practical notes

- The event action route is relative to `controller/` without the `.php` suffix; the class name is derived as `Controller` + route with non-alphanumerics stripped.
- Magic methods (`__`-prefixed) are rejected by `Action::execute`.
- Guard `getNumberOfRequiredParameters() <= count($args)`: handlers must declare optional parameters or they silently become Exceptions (ignored for before/after triggers, but the module's intended view-data mutation will not happen).
