# Customers

## Scope

This domain covers storefront customer registration, login/session, profile editing, password
change, password-reset (forgotten), addresses, customer groups, the approval workflow, affiliate
sign-up, reward/transaction balances, login throttling, and admin customer management.

It excludes order data (see Orders) and the mail bodies themselves (see Mail); only the customer
side triggers into those domains are covered here.

## Baseline

- Version: OpenCart 3.0.x (LiveStore build)
- Source: https://github.com/19th19th/LiveStore tag `v3.0.4.4` (`upload/...`)
- Verified on: 2026-07-04

## Confidence Levels

Claims carry an explicit confidence tag. Do not treat lower tiers as facts.

- `[verified]` — the cited source file was read in full (or the exact cited block was read) at `v3.0.4.4`.
- `[baseline]` — observed via a targeted grep of the real file (e.g. a `public function` list,
  an `oc_event` row) but the whole file was not read.
- `[exists]` — path existence-checked at `v3.0.4.4` (HTTP HEAD); CONTENT not read.
- `[needs-verification]` — general OpenCart knowledge / inference, not confirmed against this baseline.

The authoritative confidence breakdown is in `## Verification Notes` at the end.

## Main Routes

### Storefront (`account/*`) `[exists]` (controllers existence-checked)
- `account/register`, `account/login`, `account/logout`
- `account/edit` (profile), `account/password` (change password)
- `account/forgotten` (password reset via email code)
- `account/address` (address book), `account/account` (dashboard)

### Admin
- `customer/customer` — controller methods `index`, `add`, `edit`, `delete`, `unlock`, `login`,
  `history`, `addHistory`, `transaction`, `addTransaction`, `reward`, `addReward`, `ip`,
  `autocomplete`, `customfield`, `address`. `[baseline]`
- `customer/customer_approval` — approval queue (approve/deny customers & affiliates). `[exists]`
- `customer/customer_group` — customer groups. `[needs-verification]`

### API `[exists]`
- `api/customer` and `api/login` controllers exist. Detailed route/method behavior belongs in
  the API pack.

## Core Files

### Controllers

- Storefront `catalog/controller/account/`: `register.php`, `login.php`, `logout.php`,
  `edit.php`, `password.php`, `forgotten.php`, `address.php`, `account.php`. `[exists]`
- `admin/controller/customer/customer.php` — `ControllerCustomerCustomer` (routes above). `[baseline]`
- `admin/controller/customer/customer_approval.php`, `.../customer_group.php`. `[exists]`

### Models

- `catalog/model/account/customer.php` — `ModelAccountCustomer` (read in full `[verified]`):
  - `addCustomer($data)` — inserts `customer` with `salt = token(9)` and
    `password = sha1($salt . sha1($salt . sha1($password)))`; `status` = NOT the group's `approval`
    flag; if the group requires approval, inserts `customer_approval` (`type = 'customer'`).
  - `editCustomer`, `editPassword($email, $password)` (re-salts, clears `code`),
    `editAddressId`, `editCode($email, $code)` (sets reset code), `editNewsletter`.
  - `getCustomer`, `getCustomerByEmail`, `getCustomerByCode`, `getCustomerByToken` (one-time
    auto-login token; clears `token` on read), `getTotalCustomersByEmail`.
  - `addTransaction`, `deleteTransactionByOrderId`, `getTransactionTotal`,
    `getTotalTransactionsByOrderId`, `getRewardTotal`, `getIps`.
  - Login throttling: `addLoginAttempt`, `getLoginAttempts`, `deleteLoginAttempts` (`customer_login`).
  - Affiliate: `addAffiliate` (writes `customer_affiliate`, and `customer_approval` type
    `'affiliate'` if `config_affiliate_approval`), `editAffiliate`, `getAffiliate`,
    `getAffiliateByTracking`.
- `catalog/model/account/address.php` — `ModelAccountAddress` (grep `[baseline]`): `addAddress`,
  `editAddress`, `deleteAddress`, `getAddress`, `getAddresses`, `getTotalAddresses`.
