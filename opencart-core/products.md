# Products

## Scope

This domain covers storefront product rendering (product page, category/search/manufacturer/
special listings), the storefront catalog product model (pricing, discount, special, reward,
options, attributes, images, related), admin product CRUD, the product table set, and the
common theme/extension override zones.

It excludes the cart/checkout usage of products (see Orders) and category tree logic beyond
the product↔category link.

## Baseline

- Version: OpenCart 3.0.x (LiveStore build)
- Source: https://github.com/19th19th/LiveStore tag `v3.0.4.4` (`upload/...`)
- Verified on: 2026-07-04

## Confidence Levels

Claims carry an explicit confidence tag. Do not treat lower tiers as facts.

- `[verified]` — the cited source file was read in full (or the exact cited block was read) at `v3.0.4.4`.
- `[baseline]` — observed directly via a targeted grep of the real file (e.g. a `public function`
  list, an `oc_event` row) but the whole file was not read.
- `[exists]` — path existence-checked at `v3.0.4.4` (HTTP HEAD); CONTENT not read.
- `[needs-verification]` — general OpenCart knowledge / inference, not confirmed against this baseline.

The authoritative confidence breakdown is in `## Verification Notes` at the end.

## Main Routes

### Storefront
- `product/product` — product page. Controller methods `index`, `review`, `write`,
  `getRecurringDescription` `[baseline]`.
- `product/category` — category listing.
- `product/search` — search results.
- `product/special` — specials listing.
- `product/manufacturer` — brand list (`index`) and per-brand products (`info`) `[needs-verification]`.

### Admin
- `catalog/product` — controller methods `index`, `add`, `edit`, `delete`, `copy`, `enable`,
  `disable`, `autocomplete` `[baseline]`.

### API
- None. OpenCart 3.0.x core ships no `catalog/controller/api/product.php`
  (existence-checked as absent at `v3.0.4.4`; HTTP HEAD returned 404); products are not
  exposed through the storefront API by default. `[exists]`

## Core Files

### Controllers

- `catalog/controller/product/product.php` — `ControllerProductProduct`. `index()` renders the
  `product/product` view; it loads `catalog/category`, `catalog/manufacturer`, `catalog/product`,
  `catalog/review`, `catalog/information`, `tool/image`, and calls
  `model_catalog_product->getProduct/getProductImages/getProductDiscounts/getProductOptions/
  getProductAttributes/getProductRelated` and `updateViewed()`. `review()`/`write()` handle the
  review AJAX (`product/review` view). `[baseline]` (grep of model/view/language calls)
- `catalog/controller/product/category.php`, `search.php`, `manufacturer.php`, `special.php`
  — storefront listing controllers. `[exists]`
- `admin/controller/catalog/product.php` — `ControllerCatalogProduct` (routes above). `[baseline]`

### Models

- `catalog/model/catalog/product.php` — `ModelCatalogProduct` (storefront read side, read in
  full `[verified]`): `updateViewed`, `getProduct`, `getProducts($data)`, `getProductSpecials`,
  `getLatestProducts($limit)`, `getPopularProducts($limit)`, `getBestSellerProducts($limit)`
  (these three are cache-backed: `product.latest|popular|bestseller.*`), `getProductAttributes`,
  `getProductOptions`, `getProductDiscounts`, `getProductImages`, `getProductRelated`,
  `getProductLayoutId`, `getCategories`, `getTotalProducts($data)`, `getProfile`, `getProfiles`,
  `getTotalProductSpecials`.
- `admin/model/catalog/product.php` — `ModelCatalogProduct` (admin write/read side). `addProduct`
  and `editProduct` read in full `[verified]`; the remaining methods verified by grep `[baseline]`:
  `editProductStatus`, `copyProduct`, `deleteProduct`, `getProduct`, `getProducts`,
  `getProductsByCategoryId`, `getProductDescriptions`, `getProductCategories`,
  `getProductMainCategoryId`, `getProductFilters`, `getProductAttributes`, `getProductOptions`,
  `getProductOptionValue`, `getProductImages`, `getProductDiscounts`, `getProductSpecials`,
  `getProductRewards`, `getProductDownloads`, `getProductStores`, `getProductSeoUrls`,
  `getProductLayouts`, `getProductRelated`, `getArticleRelated`, `getRecurrings`,
  `getTotalProducts`, and the `getTotalProductsBy*` family.
