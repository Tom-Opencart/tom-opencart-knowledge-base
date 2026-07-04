# Orders

## Scope

This domain covers order creation, order persistence, order retrieval, order totals,
status transitions, order history writes, invoice numbering, and the links from orders
into checkout, mail, statistics, stock, reward/affiliate, and the storefront API.

It excludes returns (`return`), the cart engine itself, and the individual payment/shipping
gateway logic (only their touch point with `addOrderHistory` is covered here).

## Baseline

- Version: OpenCart 3.0.x (LiveStore build)
- Source: https://github.com/19th19th/LiveStore tag `v3.0.4.4` (`upload/...`)
- Verified on: 2026-07-04

## Confidence Levels

Claims in this pack carry an explicit confidence tag. Do not treat lower tiers as facts.

- `[verified]` — the cited source file was read in full (or the exact cited block was read)
  at `v3.0.4.4`; method signature / SQL / line is quoted from real source.
- `[baseline]` — observed directly in the baseline via a targeted grep of the real file
  (e.g. a specific event row, a `public function` list) but not the whole file was read.
- `[exists]` — the path was existence-checked at `v3.0.4.4` (HTTP HEAD) but its CONTENT was
  not read; treat filename as real, contents as unconfirmed.
- `[needs-verification]` — general OpenCart knowledge / inference, not confirmed against this
  baseline. Verify before relying on it.

The per-section text below is prose; the authoritative confidence breakdown is in
`## Verification Notes` at the end.

## Main Routes

### Storefront checkout
- `checkout/cart`
- `checkout/checkout`
- `checkout/confirm` — assembles order data and writes the order (status 0)
- `checkout/success` — clears cart/session after a gateway confirms

### Storefront account
- `account/order` — order history list
- `account/order/info` — single order view
- `account/order/reorder`

### Admin
- `sale/order` controller methods: `index`, `add`, `edit`, `delete`, `getForm`, `info`,
  `createInvoiceNo`, `addReward`, `removeReward`, `addCommission`, `removeCommission`,
  `history`, `invoice`, `shipping`

### Storefront API
- `api/order/add`
- `api/order/edit`
- `api/order/delete`
- `api/order/info`
- `api/order/history`

Note: the admin order screen is directly tied to `api/order/history` via `api_token`.
Do not assume the full admin UI uses every `api/order/*` endpoint listed above without re-checking the controller/template flow.

### Order mail side effect (event-driven)
- `mail/order` (`index`/`add`/`edit`) and `mail/order/alert`

## Core Files

### Controllers

- `catalog/controller/checkout/confirm.php` — `ControllerCheckoutConfirm::index()`
  builds `$order_data`, runs the enabled `total` extensions, then calls
  `$this->model_checkout_order->addOrder($order_data)` and stores `session.data['order_id']`.
  It does NOT set an order status (order is written with `order_status_id = 0`).
- `catalog/controller/checkout/success.php` — `ControllerCheckoutSuccess::index()`
  clears the cart and unsets checkout session keys; renders `common/success`.
- `catalog/controller/account/order.php` — `ControllerAccountOrder`: `index`, `info`, `reorder`.
- `catalog/controller/api/order.php` — `ControllerApiOrder`: `add`, `edit`, `delete`,
  `info`, `history` (guarded by `session.data['api_id']`).
- `admin/controller/sale/order.php` — `ControllerSaleOrder` (routes listed above). The
  admin order screen has no direct write model; `getForm()`/`info()` expose
  `$data['api_token'] = $session->getId()` and `$data['store_url'] = HTTPS_CATALOG/HTTP_CATALOG`
  `[baseline]` (grep of `api_token`/`store_url` in the controller). Status changes are POSTed by
  AJAX to the catalog `api/order/history`: `admin/view/template/sale/order_info.twig:581`
  calls `{{ catalog }}index.php?route=api/order/history&api_token={{ api_token }}&store_id={{ store_id }}&order_id={{ order_id }}`
  after fetching the token `[verified]`.
- `catalog/controller/mail/order.php` — `ControllerMailOrder` (see Mail Links).
- Payment gateways call `addOrderHistory` to move status 0 → first real status, e.g.
  `catalog/controller/extension/payment/cod.php::confirm()` calls
  `$this->model_checkout_order->addOrderHistory($this->session->data['order_id'], $this->config->get('payment_cod_order_status_id'))`.

### Models

