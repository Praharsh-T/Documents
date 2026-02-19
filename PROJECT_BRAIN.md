# GOVERNMENT-GRADE SMART METER INSTALLATION MANAGEMENT SYSTEM
## MASTER CONTEXT + FRONTEND PROMPT + BACKEND PROMPT + INTEGRATION PROMPT

This document is the SINGLE SOURCE OF TRUTH for this project.

If any code, schema, or logic violates this document, it is WRONG.

────────────────────────────────────────────────────────────

# 1. SYSTEM NATURE

This is NOT:
❌ A CRUD app  
❌ A demo  
❌ A simple dashboard  

This IS:
✅ A lifecycle-governed system  
✅ A multi-role platform  
✅ A state-machine-based domain  
✅ An audit-heavy government-grade system  

Shortcuts are NOT allowed.

────────────────────────────────────────────────────────────

# 2. TECH STACK

Frontend:
- Next.js (App Router)
- Tailwind CSS
- TypeScript
- GraphQL Client (Apollo/urql)

Backend:
- NestJS
- GraphQL (Code-first)
- PostgreSQL
- Drizzle ORM
- JWT Authentication
- RBAC
- State Machines
- Audit Logs

────────────────────────────────────────────────────────────

# 3. USER ROLES (NEVER MERGE)

## Role Hierarchy (Creation Permissions)
- SUPER_ADMIN → creates ADMIN
- ADMIN → creates RETAIL_OUTLET (RO)
- RETAIL_OUTLET → creates CONTRACTOR, SUB_USER, CONSUMER

## All Roles
- SUPER_ADMIN (System-wide administration)
- ADMIN (Regional administration)
- CONSUMER (End user with OTP-based login)
- RETAIL_OUTLET (RO - Operations management)
- CONTRACTOR (Installation and maintenance)
- SUB_USER (Assistant to RO)

Each role has distinct permissions, workflows, and UI.

## Authentication Methods
- SUPER_ADMIN, ADMIN, RO, CONTRACTOR, SUB_USER: Email + Password
- CONSUMER: Phone + OTP (Dummy OTP for testing: 123456)

────────────────────────────────────────────────────────────

# 4. CORE DOMAIN MODEL

### Consumer
A person or organization.

### Site (Connection)
A physical service location.
A consumer can have MULTIPLE sites.

### Meter
A physical device.
A site can have MULTIPLE meters.
A meter can belong to ONLY ONE site at a time.

### Site–Meter Connection (MANDATORY DOMAIN ENTITY)
This is NOT a simple join table.

It must:
- Track assignment lifecycle
- Preserve history
- Support replacements
- Support rollbacks
- Store tariff snapshot
- Support audit
- Prevent overwrites

────────────────────────────────────────────────────────────

# 5. LIFECYCLE STATE MACHINES

## Meter Lifecycle
AVAILABLE → RESERVED → INSTALLED → FAILED  
RESERVED → TIMEOUT → AVAILABLE  

## Site Lifecycle
CREATED → SITE_PENDING → SITE_VERIFIED → METER_ASSIGNED → INSTALLED → VERIFIED → BILLED  

Illegal transitions must be BLOCKED.

────────────────────────────────────────────────────────────

# 6. STRICT BUSINESS RULES

1. A consumer can have multiple sites.
2. A site can have multiple meters.
3. A meter can be attached to only one site at a time.
4. Site–meter linkage must be via a domain join entity.
5. Assignment must never overwrite history.
6. Replacement must create a new record.
7. Tariff code comes from Excel and MUST be persisted.
8. Tariff code must be immutable after site verification.
9. Tariff must be copied into the site–meter connection at assignment.
10. Meter compatibility must be validated against site requirements.
11. Installation cannot happen before assignment.
12. Verification cannot happen before installation.
13. Billing cannot happen before verification.
14. No step can be skipped.
15. Every state change must be audited.
16. Every action must be timestamped.
17. Frontend is NEVER the authority.
18. Backend enforces ALL business logic.

────────────────────────────────────────────────────────────

# 7. METER ATTRIBUTES (NORMALIZED, NOT OPTIONAL)

- Phase (1P / 3P) - REQUIRED for meter-site matching
- Voltage
- Network Type (RF, GPRS, NB_IOT, LORA, DLMS, MODBUS) - REQUIRED for meter-site matching
- RAPDRP / Non-RAPDRP - REQUIRED for meter-site matching
- AEH / Non-AEH
- Prepaid / Postpaid

