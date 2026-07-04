# Mail

## Scope

This domain covers OpenCart mail dispatch triggers, mail templates/views, admin alerts, customer notifications, and extension collision points around order/customer/account emails.

## Baseline

- Version: pending initial verification
- Source: pending verified upstream baseline
- Verified on: pending

## Main Routes

- account registration-related flows
- order confirmation and status update flows
- password reset flows
- admin alert flows

## Core Files

### Controllers

- controllers that trigger mail sends: pending verification

### Models

- model paths that prepare order/customer payloads for email: pending verification

### Views

- mail template files: pending verification

### Language Files

- customer/admin email subject/body strings: pending verification

## Data Flow

Expected high-level chain:

1. business event occurs
2. data payload is assembled
3. mail engine/settings are resolved
4. template and language strings are rendered
5. dispatch is attempted

## Database

### Tables

- mail itself is mostly config-driven rather than table-driven
- related domain tables depend on the event source

## Events

- mail-related event hooks: pending verification

## API Links

- indirect only; some API actions trigger mail through normal business flows

## Mail Links

- this is the primary domain

## OCMOD Hotspots

- order confirmation email text
- order status mail text
- registration mail text
- admin new-order alerts

## Theme Override Risks

- mail output may be altered by template engines, theme mail packs, or custom HTML mail modules

## Common Extension Collision Zones

- SMTP/mailer modules
- branded email template packs
- order status automation modules
- marketplace or CRM connectors

## Verification Notes

- This file is a starter pack and still requires exact baseline verification.