- `catalog/model/checkout/order.php` — `ModelCheckoutOrder` (the write-side authority):
  - `addOrder($data)` → returns `order_id`; inserts `order`, `order_product`, `order_option`,
    `order_voucher`, `order_total`.
  - `editOrder($order_id, $data)` → voids first via `addOrderHistory($order_id, 0)`, then rewrites rows.
  - `deleteOrder($order_id)` → voids via `addOrderHistory($order_id, 0)`, deletes order + related rows.
  - `getOrder($order_id)`, `getOrderProducts($order_id)`,
    `getOrderOptions($order_id, $order_product_id)`, `getOrderVouchers($order_id)`,
    `getOrderTotals($order_id)`.
  - `addOrderHistory($order_id, $order_status_id, $comment = '', $notify = false, $override = false)`
    — status transition engine: runs anti-fraud, redeems/unconfirms coupon/voucher/reward totals,
    subtracts/restocks stock, adds/removes affiliate commission, updates `order.order_status_id`
    + `date_modified`, inserts `order_history`, clears `product` cache.
- `admin/model/sale/order.php` — `ModelSaleOrder` (read/report side, no order writes):
  `getOrder`, `getOrders`, `getOrderProducts`, `getOrderOptions`, `getOrderVouchers`,
  `getOrderVoucherByVoucherId`, `getOrderTotals`, `getTotalOrders`, `getTotalOrdersByStoreId`,
  `getTotalOrdersByOrderStatusId`, `getTotalOrdersByProcessingStatus`,
  `getTotalOrdersByCompleteStatus`, `getTotalOrdersByLanguageId`, `getTotalOrdersByCurrencyId`,
  `getTotalSales`, `createInvoiceNo($order_id)`, `getOrderHistories($order_id, $start = 0, $limit = 10)`,
  `getTotalOrderHistories`, `getTotalOrderHistoriesByOrderStatusId`,
  `getEmailsByProductsOrdered`, `getTotalEmailsByProductsOrdered`.
- `catalog/model/account/order.php` — `ModelAccountOrder` (storefront read side):
  `getOrder`, `getOrders($start = 0, $limit = 20)`, `getOrderProduct`, `getOrderProducts`,
  `getOrderOptions`, `getOrderVouchers`, `getOrderTotals`, `getOrderHistories`, `getTotalOrders`,
  `getTotalOrderProductsByOrderId`, `getTotalOrderVouchersByOrderId`.

### Views

- Admin: `admin/view/template/sale/order_list.twig`, `order_info.twig`, `order_form.twig`,
  `order_history.twig`.
- Storefront checkout: `catalog/view/theme/default/template/checkout/confirm.twig`;
  success renders `catalog/view/theme/default/template/common/success.twig`.
- Storefront account: `catalog/view/theme/default/template/account/order_list.twig`,
  `account/order_info.twig`.
- Order mail (see Mail domain): `catalog/view/theme/default/template/mail/order_add.twig`,
  `mail/order_edit.twig`, `mail/order_alert.twig`.

### Language Files

- Admin: `admin/language/en-gb/sale/order.php`.
- Storefront checkout: `catalog/language/en-gb/checkout/checkout.php`,
  `catalog/language/en-gb/checkout/success.php`.
- Storefront account: `catalog/language/en-gb/account/order.php`.
- API: `catalog/language/en-gb/api/order.php`.
- Order mail: `catalog/language/en-gb/mail/order_add.php`, `mail/order_edit.php`, `mail/order_alert.php`.

## Data Flow

1. `checkout/confirm` runs enabled `total` extensions, assembles `$order_data`, and calls
   `ModelCheckoutOrder::addOrder()` → order row written with `order_status_id = 0`.
2. Customer is sent to the payment gateway; on confirmation the gateway calls
   `ModelCheckoutOrder::addOrderHistory($order_id, $status_id)`.
3. `addOrderHistory` fires events. The `before` event runs the mail controller while the OLD
   status is still readable (0 → confirmation email); anti-fraud may override the status.
4. When status enters a processing/complete status: coupon/voucher/reward totals are `confirm`ed,
   stock is subtracted, affiliate commission is added.
5. `order.order_status_id` + `date_modified` are updated and a row is inserted into `order_history`.
6. When status leaves processing/complete: restock + `unconfirm` totals + remove commission.
7. `checkout/success` clears the cart/session (does not touch the order).

## Database

### Tables (all created in `install/opencart.sql`, MyISAM) `[verified]`

`CREATE TABLE` statements for the following were read directly in `install/opencart.sql`
at `v3.0.4.4` (line numbers are for that file):

