# API

## Scope

This domain covers the OpenCart storefront API: key-based authentication (`api/login`), the
API session/token model, IP whitelisting, the `api/*` controllers (cart, order, customer,
payment, shipping, coupon, voucher, reward, currency), and the admin side that manages API
users (`user/api`).

It documents how the API authenticates and which endpoints exist; the business logic each
endpoint drives lives in the linked domain packs (Orders, Customers).

## Baseline

- Version: OpenCart 3.0.x (LiveStore build)
- Source: https://github.com/19th19th/LiveStore tag `v3.0.4.4` (`upload/...`)
- Verified on: 2026-07-04

## Confidence Levels

Claims carry an explicit confidence tag. Do not treat lower tiers as facts.

- `[verified]` — the cited source file was read in full (or the exact cited block was read) at `v3.0.4.4`.
- `[baseline]` — observed via a targeted grep of the real file (e.g. a `public function` list,
  a dir listing) but the whole file was not read.
- `[exists]` — path existence-checked at `v3.0.4.4` (HTTP HEAD); CONTENT not read.
- `[needs-verification]` — general OpenCart knowledge / inference, not confirmed against this baseline.

The authoritative confidence breakdown is in `## Verification Notes` at the end.

## Main Routes

### Storefront API endpoints (`catalog/controller/api/*`)

Directory listing verified at `v3.0.4.4` `[baseline]`; method lists via grep of each file
`[baseline]`; the `session.data['api_id']` guard count is quoted per file:

- `api/login` — `index` (no `api_id` guard; this is the auth entry point). `[verified]`
- `api/cart` — `add`, `edit`, `remove`, `products` (4 guards).
- `api/order` — `add`, `edit`, `delete`, `info`, `history` (5 guards). See Orders pack.
- `api/customer` — `index` (1 guard). See Customers pack.
- `api/payment` — `address`, `methods`, `method` (3 guards).
- `api/shipping` — `address`, `methods`, `method` (3 guards).
- `api/coupon` — `index` (1 guard).
- `api/voucher` — `index`, `add` (2 guards).
- `api/reward` — `index`, `maximum`, `available` (3 guards).
- `api/currency` — `index` (1 guard).

Build note: this LiveStore build uses consolidated `api/payment` and `api/shipping` controllers
(with `address`/`methods`/`method` methods). It does NOT ship the stock OpenCart 3.0.x split
files `api/payment_address`, `api/payment_method`, `api/shipping_address`, `api/shipping_method`,
nor `api/product` / `api/account` (all existence-checked absent). Re-verify on a non-LiveStore
baseline — vanilla 3.0.x differs here. `[baseline]` (from the verified dir listing)

### Admin
- `user/api` — admin CRUD for API users (Users → API). `[baseline]`

## Core Files

### Controllers

- `catalog/controller/api/login.php` — `ControllerApiLogin::index` (read in full `[verified]`):
  the authentication entry point (see Data Flow).
- `catalog/controller/api/{cart,order,customer,payment,shipping,coupon,voucher,reward,currency}.php`
  — the endpoint controllers; each method checks `$this->session->data['api_id']`. `[baseline]`
- `admin/controller/user/api.php` — admin API-user management controller. `[exists]`

### Models

- `catalog/model/account/api.php` — `ModelAccountApi` (runtime auth, read in full `[verified]`):
  - `login($username, $key)` — `SELECT * FROM api WHERE username = ? AND key = ? AND status = '1'`.
  - `addApiSession($api_id, $session_id, $ip)` — inserts `api_session`.
  - `getApiIps($api_id)` — the IP whitelist for the key.
- `admin/model/user/api.php` — `ModelUserApi` (admin management, read in full `[verified]`):
  `addApi`, `editApi`, `deleteApi`, `getApi`, `getApis`, `getTotalApis`, `addApiIp`, `getApiIps`,
  `addApiSession`, `getApiSessions`, `deleteApiSession`, `deleteApiSessionBySessionId`.

### Views / Language

- Admin views: `admin/view/template/user/api_form.twig`, `user/api_list.twig` `[exists]`
  (there is no `user/api.twig` — existence-checked as absent at `v3.0.4.4`; HTTP HEAD returned 404).
- Admin language: `admin/language/en-gb/user/api.php` `[exists]`.
- Endpoint responses are JSON; storefront language: `catalog/language/en-gb/api/*` `[needs-verification]`
  (only `api/login`, `api/order`, `api/customer` language loads were observed in read files).

## Data Flow

Authentication (`api/login::index`, read in full `[verified]`):

1. Client POSTs `username` + `key` (or just `key`, which defaults `username` to `'Default'`).
2. `ModelAccountApi::login()` matches an `api` row with `status = 1`.
3. The caller IP (`REMOTE_ADDR`) must be present in `api_ip` for that key (`getApiIps`); otherwise
   an `error.ip` is returned.
