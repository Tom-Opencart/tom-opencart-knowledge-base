# Mail

## Scope

This domain covers how OpenCart sends transactional email: the `Mail` library and its
adaptors, the mail settings, and the event-driven `mail/*` controllers that build and send
customer notifications (order confirmation/status, registration, password reset, transaction,
affiliate, gift voucher) and admin alerts (new order, customer/affiliate approval, reward,
return, forgotten password).

It is tightly linked to Orders (order emails are the highest-traffic mail path).

## Baseline

- Version: OpenCart 3.0.x (LiveStore build)
- Source: https://github.com/19th19th/LiveStore tag `v3.0.4.4` (`upload/...`)
- Verified on: 2026-07-04

## Confidence Levels

Claims in this pack carry an explicit confidence tag. Do not treat lower tiers as facts.

- `[verified]` — the cited source file was read in full (or the exact cited block was read)
  at `v3.0.4.4`.
- `[baseline]` — observed directly in the baseline via a targeted grep of the real file
  (e.g. a specific `oc_event` row, a config-key grep) but not the whole file was read.
- `[exists]` — the path was existence-checked at `v3.0.4.4` (HTTP HEAD) but its CONTENT was
  not read.
- `[needs-verification]` — general OpenCart knowledge / inference, not confirmed against this
  baseline.

The authoritative confidence breakdown is in `## Verification Notes` at the end.

## Main Routes

Mail is not reached by a browser route; each mail controller is invoked by the event system
as the action of a registered event (see Events). Action routes:

### Customer-facing (fired in the catalog app)
- `mail/order` and `mail/order/alert`
- `mail/register` and `mail/register/alert`
- `mail/forgotten`
- `mail/transaction`
- `mail/affiliate` and `mail/affiliate/alert`
- `extension/total/voucher/send` (gift voucher email)

### Admin-facing (fired in the admin app)
- `mail/customer/approve`, `mail/customer/deny`
- `mail/affiliate/approve`, `mail/affiliate/deny`
- `mail/reward`
- `mail/return`
- `mail/transaction`
- `mail/forgotten`

## Core Files

### Mail library

- `system/library/mail.php` — `class Mail extends \stdClass`.
  - `__construct($adaptor = 'mail')` loads `Mail\{adaptor}` from `system/library/mail/`.
  - Methods: `setTo`, `setFrom`, `setSender`, `setReplyTo`, `setSubject`, `setText`, `setHtml`,
    `addAttachment`, `send`. Public property `$parameter`.
  - `send()` requires `to`, `from`, `sender`, `subject`, and one of `text`/`html`, then copies
    all object vars onto the adaptor and calls `$adaptor->send()`.
- Adaptors: `system/library/mail/mail.php` (`Mail\mail`, PHP `mail()`),
  `system/library/mail/smtp.php` (`Mail\smtp`). Chosen by `config_mail_engine`.

### Controllers (each is the action target of an event)

Only `catalog/controller/mail/order.php` was read in full `[verified]`. For the other
controllers the FILE PATH is existence-checked `[exists]`, and the method name is INFERRED from
the event `action` route (e.g. action `mail/register/alert` implies `register.php::alert`)
`[needs-verification]` — read the controller before relying on a specific method/signature.

Customer-facing (`catalog/controller/mail/`):
- `order.php` — `ControllerMailOrder`: `index(&$route,&$args)`, `add()`, `edit()`,
  `alert(&$route,&$args)`. `[verified]`
- `register.php` — actions `mail/register` (→ `index`) and `mail/register/alert` (→ `alert`). `[exists]` + inferred
- `forgotten.php` — action `mail/forgotten`. `[exists]`
- `transaction.php` — action `mail/transaction`. `[exists]`
- `affiliate.php` — actions `mail/affiliate` (→ `index`) and `mail/affiliate/alert` (→ `alert`). `[exists]` + inferred
- `extension/total/voucher.php` — action `extension/total/voucher/send` (→ `send`). `[exists]`

Admin-facing (`admin/controller/mail/`), each `[exists]` with the method inferred from the action:
- `reward.php` (`mail/reward`), `return.php` (`mail/return`), `forgotten.php` (`mail/forgotten`),
  `transaction.php` (`mail/transaction`).
- `customer.php` — actions `mail/customer/approve` and `mail/customer/deny`.
- `affiliate.php` — actions `mail/affiliate/approve` and `mail/affiliate/deny`.

Note: there is NO `admin/controller/mail/order.php` (existence-checked as absent at `v3.0.4.4`; HTTP HEAD returned 404). Order mail is catalog-only because the trigger (`ModelCheckoutOrder::addOrderHistory`) runs in the catalog app. `[exists]`