- `catalog/model/account/customer_group.php` — `getCustomerGroup(s)`. `[exists]`
- `admin/model/customer/customer.php` — `ModelCustomerCustomer` (grep `[baseline]`): `addCustomer`,
  `editCustomer`, `editToken`, `deleteCustomer`, `getCustomer(s)`, `getCustomerByEmail`,
  `getAddress(es)`, `getTotalCustomers*`, `getAffiliate*`, `addHistory`/`getHistories`,
  `addTransaction`/`deleteTransactionByOrderId`/`getTransactions`/`getTransactionTotal`,
  `addReward`/`deleteReward`/`getRewards`/`getRewardTotal`, `getIps`, `getTotalLoginAttempts`,
  `deleteLoginAttempts`.
- `admin/model/customer/customer_approval.php` — `ModelCustomerCustomerApproval` (read in full
  `[verified]`): `getCustomerApprovals`, `getCustomerApproval`, `getTotalCustomerApprovals`,
  `approveCustomer` (sets `customer.status = 1`, deletes the `customer` approval row),
  `denyCustomer`, `approveAffiliate` (sets `customer_affiliate.status = 1`), `denyAffiliate`.

### Session library

- `system/library/cart/customer.php` — `Cart\Customer` (read in full `[verified]`):
  - `__construct($registry)` loads the logged-in customer from `session.data['customer_id']`
    where `status = 1`, refreshes `language_id`/`ip`, records `customer_ip`, logs out if not found.
  - `login($email, $password, $override = false)` — matches salted `SHA1(CONCAT(salt, SHA1(...)))`
    OR legacy `md5($password)`; sets the session on success.
  - `logout`, `isLogged` (returns `customer_id`), `getId`, `getFirstName`, `getLastName`,
    `getGroupId`, `getEmail`, `getTelephone`, `getNewsletter`, `getAddressId`,
    `getBalance` (`customer_transaction` sum), `getRewardPoints` (`customer_reward` sum).

### Views / Language `[exists]`

- Storefront views: `account/register.twig`, `login.twig`, `edit.twig`, `password.twig`,
  `forgotten.twig`, `address_form.twig` (+ others under `account/`).
- Storefront language: `catalog/language/en-gb/account/register.php`, `login.php`, `forgotten.php`, …
- Admin views: `admin/view/template/customer/customer_list.twig`, `customer_form.twig`,
  `customer_approval.twig`.
- Admin language: `admin/language/en-gb/customer/customer.php`.

## Data Flow

Registration (`account/register`):
1. Controller validates and calls `ModelAccountCustomer::addCustomer()`.
2. Password is salted+SHA1 hashed; `status` = 1 unless the customer group's `approval` flag is set.
3. If approval is required, a `customer_approval` row (`type='customer'`) is created and the
   customer stays `status = 0`.
4. The `addCustomer/after` event fires `mail/register` (welcome) and `mail/register/alert` (admin). `[verified]` (event rows)

Registration group resolution `[baseline]`:
1. The final `customer` insert happens in `catalog/model/account/customer.php::addCustomer($data)`.
2. But the effective `customer_group_id` for a new storefront customer is not owned by the model alone.
3. In the stock baseline, `catalog/controller/account/register.php` and `catalog/controller/checkout/register.php`
   both resolve `customer_group_id` before the model call, and both reuse that group for group-scoped
   custom-field validation.
4. In the same baseline, `ModelAccountCustomer::addCustomer()` independently resolves its own fallback
   `customer_group_id` before writing the row and before checking the target group's `approval` flag.
5. Practical implication: a custom rule like "all new customers must enter group X" is a multi-layer change.
   Patching only `catalog/model/account/customer.php` is not sufficient; verify the full registration chain:
   `catalog/controller/account/register.php`, `catalog/controller/checkout/register.php`, and
   `catalog/model/account/customer.php`.

Login (`account/login`): `Cart\Customer::login()` verifies the hash (or legacy md5) and stores
`customer_id` in the session; `__construct` re-hydrates on each request and enforces `status = 1`.

Password reset (`account/forgotten`): `editCode()` stores a reset `code`; the `editCode/after`
event fires `mail/forgotten`. `getCustomerByCode()` validates the code; `editPassword()` re-salts
and clears the code.

