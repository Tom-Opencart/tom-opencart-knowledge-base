# Products

## Scope

This domain covers storefront product rendering, admin product management, pricing/discount/special flows, options, images, manufacturer/category relations, and common theme override zones.

## Baseline

- Version: pending initial verification
- Source: pending verified upstream baseline
- Verified on: pending

## Main Routes

- catalog product page route
- category/search/manufacturer listing routes
- admin catalog product routes

## Core Files

### Controllers

- storefront product and listing controllers: pending verification
- admin product controller chain: pending verification

### Models

- catalog product model chain: pending verification
- admin catalog product model chain: pending verification

### Views

- product page and listing templates: pending verification
- admin product form/list templates: pending verification

### Language Files

- storefront product language files: pending verification
- admin catalog product language files: pending verification

## Data Flow

Expected high-level chain:

1. product data is fetched
2. pricing/tax/special/discount logic is applied
3. images/options/attributes are resolved
4. controller builds template payload
5. theme renders product or listing output

## Database

### Tables

- `product`
- `product_description`
- `product_to_category`
- `product_to_store`
- `product_image`
- `product_option`
- `product_option_value`
- `product_discount`
- `product_special`
- related tables pending verification

### Important Columns

- identifiers, status, date, stock, price, tax, model, manufacturer

## Events

- product-related event hooks: pending verification

## API Links

- API product read/write areas: pending verification

## Mail Links

- indirect only unless extensions add alerts

## OCMOD Hotspots

- price block
- option block
- image block
- attribute/specification block
- product card/listing markup

## Theme Override Risks

- themes heavily override product cards, gallery, tabs, options, price, and related product areas

## Common Extension Collision Zones

- filter/search modules
- product label/badge modules
- option/price modifiers
- gallery/image zoom modules
- SEO/meta modules

## Verification Notes

- This file is a starter pack and still requires exact baseline verification.