### Models

Mail has no model of its own. It reads:
- `setting/setting` — `getSettingValue('config_email', $store_id)` for the store-scoped From address.
- domain models for payload (e.g. order mail calls `model_checkout_order->getOrder/getOrderProducts/
  getOrderOptions/getOrderVouchers/getOrderTotals`, `tool/upload` for file options).

### Views

Customer-facing (`catalog/view/theme/default/template/mail/`): `order_add.twig` (HTML),
`order_edit.twig` (text), `order_alert.twig`, `register.twig`, `forgotten.twig`,
`transaction.twig`, `affiliate.twig`, `voucher.twig`.

Admin-facing (`admin/view/template/mail/`): `reward.twig`, `return.twig`, `forgotten.twig`,
`transaction.twig`, `customer_approve.twig`, `customer_deny.twig`, `affiliate_approve.twig`,
`affiliate_deny.twig`.

### Language Files

- Order mail (catalog): `catalog/language/en-gb/mail/order_add.php`, `mail/order_edit.php`,
  `mail/order_alert.php`. (There is no `mail/order.php` language file.)
- Other customer mail language files live under `catalog/language/{lang}/mail/`; admin mail
  language files under `admin/language/{lang}/mail/`.
- Customer emails load language by the ORDER/customer language, not the sender's admin language:
  order mail uses `new Language($order_info['language_code']); $language->load('mail/order_add');`.

## Data Flow

1. A business event runs a model method (e.g. `addCustomer`, `editCode`, `addTransaction`,
   `addOrderHistory`, `addReturnHistory`, approval methods).
2. The registered event fires the mapped `mail/*` action with the method args.
3. The mail controller assembles `$data`, loads the recipient-language strings, and resolves
   the From address (`getSettingValue('config_email', $store_id)`, falling back to `config_email`).
4. It builds `new Mail(config_mail_engine)`, sets `parameter` + `smtp_*` properties from config,
   sets To/From/Sender/Subject and `setHtml()` or `setText()`, then `send()`.
5. The adaptor (`Mail\mail` or `Mail\smtp`) performs the actual delivery.

## Database

Mail itself is config-driven, not table-driven.

- Settings are stored in `oc_setting` (`config_mail_*`, `config_email`, `config_name`,
  `config_logo`), managed by admin route `setting/setting` (Mail tab).
- Event registrations live in `oc_event` (`code`, `trigger`, `action`, `status`).
- Payload tables depend on the mail type (e.g. `order*`, `customer`, `customer_transaction`,
  `return*`).

### Relevant config keys (`[baseline]` via grep of `admin/controller/setting/setting.php`)

- `config_mail_engine` — `mail` or `smtp`.
- `config_mail_parameter` — extra params for the PHP `mail()` engine.
- `config_mail_smtp_hostname`, `config_mail_smtp_username`, `config_mail_smtp_password`
  (`html_entity_decode`d before use), `config_mail_smtp_port` (default `25`),
  `config_mail_smtp_timeout` (default `5`).
- `config_mail_alert` — array of alert event codes that are enabled (e.g. `order`); the order
  alert checks `in_array('order', config_mail_alert)`.
- `config_mail_alert_email` — comma-separated additional admin alert recipients.
- `config_email` — default From/sender and default alert To; `config_name`, `config_logo` used in bodies.

## Events

Default `oc_event` rows relevant to mail (from `install/opencart.sql`):

Customer-facing:
- `mail_customer_add` — `catalog/model/account/customer/addCustomer/after` → `mail/register`
- `mail_customer_alert` — `catalog/model/account/customer/addCustomer/after` → `mail/register/alert`
- `mail_forgotten` — `catalog/model/account/customer/editCode/after` → `mail/forgotten`
- `mail_transaction` — `catalog/model/account/customer/addTransaction/after` → `mail/transaction`
- `mail_affiliate_add` — `catalog/model/account/customer/addAffiliate/after` → `mail/affiliate`
- `mail_affiliate_alert` — `catalog/model/account/customer/addAffiliate/after` → `mail/affiliate/alert`
- `mail_order_add` — `catalog/model/checkout/order/addOrderHistory/before` → `mail/order`
- `mail_order_alert` — `catalog/model/checkout/order/addOrderHistory/before` → `mail/order/alert`
- `mail_voucher` — `catalog/model/checkout/order/addOrderHistory/after` → `extension/total/voucher/send`