Approval (admin `customer/customer_approval`): `approveCustomer`/`approveAffiliate` flip `status`
and remove the approval row; the `*/after` events fire `mail/customer/approve|deny` and
`mail/affiliate/approve|deny`. `[verified]` (event rows)

## Database

### Tables (verified `CREATE TABLE` read in `install/opencart.sql`) `[verified]`

`customer` (821), `customer_activity` (855), `customer_affiliate` (872), `customer_approval` (900),
`customer_group` (915), `customer_group_description` (936), `customer_history` (958),
`customer_login` (974), `customer_ip` (993), `customer_online` (1009), `customer_reward` (1026),
`customer_transaction` (1044), `customer_search` (1062), `customer_wishlist` (1085), `address` (14).

No LiveStore-specific columns were observed in the customer tables at `v3.0.4.4` (unlike the
product/order tables); the schema matches standard OpenCart 3.0.x. `[verified]`

### Important columns `[verified]`

- `customer`: `customer_id` (PK), `customer_group_id`, `store_id`, `language_id`, `firstname`,
  `lastname`, `email`, `telephone`, `fax`, `password` varchar(40) (SHA1), `salt` varchar(9),
  `cart` text, `wishlist` text, `newsletter`, `address_id`, `custom_field` text, `ip`, `status`,
  `safe` (fraud whitelist flag — read by order `addOrderHistory`), `token` text (one-time
  auto-login), `code` varchar(40) (password-reset code), `date_added`. Index: `customer_group_id`.
- `customer_approval`: `customer_approval_id` (PK), `customer_id`, `type` varchar(9)
  (`'customer'` | `'affiliate'`), `date_added`.
- `customer_group`: `customer_group_id` (PK), `approval` (0/1 — forces approval queue), `sort_order`.

## Events

Default `oc_event` rows in this domain (verified in `install/opencart.sql`, lines ~1218–1250) `[verified]`:

Customer (catalog model `catalog/model/account/customer`):
- `addCustomer/after` → `event/activity/addCustomer`, `mail/register`, `mail/register/alert`
- `editCustomer/after` → `event/activity/editCustomer`
- `editPassword/after` → `event/activity/editPassword`
- `editCode/after` → `event/activity/forgotten`, `mail/forgotten`
- `addTransaction/after` → `event/activity/addTransaction`, `mail/transaction`
- `deleteLoginAttempts/after` → `event/activity/login`
- `addAffiliate/after` → `event/activity/addAffiliate`, `mail/affiliate`, `mail/affiliate/alert`
- `editAffiliate/after` → `event/activity/editAffiliate`

Address (`catalog/model/account/address`):
- `addAddress/after` → `event/activity/addAddress`; `editAddress/after`; `deleteAddress/after`.

Admin:
- `admin/model/customer/customer_approval/approveCustomer/after` → `mail/customer/approve`
  (+ `denyCustomer` → `mail/customer/deny`, `approveAffiliate`/`denyAffiliate` → `mail/affiliate/*`)
- `admin/model/customer/customer/addReward/after` → `mail/reward`
- `admin/model/customer/customer/addTransaction/after` → `mail/transaction`

## API Links

- `catalog/controller/api/customer.php` and `api/login.php` exist. `[exists]`
  Their exact role in admin/customer/order flows is documented in the API pack.

## Mail Links

- `mail/register` + `mail/register/alert` (registration), `mail/forgotten` (password reset),
  `mail/transaction`, `mail/affiliate` + `mail/affiliate/alert`, and the admin approve/deny +
  reward mails. All are driven by the events above — see the Mail pack.

## OCMOD Hotspots

- `catalog/model/account/customer.php::addCustomer` — registration write (custom fields, extra
  columns, hashing). Prefer the `addCustomer/before|after` event.
- `catalog/controller/account/register.php` / `edit.php` — validation and custom-field handling.
- New-customer group customization is a multi-layer hotspot: do not patch only
  `catalog/model/account/customer.php::addCustomer` and assume the behavior is complete. The stock
  `account/register` and `checkout/register` controllers also prepare and validate `customer_group_id`
  before the model write.