- `order` — main order row (line 2121).
- `order_history` — status change log (line 2195).
- `order_option` — chosen product options per line item (line 2214).
- `order_product` — line items (line 2235).
- `order_recurring` (line 2258) / `order_recurring_transaction` (line 2290) — recurring profiles.
- `order_status` — status id → name per `language_id`, composite PK `order_status_id,language_id` (line 2349).
- `order_total` — total rows; `code` values (`sub_total`, `shipping`, `tax`, `total`,
  coupon/voucher/reward, …) come from the enabled `total` extensions `[needs-verification]`, not
  from the schema (line 2397).
- `order_voucher` — gift vouchers purchased in the order (line 2415).
- `order_shipment` (line 2306) + `shipping_courier` (line 2322, with 6 seed couriers at line 2334)
  — present in this LiveStore build's `install/opencart.sql` and used by `sale/order::shipping`
  (tracking number + courier). `[verified]` These are NOT in vanilla OpenCart 3.0.x core; treat as
  a LiveStore build extra and re-confirm on any non-LiveStore baseline.

### Important Columns

- `order`: `order_id` (PK), `invoice_no`, `invoice_prefix`, `store_id`, `store_name`, `store_url`,
  `customer_id`, `customer_group_id`, `firstname`, `lastname`, `email`, `telephone`, `fax`,
  `custom_field` (JSON text), full `payment_*` and `shipping_*` address blocks
  (incl. `*_custom_field` JSON, `*_address_format`, `*_method`, `*_code`), `comment`,
  `total` decimal(15,4), `order_status_id` (default 0), `affiliate_id`, `commission`,
  `marketing_id`, `tracking`, `language_id`, `currency_id`, `currency_code`,
  `currency_value` decimal(15,8), `ip`, `forwarded_ip`, `user_agent`, `accept_language`,
  `date_added`, `date_modified`. Indexes: `customer_id`, `order_status_id`.
- `order_history`: `order_history_id` (PK), `order_id`, `order_status_id`, `notify` tinyint,
  `comment` text, `date_added`.
- `order_product`: `order_product_id` (PK), `order_id`, `product_id`, `name`, `model`,
  `quantity`, `price`, `total`, `tax`, `reward`.
- `order_option`: `order_option_id` (PK), `order_id`, `order_product_id`, `product_option_id`,
  `product_option_value_id`, `name`, `value` text, `type`.
- `order_total`: `order_total_id` (PK), `order_id`, `code`, `title`, `value`, `sort_order`.
- Key concept: `order_status_id = 0` means an incomplete/abandoned order (not shown in admin list
  by default: `getOrders`/`getTotalOrders` default filter is `order_status_id > 0`).

## Events

Default registrations from `install/opencart.sql` (`oc_event`). Note the order MAIL events fire on
`before` so the mail controller can read the OLD status:

- `activity_order_add` — `catalog/model/checkout/order/addOrderHistory/before` → `event/activity/addOrderHistory`
- `mail_order_add` — `catalog/model/checkout/order/addOrderHistory/before` → `mail/order`
- `mail_order_alert` — `catalog/model/checkout/order/addOrderHistory/before` → `mail/order/alert`
- `mail_voucher` — `catalog/model/checkout/order/addOrderHistory/after` → `extension/total/voucher/send`
- `statistics_order_history` — `catalog/model/checkout/order/addOrderHistory/after` → `event/statistics/addOrderHistory`

Any custom logic on order status change should hook `catalog/model/checkout/order/addOrderHistory/before`
or `/after` and expect args `[$order_id, $order_status_id, $comment, $notify, $override]`.

## API Links

- `api/order/add` → `ModelCheckoutOrder::addOrder()` then `addOrderHistory()`.
- `api/order/edit` → `editOrder()` then `addOrderHistory()`.
- `api/order/delete` → `deleteOrder()`.
- `api/order/info` → `getOrder()`.
- `api/order/history` → `addOrderHistory($order_id, $order_status_id, $comment, $notify, $override)`.
  This is the endpoint the admin order screen calls to change status (via `api_token`).

## Mail Links

- Order status changes fire `mail/order` (`ControllerMailOrder::index`) on the
  `addOrderHistory/before` event:
  - old status `0` and new status `> 0` → `add()` sends the HTML confirmation
    (`mail/order_add` view + `mail/order_add` language) to the customer.
  - old status `> 0`, new status `> 0`, and `notify` → `edit()` sends the plain-text
    status-update mail (`mail/order_edit`).