Admin-facing:
- `admin_mail_customer_approve` — `admin/model/customer/customer_approval/approveCustomer/after` → `mail/customer/approve`
- `admin_mail_customer_deny` — `admin/model/customer/customer_approval/denyCustomer/after` → `mail/customer/deny`
- `admin_mail_affiliate_approve` — `admin/model/customer/customer_approval/approveAffiliate/after` → `mail/affiliate/approve`
- `admin_mail_affiliate_deny` — `admin/model/customer/customer_approval/denyAffiliate/after` → `mail/affiliate/deny`
- `admin_mail_reward` — `admin/model/customer/customer/addReward/after` → `mail/reward`
- `admin_mail_transaction` — `admin/model/customer/customer/addTransaction/after` → `mail/transaction`
- `admin_mail_return` — `admin/model/sale/return/addReturnHistory/after` → `mail/return`
- `admin_mail_forgotten` — `admin/model/user/user/editCode/after` → `mail/forgotten`

Key nuance: the two ORDER mail events use the `before` trigger (so `mail/order` can read the
OLD `order_status_id`); most other mail events use `after`.

## API Links

Indirect only. API actions that call the underlying domain models (e.g. `api/order/add`,
`api/order/history`) fire the same events and therefore trigger the same mail.

## Mail Links

This is the primary domain. See per-controller mapping above and the Orders pack for order mail.

## OCMOD Hotspots

- `system/library/mail.php` — the `Mail` class (rarely patched; prefer swapping the engine/adaptor).
- Individual `catalog/controller/mail/*.php` and `admin/controller/mail/*.php` — email subject,
  body data, and recipient logic.
- Mail templates under `catalog/view/theme/default/template/mail/` and `admin/view/template/mail/`.
- `admin/controller/setting/setting.php` + Mail-tab template — mail settings UI.
- Prefer hooking the relevant `{model method}/before|after` event over patching a mail controller,
  so multiple mail modifications can coexist.

## Theme Override Risks

- Customer mail templates can be overridden by the active theme; a themed copy of
  `template/mail/*.twig` shadows the default theme.
- Branded HTML-email packs replace `order_add.twig`/`register.twig` etc.; verify which theme
  actually renders before editing default.

## Common Extension Collision Zones

- SMTP / mailer modules (replace or wrap the `Mail` library or override `config_mail_engine`).
- Branded email-template packs (swap `mail/*` templates).
- Order-status automation modules (re-fire or suppress order mail via the `addOrderHistory` events).
- Marketing / CRM / newsletter connectors (add their own `mail/*` events or intercept sends).
- Transaction/queue mailers that defer `Mail::send()`.

## Verification Notes

Verified against LiveStore `v3.0.4.4` on 2026-07-04. Confidence is split by how each item was checked.

### `[verified]` — read in full from source
- `system/library/mail.php` (the `Mail` class API + adaptor loading).
- `catalog/controller/mail/order.php` (`index`/`add`/`edit`/`alert`, engine/SMTP wiring,
  recipient-language loading, alert gating on `config_mail_alert`).
- `install/opencart.sql` `oc_event` rows for every mail mapping listed under `## Events`
  (read directly, lines ~1218–1257).

### `[baseline]` — confirmed by targeted grep, not full read
- `config_mail_*` keys and defaults (`config_mail_smtp_port = 25`, `config_mail_smtp_timeout = 5`)
  via grep of `admin/controller/setting/setting.php`.

### `[exists]` — path existence-checked only (HTTP HEAD), CONTENT NOT read
- The customer-facing mail controllers `register.php`, `forgotten.php`, `transaction.php`,
  `affiliate.php`, and `extension/total/voucher.php`.
- The admin-facing mail controllers `reward.php`, `return.php`, `customer.php`, `affiliate.php`,
  `forgotten.php`, `transaction.php`.
- All cited mail templates and language files (`catalog/view/theme/default/template/mail/*.twig`,
  `admin/view/template/mail/*.twig`, `*/language/*/mail/*.php`).
- Confirmed by HEAD that `admin/view/template/mail/customer.twig` and `.../affiliate.twig` do NOT
  exist, whereas `customer_approve/customer_deny/affiliate_approve/affiliate_deny.twig` DO — hence
  those controllers use the `*_approve`/`*_deny` templates.

### `[needs-verification]` — re-check before relying
- Exact bodies, `$data` keys, subjects, and recipient logic of every mail controller EXCEPT
  `mail/order` (only order mail was line-read). Re-read the specific controller before changing it.
- The exact template each non-order controller renders (only the order templates were tied to code).
- On the live project: active-theme mail templates, real SMTP settings, and any module that
  overrides the `Mail` library or intercepts the mail events.