- `system/library/cart/customer.php::login` — auth (social-login / SSO modules hook here).
- `admin/model/customer/customer_approval.php` — approval side effects.
- Storefront `account/*.twig` — form markup.
- Customer-list bulk group change is a native admin-chain hotspot, not just a Twig insertion:
  `admin/controller/customer/customer.php` + `admin/model/customer/customer.php` +
  `admin/language/*/customer/customer.php` + `admin/view/template/customer/customer_list.twig`.
  A list-only Twig patch is insufficient if the action needs new JSON handling, messages, or
  data mutation. `[baseline]`
- If a module changes `customer.customer_group_id` directly in SQL during storefront execution,
  the logged-in `Cart\Customer` object is not automatically refreshed inside the same request.
  If the current session's customer was moved, re-hydrate the session explicitly (for example by
  calling `Cart\Customer::login(..., $override = true)`) before expecting `getGroupId()` and
  group-scoped pricing to reflect the new value. `[verified]` for `Cart\Customer::login` override
  support / `[baseline]` for practical module-side use.

## Theme Override Risks

- Themes commonly override `register.twig`, `login.twig`, `edit.twig`, `address_form.twig`, and
  custom-field rendering. One-page-checkout / one-page-account modules replace the whole flow.

## Common Extension Collision Zones

- One-page checkout/account modules (rewrite register/login/address).
- Social login / SSO modules (hook `Cart\Customer::login` or add their own auth).
- B2B / customer-group modules (touch `customer_group`, `approval`, custom pricing).
- Approval/verification add-ons (SMS/email confirm) interacting with `customer_approval` + `status`.
- Loyalty / reward / wallet modules (`customer_reward`, `customer_transaction`).
- Customer-group transition / B2B pricing / VIP modules that:
  - update `customer.customer_group_id` directly
  - read the current group from `Cart\Customer`
  - inject bulk-change UI into `customer/customer_list.twig`
  - rely on `catalog/model/checkout/order/addOrderHistory/after` for runtime transitions
  These modules commonly collide at the customer list, session refresh boundary, and group-scoped
  pricing/discount logic. `[baseline]`

## Verification Notes

Verified against LiveStore `v3.0.4.4` on 2026-07-04. Confidence split by how each item was checked.

### `[verified]` — read in full from source
- `catalog/model/account/customer.php` (all methods, incl. hashing, approval, login-throttle, affiliate).
- `system/library/cart/customer.php` (`Cart\Customer` session/auth API).
- `admin/model/customer/customer_approval.php` (approve/deny logic).
- `install/opencart.sql`: the `oc_customer*` / `oc_customer_approval` / `oc_customer_group` /
  `oc_address` `CREATE TABLE` blocks (lines ~14, 821–1085) and the customer/address/affiliate/
  approval `oc_event` rows (lines ~1218–1250).

### `[baseline]` — confirmed by targeted grep, not full read
- `catalog/controller/account/register.php` and `catalog/controller/checkout/register.php`
  `customer_group_id` handling was confirmed by targeted grep: both stock controllers resolve a
  group before the model call and reuse it for custom-field validation; `catalog/model/account/customer.php::addCustomer()`
  separately resolves its own fallback `customer_group_id` before insert / approval handling.
- `public function` lists for `admin/controller/customer/customer.php`,
  `admin/model/customer/customer.php`, and `catalog/model/account/address.php`.

### `[exists]` — path existence-checked only (HTTP HEAD), CONTENT not read
- All `catalog/controller/account/*` controllers, `catalog/model/account/customer_group.php`,
  `admin/controller/customer/customer_approval.php`.
- `catalog/controller/api/customer.php`, `catalog/controller/api/login.php`.
- All cited storefront/admin views and language files.

### `[needs-verification]` — re-check before relying
- Exact bodies/validation of the storefront account controllers and the admin customer controller methods.
- `customer/customer_group` controller/model routes and methods.
- On the live project: active-theme account templates, extra `customer*` columns, and any
  social-login/approval/B2B module altering registration, login, or approval.