- `mail/order/alert` (`ControllerMailOrder::alert`) sends the admin new-order alert
  (`mail/order_alert`) only when `in_array('order', config_mail_alert)`, to `config_email`
  plus each address in `config_mail_alert_email`.
- Gift-voucher emails on completion go through `extension/total/voucher/send`.

## OCMOD Hotspots

- `catalog/model/checkout/order.php::addOrder` — inject extra columns/tables at order write.
- `catalog/model/checkout/order.php::addOrderHistory` — most contested method; stock, totals,
  affiliate, fraud, and status all live here. Prefer the `addOrderHistory/before|after` event
  over an OCMOD `<search>` inside the method body.
- `catalog/controller/checkout/confirm.php` — order payload assembly (custom fields, marketing,
  affiliate).
- `admin/controller/sale/order.php::getForm`/`info` — admin order info tabs and the `api_token`
  wiring.
- `admin/view/template/sale/order_info.twig` — admin order screen markup (history add block,
  totals, product table).
- `catalog/controller/mail/order.php::add`/`edit`/`alert` — email content and recipients.

## Theme Override Risks

- Themes commonly override `checkout/confirm.twig`, `common/success.twig`,
  `account/order_list.twig`, and `account/order_info.twig`.
- Custom themes and mail packs frequently replace `mail/order_add.twig` / `order_edit.twig` /
  `order_alert.twig`; verify the active theme's copies before editing the default theme.
- One-page-checkout modules replace the whole `checkout/*` chain and may bypass `checkout/confirm`.

## Common Extension Collision Zones

- One-page / custom checkout modules (rewrite confirm + payment flow).
- Payment gateways (each calls `addOrderHistory`; status-id config keys differ per gateway).
- Order status automation / order editor modules (also call `addOrderHistory`, may double-fire mail).
- Stock sync / ERP / CRM / invoice / export modules (read `order*` tables, hook status events).
- Anti-fraud extensions (`fraud/*`) can override the chosen status inside `addOrderHistory`.
- Mail/SMTP and branded-email modules (intercept the `mail/order` events or `Mail` library).

## Verification Notes

Verified against LiveStore `v3.0.4.4` on 2026-07-04. Confidence is split by how each item was checked.

### `[verified]` — read in full from source
- `catalog/model/checkout/order.php` (all methods, incl. `addOrder`, `editOrder`, `deleteOrder`,
  `getOrder*`, `addOrderHistory` signature + body).
- `admin/model/sale/order.php` (all methods; confirmed it has NO order-write methods).
- `catalog/controller/checkout/confirm.php`, `catalog/controller/checkout/success.php`.
- `catalog/controller/extension/payment/cod.php` (the `addOrderHistory` call).
- `catalog/controller/mail/order.php` (order mail — documented in the Mail pack).
- `system` mail library is covered in the Mail pack.
- `install/opencart.sql`: the `oc_order*` / `oc_order_status` / `oc_order_shipment` /
  `oc_shipping_courier` `CREATE TABLE` blocks (lines ~2121–2430) and the `oc_event` rows
  (lines ~1218–1257) were read directly.
- `admin/view/template/sale/order_info.twig:581` — the AJAX call to `api/order/history`.
- `catalog/controller/api/order.php::history()` (read in full); its `add`/`edit`/`delete` calls
  to the checkout model were confirmed by grep, method bodies not fully read.

### `[baseline]` — confirmed by targeted grep, not full read
- `public function` lists for `admin/controller/sale/order.php`, `catalog/controller/account/order.php`,
  `catalog/model/account/order.php`, and `catalog/controller/api/order.php` (method NAMES verified;
  most signatures/bodies not individually read).
- `api_token` / `store_url` assignment in `admin/controller/sale/order.php` (grep only).
- `config_mail_*` keys via grep of `admin/controller/setting/setting.php` (see Mail pack).

### `[exists]` — path existence-checked only (HTTP HEAD), CONTENT NOT read
- All storefront/admin/mail view templates and language files cited in `## Core Files`
  (e.g. `sale/order_*.twig`, `checkout/confirm.twig`, `common/success.twig`,
  `account/order_*.twig`, `mail/order_*.twig`, and the `*/language/*/…/order*.php` files).
  Their existence at `v3.0.4.4` is confirmed; their internal contents are NOT verified here.

### `[needs-verification]` — re-check on the live project
- Any theme override of the order/mail templates, extra `oc_order*` columns, and any
  OCMOD/event/extension that also touches `addOrderHistory`.
- On a non-LiveStore baseline: `oc_order_shipment` / `oc_shipping_courier` and `sale/order::shipping`.