4. On success a new `Session($config->get('session_engine'), $registry)` is started,
   `addApiSession()` writes an `api_session` row, and `session.data['api_id']` is set.
5. The response returns `api_token` = the new session id.
6. Every subsequent `api/*` call carries `api_token` (which resolves that session); each endpoint
   method rejects the request unless `session.data['api_id']` is set.

Admin usage: the store API user id is configured via `config_api_id` (default `'1'`, verified in
`install/opencart.sql`). The admin order screen drives `api/order/history` through this API
(see the Orders pack, where the `order_info.twig` → `api/order/history` call is `[verified]`).

## Database

### Tables (verified `CREATE TABLE` read in `install/opencart.sql`) `[verified]`

- `api` (line 59): `api_id` (PK), `username` varchar(64), `key` text, `status` tinyint(1),
  `date_added`, `date_modified`.
- `api_ip` (line 76): `api_ip_id` (PK), `api_id`, `ip` varchar(40) — the per-key IP whitelist.
- `api_session` (line 90): `api_session_id` (PK), `api_id`, `session_id` varchar(32), `ip`,
  `date_added`, `date_modified`.

### Config

- `config_api_id` — default `'1'` (setting row at line 3352); the store API user the admin uses. `[verified]`

## Events

No default `oc_event` rows target the `api/*` controllers. `[baseline]` (no `api/` action/trigger
in the event dump). API endpoints instead call the standard domain models, so they fire those
models' events — e.g. `api/order/history` → `ModelCheckoutOrder::addOrderHistory` →
`addOrderHistory/before|after` → order mail/activity/statistics. `[verified]` (cross-ref Orders pack)

## API Links

This is the primary domain. Cross-references:
- `api/order` → Orders pack (`addOrder`, `editOrder`, `deleteOrder`, `addOrderHistory`).
- `api/customer` → Customers pack.
- `api/cart` / `api/payment` / `api/shipping` / `api/coupon` / `api/voucher` / `api/reward` /
  `api/currency` → the checkout flow.

## Mail Links

Indirect only. API actions trigger mail through the normal business flows they invoke (e.g.
`api/order/history` can send order confirmation/status mail). `[verified]` (via the order events)

## OCMOD Hotspots

- `catalog/controller/api/login.php` — auth/token issuing (custom auth, header-token adapters).
- Individual `api/*` controllers — request payload handling and the `api_id` guard.
- `catalog/model/account/api.php::login` — credential/IP validation.
- `admin/controller/user/api.php` + `admin/model/user/api.php` — API-user provisioning.

## Theme Override Risks

- Low. The API is response/JSON based, not theme-rendered. Risk appears only when a store mixes
  API logic into custom checkout modules. `[needs-verification]`

## Common Extension Collision Zones

- Mobile app / headless / PWA bridges (add or wrap `api/*` endpoints, custom token schemes).
- ERP / CRM / marketplace connectors (drive `api/order` / `api/customer`).
- Custom checkout integrations that call `api/cart` / `api/payment` / `api/shipping`.
- Anything replacing the `api_id`/session guard with a stateless token.

## Verification Notes

Verified against LiveStore `v3.0.4.4` on 2026-07-04. Confidence split by how each item was checked.

### `[verified]` — read in full from source
- `catalog/controller/api/login.php` (auth flow).
- `catalog/model/account/api.php` (`ModelAccountApi::login`/`addApiSession`/`getApiIps`).
- `admin/model/user/api.php` (`ModelUserApi`, full method set).
- `install/opencart.sql`: `oc_api` (59), `oc_api_ip` (76), `oc_api_session` (90) `CREATE TABLE`
  blocks and the `config_api_id` default (line 3352).

### `[baseline]` — confirmed by targeted grep / dir listing, not full read
- The `catalog/controller/api/` directory listing (cart, coupon, currency, customer, login,
  order, payment, reward, shipping, voucher).
- Per-controller `public function` lists and `session.data['api_id']` guard counts.
- No `api/*` trigger/action rows in the `oc_event` dump.

### `[exists]` — path existence-checked only (HTTP HEAD), CONTENT not read
- `admin/controller/user/api.php`, `admin/view/template/user/api_form.twig`,
  `user/api_list.twig`, `admin/language/en-gb/user/api.php`.
- Negative existence checks (HTTP HEAD returned 404): `api/payment_address`, `api/payment_method`,
  `api/shipping_address`, `api/shipping_method`, `api/product`, `api/account`,
  `admin/view/template/user/api.twig`.

### `[needs-verification]` — re-check before relying
- Exact request/response payloads of each `api/*` endpoint.
- Storefront `api/*` language files beyond the observed `login`/`order`/`customer`.
- On a non-LiveStore baseline: the `payment`/`shipping` vs split `*_address`/`*_method`
  controller layout, and whether `api/product`/`api/account` exist.
- The admin-side token-mint mechanism beyond the verified `config_api_id` + `api/order/history` call.