- `catalog/model/catalog/manufacturer.php` — brand model. `[exists]`

### Views

- Storefront `[exists]`: `catalog/view/theme/default/template/product/product.twig`,
  `category.twig`, `search.twig`, `special.twig`, `review.twig`, `manufacturer_list.twig`,
  `manufacturer_info.twig`. Note: `product/manufacturer.twig` does NOT exist
  (existence-checked as absent at `v3.0.4.4`; HTTP HEAD returned 404) — the manufacturer
  controller uses the `_list`/`_info` templates. `[exists]`
- Admin `[exists]`: `admin/view/template/catalog/product_list.twig`, `product_form.twig`.

### Language Files `[exists]`

- Storefront: `catalog/language/en-gb/product/product.php`, `category.php`, `search.php`,
  `manufacturer.php`, `special.php`.
- Admin: `admin/language/en-gb/catalog/product.php`.

## Data Flow

Storefront product page (`product/product::index`):

1. `getProduct($product_id)` returns the product; `price` = the qty-1 `product_discount` for the
   current customer group if present, else base `product.price`; `special` is a separate override
   resolved by date window + priority. `[verified]`
2. Images, options, discounts (qty > 1), attributes, and related products are resolved via the
   matching `getProduct*` methods. `[verified]`
3. `updateViewed()` increments `product.viewed`. `[verified]`
4. Layout is resolved by `getProductLayoutId($product_id)` (`product_to_layout`, per store). `[verified]`
5. The controller renders `product/product`; the view-event `catalog/view/product/product/before|after`
   fires around rendering. `[baseline]` (event rows) / `[needs-verification]` (exact args)

Listing/model reuse: `getProducts`, `getProductSpecials`, `getLatest/Popular/BestSeller` all loop
their id results back through `getProduct()`, so every card gets the same price/special logic. `[verified]`

## Database

### Core tables (verified `CREATE TABLE` read in `install/opencart.sql`, MyISAM) `[verified]`

Line numbers are for that file at `v3.0.4.4`:

- `product` (2439), `product_description` (2605), `product_to_category` (2939),
  `product_to_store` (3021), `product_to_download` (2994), `product_to_layout` (3007).
- `product_attribute` (2574), `product_option` (2750), `product_option_value` (2786),
  `product_image` (2712), `product_filter` (2699).
- `product_discount` (2670), `product_special` (2911), `product_reward` (2870),
  `product_recurring` (2833), `product_related` (2847).
- SEO keywords for products live in `seo_url` (written by admin `addProduct`). `[verified]`

### LiveStore build extras (present in this baseline, NOT vanilla OpenCart core) `[verified]`

- `product.noindex` tinyint(1) DEFAULT '1' (line 2471).
- `product_description.meta_h1` varchar(255) (line 2614).
- `product_to_category.main_category` tinyint(1) DEFAULT '0' (line 2942).
- `product_related_article` (line 4797), `product_related_wb` (line 4935),
  `product_related_mn` (line 4959) — additional relation tables; `product_related_article` is
  written by admin `addProduct` via `$data['product_related_article']`. `[verified]`
- `config_seo_pro` / `seopro` cache are cleared by admin `addProduct`. `[verified]`
  Re-confirm all of the above on any non-LiveStore baseline.

### Important columns `[verified]`

- `product`: `product_id` (PK), `model`, `sku`, `upc`, `ean`, `jan`, `isbn`, `mpn`, `location`,
  `quantity`, `stock_status_id`, `image`, `manufacturer_id`, `shipping`, `price`, `points`,
  `tax_class_id`, `date_available`, `weight`+`weight_class_id`, `length`/`width`/`height`+
  `length_class_id`, `subtract`, `minimum`, `sort_order`, `status`, `viewed`,
  `date_added`/`date_modified`, `noindex` (LS). Index: `manufacturer_id`.
- `product_description`: PK (`product_id`,`language_id`); `name`, `description`, `tag`,
  `meta_title`, `meta_description`, `meta_keyword`, `meta_h1` (LS). Index: `name`.
- `product_to_category`: PK (`product_id`,`category_id`), `main_category` (LS).

## Events

Default `oc_event` rows touching products are all `advertise_google` `[verified]`:

- `admin/model/catalog/product/addProduct/after` → `extension/advertise/google/addProduct`
- `admin/model/catalog/product/copyProduct/after` → `extension/advertise/google/copyProduct`
- `admin/model/catalog/product/deleteProduct/after` → `extension/advertise/google/deleteProduct`
- `catalog/view/product/product/after` → `extension/advertise/google/google_dynamic_remarketing_product`
- `catalog/view/product/search/after` → `extension/advertise/google/google_dynamic_remarketing_searchresults`
- `catalog/view/product/category/after` → `extension/advertise/google/google_dynamic_remarketing_category`

Generic hook points for custom code (standard OpenCart behavior, `[needs-verification]` for exact
args): `admin/model/catalog/product/{addProduct,editProduct,copyProduct,deleteProduct}/before|after`
and view events `catalog/view/product/{product,category,search}/{before,after}`.

## API Links

None by default: no `catalog/controller/api/product.php` (existence-checked as absent at
`v3.0.4.4`; HTTP HEAD returned 404). `[exists]` Product data reaches the cart/order
via the checkout flow, not a product API.

## Mail Links

Indirect only. Core sends no product email; stock/price alerts require extensions. `[needs-verification]`

## OCMOD Hotspots

- `catalog/model/catalog/product.php::getProduct` — the price/special/discount/reward/stock
  subqueries; the single most contested method for label/badge/price modules. Prefer the
  `catalog/view/product/*` events where possible.
- `catalog/controller/product/product.php::index` — product page assembly (options, images, tabs, related).
- `product.twig` — price block, options, gallery, tabs, related cards.
- `category.twig` / `search.twig` — listing cards, filters, pagination.
- `admin/model/catalog/product.php::addProduct`/`editProduct` — extra columns/relations at write time.
- `admin/view/template/catalog/product_form.twig` — admin product tabs.

## Theme Override Risks

- Themes heavily override `product.twig`, `category.twig`, `search.twig`, and the product card
  partials (gallery, price, options, tabs, related). Verify the ACTIVE theme's copies before editing default.
- LiveStore's `noindex`/`meta_h1`/`main_category` extras may be surfaced by the theme; a vanilla
  theme will not render them.

## Common Extension Collision Zones

- Filter/search modules (touch `getProducts`/`getTotalProducts` and `product_filter`).
- Product label/badge modules (read `product_special`/`product_discount`, hook `getProduct`).
- Option/price modifier modules.
- Gallery / image-zoom modules (override `product_image` rendering).
- SEO/meta modules (interact with `seo_url`, `meta_h1`, `noindex`, `config_seo_pro`).
- LiveStore relation extras (`product_related_article` / `_wb` / `_mn`) — project/build specific.

## Verification Notes

Verified against LiveStore `v3.0.4.4` on 2026-07-04. Confidence split by how each item was checked.

### `[verified]` — read in full from source
- `catalog/model/catalog/product.php` (all methods, incl. `getProduct` price/special/discount SQL).
- `admin/model/catalog/product.php::addProduct` + `editProduct` (full write-table set, incl. LS extras).
- `install/opencart.sql`: the `oc_product*` `CREATE TABLE` blocks (lines ~2439–3021), the LS extra
  columns/tables (`product.noindex` 2471, `product_description.meta_h1` 2614,
  `product_to_category.main_category` 2942, `product_related_article` 4797, `_wb` 4935, `_mn` 4959),
  and the `advertise_google` product event rows (lines 1251–1260).

### `[baseline]` — confirmed by targeted grep, not full read
- `public function` lists for `admin/controller/catalog/product.php`,
  `catalog/controller/product/product.php`, and the remaining methods of
  `admin/model/catalog/product.php`.
- Model/view/language calls inside `catalog/controller/product/product.php` (grep only).

### `[exists]` — path existence-checked only (HTTP HEAD), CONTENT not read
- Negative existence checks (HTTP HEAD returned 404): no `catalog/controller/api/product.php`;
  no `product/manufacturer.twig`.
- Storefront listing controllers `category.php`/`search.php`/`manufacturer.php`/`special.php`.
- `catalog/model/catalog/manufacturer.php`.
- All product view templates and language files cited in `## Core Files`.

### `[needs-verification]` — re-check before relying
- Exact bodies/signatures of listing controllers and admin controller methods.
- Exact args of the product view/model events.
- On the live project: active-theme product templates, extra `product*` columns, filter/SEO/label
  modules, and (on non-LiveStore baselines) the `noindex`/`meta_h1`/`main_category`/`product_related_*` extras.