These must be structured and validated.

## Meter-Site Matching Rules
When assigning a meter to a site, the following must match:
1. Phase type (SINGLE_PHASE / THREE_PHASE)
2. Network type (if site has requirement)
3. RAPDRP status (if site has requirement)

────────────────────────────────────────────────────────────

# 8. DATABASE PRINCIPLES

You MUST have:

- Consumers
- Consumer Sites
- Meters
- Meter Specs
- Site–Meter Connections
- Installations
- Installation Evidence
- Verifications
- Audit Logs
- Uploads
- State Transitions
- Billing Exports

Never:
❌ Attach meter directly to consumer  
❌ Overwrite assignments  
❌ Skip history  
❌ Skip audits  

────────────────────────────────────────────────────────────

# 9. GRAPHQL RULES

- Code-first schema
- Strong typing
- Strict enums
- Role-guarded resolvers
- No overfetching
- No illegal transitions
- Field-level RBAC
- Pagination & filtering

────────────────────────────────────────────────────────────

# 10. ERROR CONTRACT

{
  code: string,
  message: string,
  field?: string,
  entity?: string,
  state?: string
}

────────────────────────────────────────────────────────────

# 11. AUDIT CONTRACT

Every state-changing mutation must log:
- Actor
- Role
- Entity
- Entity ID
- Action
- Old State
- New State
- Timestamp

────────────────────────────────────────────────────────────

# 12. FRONTEND PROMPT

You are a senior frontend engineer building a production-grade enterprise web application.

STACK:
- Next.js (App Router)
- Tailwind CSS
- TypeScript
- GraphQL

Web-only (no mobile app).

Simulate:
- QR scanning via webcam
- Geo capture via browser
- Photo capture
- OCR upload UI

Roles:
- Admin
- Consumer
- RO
- Contractor

Rules:
- State-driven UI
- No skipping steps
- No illegal transitions
- No hardcoded permissions
- Backend is the authority

Folder structure:

/app
  /(auth)
  /(admin)
  /(consumer)
  /(ro)
  /(contractor)
  /components
  /graphql
  /hooks
  /lib
  /types

You must:
- Implement role layouts
- Guards
- Lifecycle-based UI
- QR scanning UI
- Photo capture
- Geo tagging
- Upload flows
- Dashboards
- Audit timeline UI

────────────────────────────────────────────────────────────

# 13. BACKEND PROMPT

You are a senior backend architect.

STACK:
- NestJS
- GraphQL (code-first)
- PostgreSQL
- Drizzle
- JWT
- RBAC
- State machines
- Audit logs

This is NOT CRUD.

You must implement:

Modules:
- Auth
- Users
- Consumers
- Sites
- Meters
- Meter Specs
- Site–Meter Connections
- Inventory
- Installation
- Verification
- Billing
- Audit
- Uploads
- Excel Imports

Rules:
- Enforce lifecycle
- Enforce transitions
- Enforce ownership
- Enforce compatibility
- Enforce tariff persistence
- Enforce multi-meter per site
- Enforce history preservation

Never:
- Put business logic in resolvers
- Skip validation
- Allow illegal transitions
- Overwrite assignments

────────────────────────────────────────────────────────────

# 14. INTEGRATION PROMPT

You are responsible for frontend ↔ backend integration.

You must:

- Map every frontend page to GraphQL contracts
- Enforce backend authority
- Validate lifecycle
- Validate transitions
- Validate role access
- Validate multi-meter per site
- Validate site–meter join entity
- Validate tariff persistence
- Validate audit creation
- Validate history preservation

If backend does not support these rules:
→ You MUST change the backend.

Never:
- Let frontend invent logic
- Let frontend skip steps
- Let frontend hardcode state

────────────────────────────────────────────────────────────

# 15. FORBIDDEN PATTERNS

❌ CRUD-only modeling  
❌ No state machines  
❌ No audit logs  
❌ Meter directly linked to consumer  
❌ No site abstraction  
❌ No join-domain entity  
❌ No tariff persistence  
❌ Overwriting assignments  
❌ Skippable steps  

────────────────────────────────────────────────────────────

# 16. IMPLEMENTATION ORDER

1. Domain modeling
2. Lifecycle modeling
3. Entity relationships
4. State machines
5. Database schema
6. Backend logic
7. GraphQL contracts
8. Frontend mapping
9. Integration
10. Auditing
11. Billing handoff

────────────────────────────────────────────────────────────

If anything violates this file, it is WRONG.
