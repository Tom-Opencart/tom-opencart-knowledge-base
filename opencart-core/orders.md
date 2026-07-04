# Orders

## Scope

This domain covers order creation, order persistence, order retrieval, order totals, status transitions, history writes, and links to checkout, mail, and API flows.

## Baseline

- Version: pending initial verification
- Source: pending verified upstream baseline
- Verified on: pending

## Main Routes

- catalog checkout flow routes
- admin sale order routes
- API order-related routes

## Core Files

### Controllers

- catalog order-related controllers: pending verification
- admin sale order controllers: pending verification
- API order controllers: pending verification

### Models

- order model chain: pending verification
- checkout/order total interaction points: pending verification

### Views

- admin order list and order info pages: pending verification
- storefront checkout success and history links: pending verification

### Language Files

- admin sale order language chain: pending verification
- checkout and account order language files: pending verification

## Data Flow

Expected high-level chain:

1. cart and checkout data is validated
2. order payload is assembled
3. order row and related rows are written
4. history/status is written
5. totals, stock, affiliate, reward, and mail side effects may run

## Database

### Tables

- `order`
- `order_product`
- `order_option`
- `order_total`
- `order_history`
- `order_voucher`
- other linked order tables pending verification

### Important Columns

- order identifiers
- customer/store metadata
- status identifiers
- totals/currency fields
- payment/shipping method snapshots

## Events

- order create/update event points: pending verification

## API Links

- order creation and lookup via API: pending verification

## Mail Links

- order confirmation and status-change mail links: pending verification

## OCMOD Hotspots

- order add/edit confirmation areas
- status update logic
- admin order info rendering
- checkout confirmation chain

## Theme Override Risks

- checkout themes often alter confirmation and success flow
- some themes inject extra totals, field mappings, or custom validation

## Common Extension Collision Zones

- checkout extensions
- payment integrations
- stock sync modules
- CRM/export/invoice modules
- mail template modifiers

## Verification Notes

- This file is a starter pack and still requires exact baseline verification.
