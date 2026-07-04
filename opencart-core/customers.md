# Customers

## Scope

This domain covers customer registration, login, profile editing, password reset, customer groups, addresses, approvals, admin customer management, and related account-side effects.

## Baseline

- Version: pending initial verification
- Source: pending verified upstream baseline
- Verified on: pending

## Main Routes

- account register/login/edit/password/address routes
- admin customer routes
- API customer routes where applicable

## Core Files

### Controllers

- storefront account controllers: pending verification
- admin customer controllers: pending verification

### Models

- catalog account customer/address models: pending verification
- admin customer model chain: pending verification

### Views

- account form templates: pending verification
- admin customer forms/lists: pending verification

### Language Files

- account and admin customer language files: pending verification

## Data Flow

Expected high-level chain:

1. customer submits account/auth/profile data
2. validation and group rules run
3. customer and related rows are updated
4. approval/session/mail side effects may run

## Database

### Tables

- `customer`
- `customer_group`
- `address`
- related approval/reward/affiliate tables pending verification

### Important Columns

- group, status, approved, safe/custom field mappings, address defaults

## Events

- account/customer event hooks: pending verification

## API Links

- customer authentication and customer management API points: pending verification

## Mail Links

- registration approval and password reset mails

## OCMOD Hotspots

- registration validation
- login/password flows
- customer group logic
- admin customer edit/save flows

## Theme Override Risks

- themes often alter registration, login, account edit, and custom field rendering

## Common Extension Collision Zones

- one-page checkout/account modules
- social login modules
- B2B/customer group modules
- approval/verification add-ons

## Verification Notes

- This file is a starter pack and still requires exact baseline verification.
