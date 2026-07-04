# Customer Group Transition Module

## Scope

This module family automatically moves customers between customer groups based on accumulated order
totals and selected order statuses. The specific implementation documented here is a single-instance
 OpenCart 3 admin module with:

- one global module status
- many transition rules inside the module page
- storefront runtime through events
- OCMOD used only for bulk manual group change in the native customer list

It does not store rule settings inside `Settings` or inside the standard `customer_group` form.

## Baseline

- Version: OpenCart 3.0.x / LiveStore build
- Source: https://github.com/19th19th/LiveStore tag `v3.0.4.4` for native customer-list chain,
  plus a verified custom module package reviewed on 2026-07-05
- Verified on: 2026-07-05

## Confidence Levels

- `[verified]` — custom module file was read directly in full during the task.
- `[baseline]` — confirmed against the real LiveStore/OpenCart chain by targeted source read or grep.
- `[needs-verification]` — live-project-specific collisions or integrations not confirmed in the
  baseline store.

## Main Routes

### Admin module

- `extension/module/customer_group_transition` — module page with one global status and rule CRUD.
  `[verified]`
- Additional admin JSON handlers inside the same controller:
  - `save`
  - `getRule`
  - `saveRule`
  - `deleteRule`

### Storefront runtime actions

- `extension/module/customer_group_transition/onCustomerLogin` via event. `[verified]`
- `extension/module/customer_group_transition/onAccountAccess` via event. `[verified]`
- `extension/module/customer_group_transition/onOrderHistoryAdd` via event. `[verified]`

### Native admin customer list

- `customer/customer/changeGroup` — injected by OCMOD into the native admin customer controller for
  bulk manual group change. `[verified]`

## Core Files

### Module Controllers `[verified]`

- `upload/admin/controller/extension/module/customer_group_transition.php`
  - module page
  - global module status save
  - rule CRUD
  - install/uninstall
  - event registration and removal
- `upload/catalog/controller/extension/module/customer_group_transition.php`
  - runtime event handlers
  - multi-step transition loop with safety cap
  - active-session refresh for the current logged-in customer after group change

### Module Models `[verified]`

- `upload/admin/model/extension/module/customer_group_transition.php`
  - schema creation/removal
  - rule CRUD
  - linked order-status persistence
- `upload/catalog/model/extension/module/customer_group_transition.php`
  - rule lookup by source group
  - downgrade-capable lookup by target group
  - order-total calculation
  - direct customer-group update

### Module View `[verified]`

- `upload/admin/view/template/extension/module/customer_group_transition.twig`
  - single-instance admin UI
  - global module status field
  - inline rule list and rule edit form

### Module Languages `[verified]`

- `upload/admin/language/en-gb/extension/module/customer_group_transition.php`
- `upload/admin/language/ru-ru/extension/module/customer_group_transition.php`

### OCMOD File `[verified]`

- `install.xml`
  - injects native customer-list bulk-change UI
  - adds `changeGroup()` to `admin/controller/customer/customer.php`
  - adds `changeCustomerGroup()` to `admin/model/customer/customer.php`
  - adds language strings to `admin/language/en-gb/customer/customer.php` and `ru-ru/...`

## Data Flow

### Automatic transition `[verified]`

1. A storefront event fires on login, account access, or order history update.
2. Runtime loads the customer's current group from DB.
3. Runtime looks for an enabled rule whose `source_customer_group_id` matches the current group.
4. Runtime loads rule order statuses and calculates the customer's total over the configured period.
5. If the total reaches `min_total`, runtime updates `customer.customer_group_id` directly.
6. If the moved customer is the current logged-in session customer, runtime refreshes the active
   `Cart\Customer` object with `login(..., $override = true)`.
7. Runtime repeats the loop up to 5 times to support multi-step transitions in one request.

### Downgrade `[verified]`

1. If no upgrade rule matched the current group, runtime checks whether the customer is currently in
   a target group of an enabled rule with `allow_downgrade = 1`.
2. Runtime recalculates the same period/status-limited total.
3. If the total is now below `min_total`, runtime moves the customer back to
   `source_customer_group_id`.

### Manual bulk change from native customer list `[verified]`

1. OCMOD adds a button and modal to `admin/view/template/customer/customer_list.twig`.
2. Selected customers and target group are posted to `customer/customer/changeGroup`.
3. The injected controller method validates permission, selected rows, and target group existence.
4. The injected model method updates `customer.customer_group_id` for the selected ids.

## Database

### Tables `[verified]`

- `oc_customer_group_transition`
  - `rule_id`
  - `status`
  - `source_customer_group_id`
  - `target_customer_group_id`
  - `min_total`
  - `period_days`
  - `allow_downgrade`
  - `date_added`
  - `date_modified`
  - unique key on `source_customer_group_id`
- `oc_customer_group_transition_status`
  - `rule_id`
  - `order_status_id`

### Important Behavioral Notes

- The module is intentionally single-instance. Different customer groups get different behavior
  through multiple rules, not through multiple module instances. `[verified]`
- The order-period threshold is computed on the PHP side and then passed into SQL as a datetime
  string; avoid depending on MySQL `NOW()` when PHP/OpenCart timezone handling matters. `[verified]`

## Events

### Module-owned events `[verified]`

- `catalog/controller/account/login/after`
- `catalog/controller/account/account/before`
- `catalog/model/checkout/order/addOrderHistory/after`

### Integration caution `[baseline]`

- Direct SQL updates to `customer.customer_group_id` do not automatically fire the standard
  customer edit model events. External CRM/ERP/newsletter integrations that only watch
  `editCustomer/after` will not see these group transitions unless a separate bridge is added.

## Related Core Domains

- `opencart-core/customers.md`
- `opencart-core/orders.md`
- `opencart-core/products.md`
- `opencart-core/mail.md` (only if later versions add notifications)

## OCMOD Hotspots

- `admin/controller/customer/customer.php` — new JSON action `changeGroup`.
- `admin/model/customer/customer.php` — new bulk mutation helper `changeCustomerGroup`.
- `admin/language/*/customer/customer.php` — UI labels and errors for the new customer-list action.
- `admin/view/template/customer/customer_list.twig` — button, modal, and AJAX flow.

Do not spread this module's rule-editing UI into `customer_group.php` or `setting/setting.php`
unless the feature scope explicitly changes. `[verified]`

## Common Collision Zones

- B2B / VIP pricing modules that also move customers between groups.
- Discount logic that reads the current customer group from `Cart\Customer` inside the same request.
- CRM/ERP bridges that rely on standard customer edit events instead of raw table changes.
- Customer-list UI modifiers that also patch `customer/customer_list.twig`.
- Order-status automation modules that add their own `addOrderHistory/after` behavior.

## Verification Notes

### `[verified]`

- All custom module files under `upload/...`
- `install.xml`
- runtime session-refresh logic via `Cart\Customer::login(..., $override = true)`
- multi-step transition safety loop
- PHP-computed date threshold for period filtering

### `[baseline]`

- Native admin customer-list chain in LiveStore/OpenCart:
  `admin/controller/customer/customer.php`,
  `admin/model/customer/customer.php`,
  `admin/language/*/customer/customer.php`,
  `admin/view/template/customer/customer_list.twig`
- Practical collision and integration risks with other modules/events

### `[needs-verification]`

- Live-project-specific extensions that override customer list, customer session behavior, pricing,
  or CRM synchronization.
