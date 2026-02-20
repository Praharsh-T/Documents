# SmartMeter — System State Report

> **Generated**: 2026-02-20 · Based on full codebase analysis  
> **Backend**: `user-service/` · **Frontend**: `web-frontend/`

---

## 1. System Understanding

### Purpose

SmartMeter is a **government-grade smart meter installation management system** for **MESCOM** (Mangalore Electricity Supply Company). It digitizes the complete lifecycle of smart meter installation for new electricity connections — from consumer registration through meter assignment, physical installation, verification, and billing handoff.

### Architecture

```
┌─────────────────────┐     GraphQL      ┌─────────────────────────┐
│   Next.js 16 App    │ ◄──────────────► │   NestJS 11 Backend     │
│   React 19 + Apollo │                  │   Code-First GraphQL    │
│   Tailwind CSS      │                  │   Drizzle ORM           │
└─────────────────────┘                  │   PostgreSQL 17         │
                                         └──────┬──────────────────┘
                                                 │
                    ┌────────────────────────────┼────────────────────────┐
                    │                            │                        │
             ┌──────▼──────┐         ┌───────────▼───────┐      ┌────────▼───────┐
             │ Azure Blob  │         │   PostgreSQL DB   │      │  External APIs │
             │ / Local     │         │   (Drizzle ORM)   │      │  Fast2SMS      │
             │ / GCP (N/A) │         │   22 tables       │      │  AWS SES       │
             └─────────────┘         └───────────────────┘      │  Cashfree(N/A) │
                                                                 └────────────────┘
```

**Pattern**: Strict **modular architecture** — each domain has its own NestJS module with `resolver → service → database` data flow. 22 feature modules registered in `AppModule`.

### Key Architecture Decisions

- **GraphQL Code-First** via `@nestjs/graphql` with auto-generated `schema.gql`
- **Drizzle ORM** for type-safe database operations (no raw SQL)
- **JWT + Passport** authentication with refresh token rotation
- **State machine enforcement** at service layer (not frontend)
- **Audit logging** on every state change
- **RBAC** via custom `@Roles()` decorator + `RolesGuard`
- **Data isolation** — each role only sees data they own/created

### Role Hierarchy

```
SUPER_ADMIN
    └──► creates ADMIN
              └──► creates RETAIL_OUTLET (RO)
                        └──► creates CONTRACTOR / CONSUMER / SUB_USER
```

### Module Interaction Map

```
Auth ────► Users ────► Consumers ────► Sites ────► Meters
                           │              │            │
                           │              ▼            ▼
                           │       Installations ◄─ SiteMeterConnections
                           │              │
                           │              ▼
                           │       Verifications ────► Billing (TODO)
                           │
                           └──► Marketplace (Jobs ↔ Payments ↔ Ratings)
                                     │
                                ┌────┼────┐
                                ▼    ▼    ▼
                             SLA  Subs  Pricing
```

---

## 2. Module-wise Breakdown

---

### 2.1 Auth Module

| Attribute         | Detail                                                                         |
| ----------------- | ------------------------------------------------------------------------------ |
| **Description**   | JWT authentication with email/password + OTP login, refresh tokens, throttling |
| **Service Lines** | 588                                                                            |
| **Status**        | ✅ Completed                                                                   |

**Features Implemented**:

- Email/password login for ADMIN, RO, CONTRACTOR roles
- OTP login for ALL roles (Consumer primary, others optional)
- Configurable OTP mode (`DUMMY` / `ACTUAL` via env)
- JWT access token generation with role/userId/email claims
- Refresh token generation, storage, rotation, and revocation
- Login attempt throttling via `auth_throttle` table (time-based lockout)
- `me` query for authenticated user profile retrieval
- `logout` mutation — invalidates refresh tokens
- SMS OTP delivery via `SmsService` (Fast2SMS)

**APIs**: `login`, `requestLoginOtp`, `verifyLoginOtp`, `requestOtp`, `verifyOtp`, `refreshAccessToken`, `logout`, `me`

**Data Models**: `users`, `refreshTokens`, `authThrottle`, `otpCodes`

**Pending**:

