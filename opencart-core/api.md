# API

## Scope

This domain covers OpenCart API authentication, token/session handling, API controllers, order/cart/customer interactions exposed through the API layer, and common extension hook points.

## Baseline

- Version: pending initial verification
- Source: pending verified upstream baseline
- Verified on: pending

## Main Routes

- API login/auth routes
- cart routes
- order routes
- customer/address routes
- product-related routes where applicable

## Core Files

### Controllers

- `catalog/controller/api/*` chain: pending verification

### Models

- models called by API controllers: pending verification

### Views

- API is mostly response-based rather than template-based

### Language Files

- API error/success message language files: pending verification

## Data Flow

Expected high-level chain:

1. API auth/token is validated
2. request payload is normalized
3. controller dispatches to standard domain models
4. response payload is returned

## Database

### Tables

- API-related auth/session tables pending verification
- downstream domain tables depend on the action

## Events

- API event hooks: pending verification

## API Links

- this is the primary domain

## Mail Links

- indirect only when API actions trigger normal business flows such as order confirmation

## OCMOD Hotspots

- token validation
- cart/order payload handling
- response normalization

## Theme Override Risks

- usually lower than storefront themes, but some stores mix API logic with custom checkout modules

## Common Extension Collision Zones

- mobile app/API bridges
- ERP/CRM connectors
- custom checkout integrations
- marketplace sync modules

## Verification Notes

- This file is a starter pack and still requires exact baseline verification.