- ⚠️ `me` query throws raw `Error('User not found')` instead of NestJS `NotFoundException` — will not be caught by global exception filter ([auth.resolver.ts:91](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/auth/auth.resolver.ts#L91))

---

### 2.2 Users Module

| Attribute       | Detail                                                            |
| --------------- | ----------------------------------------------------------------- |
| **Description** | Multi-role user management with hierarchical creation permissions |
| **Status**      | ✅ Completed                                                      |

**Features Implemented**:

- Role-specific creation: `createAdmin` (SA), `createRetailOutlet` (ADMIN), `createContractor` (RO), `createSubUser` (RO), `createConsumer` (RO/SUB_USER)
- User listing with role filter and data isolation
- `activateUser`, `deactivateUser`, `deleteUser` lifecycle operations
- Contractors by division/RO queries
- Password hashing via `bcryptjs`

**APIs**: `users`, `user`, `createAdmin`, `createRetailOutlet`, `createUser`, `createContractor`, `createSubUser`, `deactivateUser`, `activateUser`, `deleteUser`, `contractorsByDivision`, `contractorsByRo`

**Data Models**: `users`

**Pending**: None — fully implemented.

---

### 2.3 Consumers Module

| Attribute       | Detail                                                                                  |
| --------------- | --------------------------------------------------------------------------------------- |
| **Description** | Consumer profile management with lifecycle state (register/activate/suspend/deactivate) |
| **Status**      | ✅ Completed                                                                            |

**Features Implemented**:

- CRUD operations with pagination and filtering (`ConsumerFiltersInput`)
- Lifecycle: `activateConsumer`, `suspendConsumer`, `deactivateConsumer`
- `myConsumer` query for consumer self-service profile
- ResolveField resolvers for `circleName`, `divisionName`, `subdivisionName`

**APIs**: `consumers`, `consumer`, `consumerByConsumerId`, `myConsumer`, `createConsumer`, `updateConsumer`, `activateConsumer`, `suspendConsumer`, `deactivateConsumer`

**Data Models**: `consumers` (linked to `users`, hierarchy)

**Anti-Pattern**:

- ⚠️ `ConsumersResolver` directly injects raw `db` for hierarchy field resolvers instead of using `HierarchyService` — violates modular architecture

---

### 2.4 Sites Module

| Attribute         | Detail                                                             |
| ----------------- | ------------------------------------------------------------------ |
| **Description**   | Consumer site (connection) management with state machine lifecycle |
| **Service Lines** | 1,721                                                              |
| **Status**        | ✅ Completed                                                       |

**Features Implemented**:

- Full state machine: `CREATED → SITE_PENDING → SITE_VERIFIED → INSTALLATION_SCHEDULED → METER_ASSIGNED → INSTALLATION_IN_PROGRESS → INSTALLED → VERIFICATION_PENDING → VERIFIED → BILLED`
- `FAILED` state accessible from any stage; can restart to `CREATED` or `SITE_PENDING`
- Tariff code locked at `SITE_VERIFIED` (immutable after)
- Site photo upload (base64 + URL-based)
- Geo-location capture with accuracy tracking
- Site readiness status: `PENDING → PHOTO_UPLOADED → LOCATION_VERIFIED → APPROVED → REJECTED`
- Data isolation via `getAccessibleSubdivisionIds` (multi-subdivision per RO)
- Site statistics query with counts by state
- Contractor assignment, appointment scheduling
- 15+ granular update mutations (address, location, requirements, RO fields, etc.)
- Photo URL generated on-demand via Azure Blob signed URLs

**APIs**: `sites`, `site`, `siteBySiteId`, `sitesByConsumer`, `mySites`, `siteStatistics`, `createSite`, `updateSiteAddress`, `updateSiteLocation`, `updateSiteRequirements`, `updateSiteROFields`, `uploadSitePhoto`, `uploadSitePhotoBase64`, `uploadSitePhotoUrl`, `verifySite`, `scheduleAppointment`, `assignContractorToSite`, `activateConsumerSite`

**Data Models**: `consumerSites` (with 11 state enum values, readiness status)

**Pending**:

- ⚠️ TODO at [sites.service.ts:1689](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/sites/sites.service.ts#L1689): `// TODO: Call NotificationService.sendRegistrationSms()` — registration SMS not connected

---

### 2.5 Meters Module

| Attribute       | Detail                                                             |
| --------------- | ------------------------------------------------------------------ |
| **Description** | Smart meter inventory management with full lifecycle state machine |
| **Status**      | ✅ Completed                                                       |

**Features Implemented**:

- Meter CRUD with serial number, QR code, phase type, voltage, network type
- State machine: `AVAILABLE → RESERVED → ASSIGNED → INSTALLED → ACTIVE → FAILED → DECOMMISSIONED`
- Assignment to consumers via `assignMeter`
- Installation marking (`markMeterInstalled`) and verification (`verifyMeterInstallation`, `markMeterFailed`)
- Release and re-assignment (`releaseMeter`)
- Lookup by serial number and QR code
- Inventory summary (counts by state per RO)

**APIs**: `meters`, `meter`, `meterBySerialNumber`, `meterByQrCode`, `inventorySummary`, `createMeter`, `assignMeter`, `markMeterInstalled`, `verifyMeterInstallation`, `markMeterFailed`, `releaseMeter`

**Data Models**: `meters`, `meterSpecs`

**Pending**: None — fully implemented.

---

### 2.6 Meter Specs Module

| Attribute       | Detail                                                        |
| --------------- | ------------------------------------------------------------- |
| **Description** | Meter specification management (phase, voltage, network type) |
| **Status**      | ✅ Completed                                                  |

**Features Implemented**:

- CRUD for meter specifications
- Linked to meters for compatibility validation

---

### 2.7 Site-Meter Connections Module

| Attribute       | Detail                                                                           |
| --------------- | -------------------------------------------------------------------------------- |
| **Description** | Domain entity tracking meter-site assignment lifecycle (not a simple join table) |
| **Status**      | ✅ Completed                                                                     |

**Features Implemented**:

- Assignment history tracking with status: `ACTIVE → REPLACED → REMOVED → ROLLED_BACK`
- Tariff snapshot persistence (immutable after verification)
- Meter replacement flow preserving full history
- Rollback capability

**Data Models**: `siteMeterConnections`

---

### 2.8 Installations Module

| Attribute       | Detail                                                                |
| --------------- | --------------------------------------------------------------------- |
| **Description** | Installation record management linking sites, meters, and contractors |
| **Status**      | ✅ Completed                                                          |

**Features Implemented**:

- Single and bulk installation creation (`createInstallation`, `createMultipleInstallations`)
- Status transitions: `PENDING → SCHEDULED → IN_PROGRESS → COMPLETED → VERIFIED` and `FAILED → CANCELLED → REJECTED`
- Contractor assignment and filtering
- ResolveField resolvers: site, meter, contractor, consumer
- Installations by site, by contractor, by consumer queries

**APIs**: `installations`, `installation`, `installationBySite`, `myInstallations`, `installationStatusByConsumer`, `createInstallation`, `createMultipleInstallations`, `updateInstallationStatus`

**Data Models**: `installations`

**Anti-Pattern**:

- ⚠️ Uses placeholder meter ID (`00000000-0000-0000-0000-000000000000`) in multi-installation creation when meter not yet assigned ([installations.service.ts:114](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/installations/installations.service.ts#L114))

---

### 2.9 Installation Evidence Module

| Attribute       | Detail                                                                        |
| --------------- | ----------------------------------------------------------------------------- |
| **Description** | Evidence capture module for installation verification (photos, GPS, readings) |
| **Status**      | ✅ Completed                                                                  |

**Features Implemented**:

- Evidence upload linked to installations
- File type support: `SITE_PHOTO`, `INSTALLATION_PHOTO`, `OCR_IMAGE`, `VERIFICATION_PHOTO`

---

### 2.10 Verifications Module

| Attribute       | Detail                                                  |
| --------------- | ------------------------------------------------------- |
| **Description** | RO/Admin review and approval of completed installations |
| **Status**      | ✅ Completed                                            |

**Features Implemented**:

- `createVerification` for RO/SUB_USER/ADMIN
- `reviewVerification` with approve/reject flow
- Status enum: `PENDING → APPROVED → REJECTED`
- Paginated listing with filters

**APIs**: `verifications`, `verification`, `createVerification`, `reviewVerification`

**Data Models**: `verifications`

---

### 2.11 Billing Module

| Attribute       | Detail                                                |
| --------------- | ----------------------------------------------------- |
| **Description** | Billing export and handoff for verified installations |
| **Status**      | ❌ Not Implemented                                    |

**What Exists**: Empty placeholder module — resolver, service, and module files exist but contain zero logic.

**TODOs Found** ([billing.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/billing/billing.service.ts)):

```
TODO: Implement:
1. Query VERIFIED sites with ACTIVE meter connections
2. Generate CSV/Excel file with billing data
3. Upload file to storage
4. Return download URL
```

**Missing**:

- `generateBillingExport` mutation
- `billingExports` query
- `billingSummary` query
- CSV/Excel generation logic
- Integration with file storage
- Billing handoff workflow

---

### 2.12 Payments Module

| Attribute         | Detail                                                        |
| ----------------- | ------------------------------------------------------------- |
| **Description**   | Marketplace payment processing (Cashfree gateway placeholder) |
| **Service Lines** | 500                                                           |
| **Status**        | ⚠️ Partially Implemented                                      |

**Features Implemented**:

- Payment and refund data models with full schema
- `initiatePayment` mutation creates payment records in DB
- `handlePaymentWebhook` mock implementation
- `initiateRefund` with refund record creation
- Payment/refund queries with pagination and filtering
- Payment/refund retrieval by ID and by job ID

**What is Placeholder** ([payments.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/payments/payments.service.ts)):

- `initiatePayment` returns mock session ID: `MOCK_SESSION_${payment.id}`
- `handlePaymentWebhook` has no actual Cashfree webhook validation
- `initiateRefund` does not call Cashfree Refund API
- All Cashfree gateway fields in DB are marked as placeholders

**Missing**:

- Cashfree API integration (order creation, webhook signature verification)
- Payment gateway REST controller for webhook endpoints
- Actual refund processing via Cashfree
- Payment reconciliation logic

---

### 2.13 Marketplace — Jobs Module

| Attribute         | Detail                                                           |
| ----------------- | ---------------------------------------------------------------- |
| **Description**   | Consumer-contractor marketplace with job lifecycle state machine |
| **Service Lines** | 927                                                              |
| **Status**        | ⚠️ Partially Implemented                                         |

**Features Implemented**:

- Job creation (consumer) → payment pending → requested → accepted → started → completed → rated → closed
- Job number generation (unique sequential)
- Contractor accept/reject with scheduling
- OTP generation for job start/completion verification
- Consumer cancellation
- SLA deadline tracking (acceptance, start, completion)
- SLA breach recording
- `simulatePaymentSuccess` for development
- Full job query with filters, consumer jobs, contractor jobs

**TODOs Found** ([jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)):
| Line | TODO |
|------|------|
| 248 | `// TODO: Send SMS to contractor about new job request` |
| 304 | `// TODO: Send SMS to consumer about job acceptance` |
| 349 | `// TODO: Trigger refund process` |
| 350 | `// TODO: Send SMS to consumer about rejection and refund` |
| 391 | `// TODO: Trigger refund if payment was made` |
| 392 | `// TODO: Send SMS to contractor about cancellation` |
| 446 | `// TODO: Send OTP to consumer via SMS` |
| 565 | `// TODO: Send SMS notification` |

**Missing**:

- 7 SMS notification integrations
- 2 refund trigger integrations with PaymentsService
- Actual OTP delivery to consumer (currently only stored in DB)

---

### 2.14 Marketplace — Services Catalog Module

| Attribute       | Detail                                                          |
| --------------- | --------------------------------------------------------------- |
| **Description** | Service catalog with categories, pricing, and area type support |
| **Status**      | ✅ Completed                                                    |

**Features Implemented**: Full CRUD, services with prices by area type, services by category, activate/deactivate

---

### 2.15 Marketplace — Pricing Module

| Attribute       | Detail                                              |
| --------------- | --------------------------------------------------- |
| **Description** | Area-type-specific pricing for marketplace services |
| **Status**      | ✅ Completed                                        |

---

### 2.16 Marketplace — Ratings Module

| Attribute       | Detail                                       |
| --------------- | -------------------------------------------- |
| **Description** | Consumer ratings and reviews for contractors |
| **Status**      | ✅ Completed                                 |

**Features Implemented**: Create rating, rating summary, contractor reviews with pagination, admin listing

---

### 2.17 Marketplace — SLA Module

| Attribute       | Detail                                 |
| --------------- | -------------------------------------- |
| **Description** | SLA configuration and breach detection |
| **Status**      | ⚠️ Partially Implemented               |

**Features Implemented**:

- SLA configuration per service + area type
- Bulk SLA setting
- SLA matrix queries
- Scheduled breach detection (every 5 minutes) for acceptance, start, and completion deadlines
- Breach recording in database

**TODOs Found** ([sla-scheduler.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/sla/sla-scheduler.service.ts)):
| Line | TODO |
|------|------|
| 43 | `// TODO: Send notification to admin` |
| 44 | `// TODO: Consider auto-cancelling and refunding` |
| 82 | `// TODO: Send notification to consumer and admin` |
| 121 | `// TODO: Send notification to consumer and admin` |
| 122 | `// TODO: Consider partial refund` |

**Missing**:

- Notification dispatch on SLA breach
- Auto-cancellation on acceptance breach
- Partial refund on completion breach

---

### 2.18 Marketplace — Subscriptions Module

| Attribute       | Detail                                  |
| --------------- | --------------------------------------- |
| **Description** | Contractor subscription tier management |
| **Status**      | ✅ Completed                            |

**Features Implemented**: Subscription plans, status, history, upgrade initiation, payment confirmation, admin override

**Anti-Pattern**:

- ⚠️ 6 instances of raw `throw new Error()` instead of NestJS exceptions in [subscriptions.resolver.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/subscriptions/subscriptions.resolver.ts)

---

### 2.19 Marketplace — Contractor Profiles Module

| Attribute       | Detail                                    |
| --------------- | ----------------------------------------- |
| **Description** | Contractor marketplace profile management |
| **Status**      | ✅ Completed                              |

---

### 2.20 Marketplace — Service Locations Module

| Attribute       | Detail                                      |
| --------------- | ------------------------------------------- |
| **Description** | Geographic service coverage area management |
| **Status**      | ✅ Completed                                |

---

### 2.21 Marketplace — Categories & UoMs Modules

| Attribute       | Detail                                          |
| --------------- | ----------------------------------------------- |
| **Description** | Service categorization and units of measurement |
| **Status**      | ✅ Completed                                    |

---

### 2.22 Notification Module

| Attribute         | Detail                                                   |
| ----------------- | -------------------------------------------------------- |
| **Description**   | SMS (Fast2SMS) and Email (AWS SES) notification delivery |
| **Service Lines** | 386                                                      |
| **Status**        | ⚠️ Partially Implemented                                 |

**Features Implemented**:

- `sendWelcomeMessage` — SMS + email with app link
- `sendAppointmentNotification` — SMS + email with date/address
- `sendInstallationCompleteNotification` — SMS + email with meter serial/reading
- Fast2SMS integration with DLT template IDs
- AWS SES email integration
- Dev mode: console logging when gateways disabled
- HTML email templates (welcome, appointment, installation complete)
- Configurable via `SMS_GATEWAY_ENABLED` and `EMAIL_ENABLED` env vars

**Missing**:

- Not connected to marketplace jobs (7 TODOs in jobs.service.ts for SMS)
- Not connected to SLA breach notifications (5 TODOs in sla-scheduler.service.ts)
- Not connected to site registration flow (1 TODO in sites.service.ts)
- No real-time notifications (WebSocket/SSE)
- No push notification support

---

### 2.23 Files Module

| Attribute       | Detail                                 |
| --------------- | -------------------------------------- |
| **Description** | Multi-provider file upload and storage |
| **Status**      | ⚠️ Partially Implemented               |

**Features Implemented**:

- `uploadBase64File` mutation (GraphQL — deprecated)
- REST `POST /files/upload/:mode/:context` endpoint (recommended)
- File retrieval by ID and entity
- Storage providers: **Local** ✅, **Azure Blob** ✅
- Stream uploads for memory efficiency
- Signed URL generation
- File exists/delete/download operations

**Missing**:

- ❌ **GCP Storage** — throws `Error('GCP storage not yet implemented')` at [storage/index.ts:28](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/files/storage/index.ts#L28)

---

### 2.24 Audit Module

| Attribute       | Detail                                              |
| --------------- | --------------------------------------------------- |
| **Description** | Comprehensive audit trail for all system operations |
| **Status**      | ✅ Completed                                        |

**Features Implemented**:

- `auditLogs` query with pagination, entity type, and entity ID filtering
- Supports 16 action types: `CREATE`, `UPDATE`, `DELETE`, `STATE_CHANGE`, `ASSIGN`, `UNASSIGN`, `VERIFY`, `REJECT`, `ACTIVATE`, `LOGIN`, `LOGOUT`, `UPLOAD`, `IMPORT`, `REPLACE`, `ROLLBACK`
- Admin-only access

---

### 2.25 Hierarchy Module

| Attribute       | Detail                                                |
| --------------- | ----------------------------------------------------- |
| **Description** | Geographic hierarchy: Circle → Division → Subdivision |
| **Status**      | ✅ Completed                                          |

**Features Implemented**: Full CRUD for all 3 levels, nested resolvers, RO-subdivision assignment, code/name lookups, `includeInactive` flag

---

### 2.26 Meter Readings Module

| Attribute       | Detail                                             |
| --------------- | -------------------------------------------------- |
| **Description** | Meter reading capture, verification, and retrieval |
| **Status**      | ✅ Completed                                       |

**Features Implemented**: Record reading, verify reading, list by site/meter, latest reading, reading types (INITIAL, PERIODIC, FINAL, SPECIAL)

**Anti-Pattern**:

- ⚠️ Directly injects raw `db` for field resolvers instead of delegating to services

---

### 2.27 Scheduler Module

| Attribute       | Detail                                   |
| --------------- | ---------------------------------------- |
| **Description** | Background cron jobs for automated tasks |
| **Status**      | ✅ Completed                             |

**Features Implemented**:

- Meter reservation timeout (every hour) — releases expired RESERVED meters back to AVAILABLE
- Full transaction with site state rollback, connection cleanup, audit logging
- Configurable timeout via `METER_RESERVATION_TIMEOUT_HOURS` env
- Admin trigger for manual timeout check
- Monitoring: meters nearing timeout count

---

### 2.28 Support / Imports Module

| Attribute         | Detail                                                                      |
| ----------------- | --------------------------------------------------------------------------- |
| **Description**   | Excel/CSV bulk import for consumers, meters, contractors, locations, states |
| **Service Lines** | 967                                                                         |
| **Status**        | ✅ Completed                                                                |

**Features Implemented**:

- Consumer+Site import from Excel with hierarchy resolution
- Meter import from Excel
- Contractor import from Excel with password hashing
- Location and state data imports
- SHA-256 hash deduplication
- Import log tracking (status, counts, errors)
- Batch hierarchy resolution for performance

**Anti-Pattern**:

- ⚠️ Module named `support` but contains `imports.resolver.ts` — naming mismatch
- ⚠️ 18 instances of raw `throw new Error()` instead of NestJS exceptions

---

### 2.29 Retail Outlets Module

| Attribute       | Detail                |
| --------------- | --------------------- |
| **Description** | RO profile management |
| **Status**      | ✅ Completed          |

---

### 2.30 Contractors Module

| Attribute       | Detail                                                  |
| --------------- | ------------------------------------------------------- |
| **Description** | Contractor profile management with division association |
| **Status**      | ✅ Completed                                            |

---

### 2.31 Associations Module

| Attribute       | Detail                                                  |
| --------------- | ------------------------------------------------------- |
| **Description** | Entity association resolver for RO-subdivision mappings |
| **Status**      | ✅ Completed                                            |

---

### 2.32 Frontend (web-frontend)

| Attribute       | Detail                                               |
| --------------- | ---------------------------------------------------- |
| **Description** | Next.js 16 frontend with Apollo Client, Tailwind CSS |
| **Status**      | ✅ Completed                                         |

**Pages (58 total)**:

| Route Group               | Pages | Key Pages                                                                                                                                                      |
| ------------------------- | ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `(auth)`                  | 2     | Login, Register                                                                                                                                                |
| `(admin)/admin`           | 18    | Dashboard, Admins, ROs, Consumers, Sites, Master Data, Upload, Analytics, Marketplace (Services, Categories, UOMs, Pricing, SLA, Locations, Contractors, Jobs) |
| `(ro)/ro`                 | 14    | Dashboard, Site Review, Assign, Schedule, Inventory, Verify, Consumers, Contractors, Installations, Staff                                                      |
| `(consumer)/consumer`     | 10    | Dashboard, Address, Photo, Status, Help, Marketplace (Browse, Services, Book, Jobs, Job Detail)                                                                |
| `(contractor)/contractor` | 14    | Dashboard, Installations, Install Detail, Pickup, Pre-Installation, Issues, Reading, Marketplace (Overview, Profile, Jobs, Job Detail)                         |

**GraphQL Queries/Mutations**: 14 files covering all modules  
**Components**: 23 total (7 UI primitives, 4 shared, 12 marketplace-specific)  
**Custom Hooks**: 15 total (useCamera, useGeoLocation, useQRScanner, useDashboardStats + 10 marketplace hooks)

---

## 3. Codebase & Architecture Analysis

### Folder Structure

```
user-service/src/
├── app.module.ts          # Root module (22 feature modules)
├── config/                # App, DB, JWT, Upload configs
├── database/
│   ├── database.module.ts # Drizzle PostgreSQL provider
│   ├── schema/            # 16+ schema files with enums
│   └── migrations/        # Drizzle-kit generated migrations
├── modules/
│   ├── auth/              # JWT auth + OTP (588 LOC service)
│   ├── users/             # User CRUD + role hierarchy
│   ├── consumers/         # Consumer lifecycle
│   ├── sites/             # Site state machine (1,721 LOC service)
│   ├── meters/            # Meter inventory lifecycle
│   ├── meter-specs/       # Meter specifications
│   ├── meter-readings/    # Reading capture + verification
│   ├── site-meter-connections/ # Domain join entity
│   ├── installations/     # Installation tracking
│   ├── installation-evidence/ # Evidence photos/data
│   ├── verifications/     # Installation verification
│   ├── hierarchy/         # Circle/Division/Subdivision
│   ├── retail-outlets/    # RO profiles
│   ├── contractors/       # Contractor profiles
│   ├── billing/           # PLACEHOLDER (empty)
│   ├── payments/          # Marketplace payments (mock Cashfree)
│   ├── notification/      # SMS + Email service
│   ├── files/             # Multi-provider file storage
│   ├── audit/             # Audit logging
│   ├── scheduler/         # Cron jobs (meter timeout)
│   ├── support/           # Excel imports (misnamed)
│   ├── import-logs/       # Import tracking
│   ├── associations/      # Entity associations
│   └── marketplace/       # 10 sub-modules
│       ├── jobs/          # Job state machine (927 LOC)
│       ├── services-catalog/
│       ├── categories/
│       ├── pricing/
│       ├── ratings/
│       ├── sla/           # SLA + scheduler
│       ├── subscriptions/
│       ├── contractors/   # Contractor profiles
│       ├── service-locations/
│       └── uoms/
└── common/
    ├── decorators/        # @CurrentUser, @Roles
    ├── guards/            # GqlAuthGuard, RolesGuard
    ├── enums/             # UserRole, LoginType
    └── filters/           # Global exception filter
```

### Good Practices Followed

1. ✅ **Strict modular architecture** — clean separation of concerns
2. ✅ **State machine enforcement** at service layer, not frontend
3. ✅ **Audit logging** on every state change
4. ✅ **Data isolation** per role — query-level filtering
5. ✅ **Code-first GraphQL** with auto-generated schema
6. ✅ **Drizzle ORM** — type-safe queries, no raw SQL
7. ✅ **Configurable providers** — storage mode, OTP mode, SMS/email toggles
8. ✅ **Domain entities** — site-meter connections as first-class entities
9. ✅ **Transaction support** — critical operations use `db.transaction()`
10. ✅ **Dedup protection** — SHA-256 hash for imports

### Issues & Anti-Patterns

1. ❌ **Raw `throw new Error()`** — 32+ instances in resolvers instead of NestJS typed exceptions (`NotFoundException`, `BadRequestException`, etc.). These bypass the global exception filter.
2. ❌ **Direct DB injection in resolvers** — `consumers.resolver.ts` and `meter-readings.resolver.ts` inject raw `db` for field resolvers instead of using services
3. ❌ **Module naming mismatch** — `support/` module contains `imports.resolver.ts` (data import logic, not helpdesk)
4. ❌ **Placeholder meter ID** — `installations.service.ts:114` uses `00000000-0000-0000-0000-000000000000` as placeholder
5. ❌ **No input validation DTOs** — GraphQL inputs lack `class-validator` decorators despite user rules mandating it
6. ❌ **Deprecated mutation still active** — `uploadBase64File` is deprecated but has no runtime enforcement

---

## 4. Pending Integrations

| Integration                    | Status             | Detail                                                                                                                                                                         |
| ------------------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **SMS (Fast2SMS)**             | ⚠️ Partial         | Infrastructure complete. Connected to auth OTP + 3 notification types. **NOT connected** to marketplace jobs (7 TODOs), SLA breaches (3 TODOs), or site registration (1 TODO). |
| **Email (AWS SES)**            | ⚠️ Partial         | Infrastructure complete with HTML templates. Same gaps as SMS — only auth + 3 notification types wired up.                                                                     |
| **Payment Gateway (Cashfree)** | ❌ Not Integrated  | Entire payments service uses mock data. No API calls, no webhook handler, no signature verification. DB schema has placeholder fields.                                         |
| **Real-Time Notifications**    | ❌ Not Implemented | No WebSocket, SSE, or push notification support.                                                                                                                               |
| **Background Jobs**            | ✅ Complete        | `@nestjs/schedule` with meter timeout (hourly) + SLA breach detection (every 5 min).                                                                                           |
| **File Storage**               | ⚠️ Partial         | Local ✅, Azure Blob ✅, **GCP ❌** (throws runtime error).                                                                                                                    |
| **Rate Limiting**              | ⚠️ Partial         | Auth throttling exists for login/OTP. **No global rate limiting** on API endpoints.                                                                                            |
| **Input Validation**           | ⚠️ Partial         | GraphQL types enforce some structure but `class-validator` decorators are not applied to DTOs.                                                                                 |
| **Global Exception Filter**    | ⚠️ Partial         | Filter exists but 32+ resolvers throw raw `Error` instead of NestJS exceptions, bypassing it.                                                                                  |

---

## 5. Technical Debt & Improvements

### Critical

| #   | Issue                                       | Files Affected                              | Recommendation                                                                                              |
| --- | ------------------------------------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 1   | **Billing module is completely empty**      | `billing.resolver.ts`, `billing.service.ts` | Implement queries/mutations per TODO comments                                                               |
| 2   | **Cashfree payment gateway not integrated** | `payments.service.ts`, `payments.module.ts` | Replace mock implementations with actual Cashfree API calls                                                 |
| 3   | **Zero test coverage**                      | Entire codebase                             | Only `app.controller.spec.ts` (boilerplate) + `app.e2e-spec.ts` (boilerplate) exist. No module-level tests. |

### High

| #   | Issue                                               | Files Affected                                                                                               | Recommendation                                                                |
| --- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| 4   | **32+ raw `throw new Error()`**                     | `jobs.resolver.ts` (6), `subscriptions.resolver.ts` (6), `imports.resolver.ts` (18+), `auth.resolver.ts` (1) | Replace with `NotFoundException`, `BadRequestException`, `ForbiddenException` |
| 5   | **11 missing notification integrations**            | `jobs.service.ts` (7), `sla-scheduler.service.ts` (3), `sites.service.ts` (1)                                | Wire up existing `NotificationService` to these call sites                    |
| 6   | **No refund trigger in job rejection/cancellation** | `jobs.service.ts:349,391`                                                                                    | Connect `PaymentsService.initiateRefund()` to rejection/cancellation flows    |
| 7   | **GCP storage throws runtime error**                | `storage/index.ts:28`                                                                                        | Implement `CloudStorageProvider` or remove GCP from config options            |

### Medium

| #   | Issue                                     | Files Affected                                        | Recommendation                                      |
| --- | ----------------------------------------- | ----------------------------------------------------- | --------------------------------------------------- |
| 8   | **Direct DB injection in resolvers**      | `consumers.resolver.ts`, `meter-readings.resolver.ts` | Refactor to use respective services                 |
| 9   | **Module naming mismatch**                | `support/imports.resolver.ts`                         | Rename to `imports/` or `data-import/` module       |
| 10  | **Placeholder meter ID in multi-install** | `installations.service.ts:114`                        | Handle gracefully or validate before insert         |
| 11  | **No global rate limiting**               | `app.module.ts`                                       | Add `@nestjs/throttler` for API-level rate limiting |
| 12  | **Missing `class-validator` on DTOs**     | All GraphQL input types                               | Add validation decorators per user rules            |
| 13  | **Marketplace OTP not sent via SMS**      | `jobs.service.ts:446`                                 | Wire up SMS delivery for job start/completion OTPs  |

### Low

| #   | Issue                                    | Files Affected              | Recommendation                                                                                |
| --- | ---------------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------- |
| 14  | **Deprecated mutation still active**     | `files.resolver.ts`         | Add `@Deprecated()` decorator with runtime warning                                            |
| 15  | **`confirmPayment` sync/async mismatch** | `subscriptions.resolver.ts` | `getPaymentIntent()` is sync but used in async resolver — will break if storage becomes async |

---

## 6. All TODO Comments in Codebase

| File                                                                                                                                                | Line | TODO                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | ---------------------------------------------------------------------------------------- |
| [billing.module.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/billing/billing.module.ts)                       | 4    | `TODO: Implement actual billing export flow`                                             |
| [billing.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/billing/billing.service.ts)                     | 5    | `TODO: Implement: 1. Query VERIFIED sites 2. Generate CSV/Excel 3. Upload 4. Return URL` |
| [billing.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/billing/billing.service.ts)                     | 16   | `TODO: Implement billing export methods`                                                 |
| [billing.resolver.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/billing/billing.resolver.ts)                   | 5    | `TODO: Implement generateBillingExport, billingExports, billingSummary`                  |
| [billing.resolver.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/billing/billing.resolver.ts)                   | 18   | `TODO: Add billing queries and mutations`                                                |
| [storage/index.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/files/storage/index.ts)                           | 28   | `TODO: Implement Google Cloud Storage provider`                                          |
| [sla-scheduler.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/sla/sla-scheduler.service.ts) | 43   | `TODO: Send notification to admin`                                                       |
| [sla-scheduler.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/sla/sla-scheduler.service.ts) | 44   | `TODO: Consider auto-cancelling and refunding`                                           |
| [sla-scheduler.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/sla/sla-scheduler.service.ts) | 82   | `TODO: Send notification to consumer and admin`                                          |
| [sla-scheduler.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/sla/sla-scheduler.service.ts) | 121  | `TODO: Send notification to consumer and admin`                                          |
| [sla-scheduler.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/sla/sla-scheduler.service.ts) | 122  | `TODO: Consider partial refund`                                                          |
| [sites.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/sites/sites.service.ts)                           | 1689 | `TODO: Call NotificationService.sendRegistrationSms()`                                   |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 248  | `TODO: Send SMS to contractor about new job request`                                     |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 304  | `TODO: Send SMS to consumer about job acceptance`                                        |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 349  | `TODO: Trigger refund process`                                                           |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 350  | `TODO: Send SMS to consumer about rejection and refund`                                  |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 391  | `TODO: Trigger refund if payment was made`                                               |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 392  | `TODO: Send SMS to contractor about cancellation`                                        |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 446  | `TODO: Send OTP to consumer via SMS`                                                     |
| [jobs.service.ts](file:///Users/praharsh/Projects/antigravity-workspace/user-service/src/modules/marketplace/jobs/jobs.service.ts)                  | 565  | `TODO: Send SMS notification`                                                            |

---

## 7. Final Summary

| Metric                        | Value                        |
| ----------------------------- | ---------------------------- |
| **Total Backend Modules**     | 22 (registered in AppModule) |
| **Total Resolvers**           | 31                           |
| **Total Frontend Pages**      | 58                           |
| **Total GraphQL Query Files** | 14                           |
| **Database Schema Files**     | 16+                          |
| **Enum Definitions**          | 178 values across 17 enums   |

### Module Status Summary

| Status                       | Count | Modules                                                                                                                                                                                                                                                                                                      |
| ---------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ✅ **Completed**             | 18    | Auth, Users, Consumers, Sites, Meters, MeterSpecs, SiteMeterConnections, Installations, InstallationEvidence, Verifications, Hierarchy, Audit, Scheduler, MeterReadings, RetailOutlets, Contractors, Associations, Marketplace (Services, Pricing, Ratings, Categories, UoMs, ContractorProfiles, Locations) |
| ⚠️ **Partially Implemented** | 4     | Payments (mock Cashfree), Jobs (missing SMS/refund), SLA (missing notifications), Notifications (not wired to marketplace), Files (no GCP)                                                                                                                                                                   |
| ❌ **Not Implemented**       | 1     | Billing (entire module is placeholder)                                                                                                                                                                                                                                                                       |

### Major Gaps (Blocking Production)

1. **Billing module** — zero functionality, needed for utility billing handoff
2. **Cashfree payment integration** — all payment processing is mock data
3. **Test coverage** — effectively zero (1 boilerplate spec + 1 boilerplate e2e)
4. **Notification wiring** — 11 notification call sites have TODO stubs instead of actual service calls

### Readiness Assessment

| Category                                                         | Score    | Rationale                                                                                        |
| ---------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------ |
| Core Infrastructure (Auth, Users, RBAC)                          | 95%      | Fully functional, minor error handling issues                                                    |
| Smart Meter Domain (Sites, Meters, Installations, Verifications) | 90%      | Complete state machines, audit logging, data isolation                                           |
| Marketplace Platform                                             | 65%      | Logic complete but payments mock, SMS notifications unconnected, refunds not triggered           |
| External Integrations                                            | 40%      | SMS/Email infrastructure built but under-connected; Cashfree not integrated; GCP storage missing |
| Testing & Quality                                                | 5%       | No meaningful tests exist                                                                        |
| **Overall Production Readiness**                                 | **~65%** | Core domain is solid; marketplace payments, billing, notifications, and testing are the blockers |
