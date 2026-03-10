# SmartMeter — Project Management System (PMS) Breakdown

> **Project**: SmartMeter — Government-Grade Smart Meter Installation Management System ()
> **Tech Stack**: NestJS 11 · GraphQL (Code-First) · Drizzle ORM · PostgreSQL 17 · Next.js 16 · React 19 · Apollo Client · Tailwind CSS
> **Generated**: 2026-02-20 · Based on full codebase analysis

---

## Epic 0 — Recent Additions (Location Dashboard & Core System Hub)

### Capability 0.0 — LGD Import Resilience & Location Search

- **Data Parsing**: Extended the Excel parser within the `user-service` to dynamically handle explicit LGD naming conventions (e.g., "District Name (In English)").
- **Admin Utilities**: Injected real-time multi-column search capabilities into the Marketplace Locations dashboard for Districts, Sub-Districts, and Villages.
- **Query Optimization**: Implemented `DISTINCT ON` rules to the District and Sub-District location queries within `locations.service.ts` to strictly safeguard the frontend against listing UI duplications.

### Capability 0.1 — Subscription Plan Versioning

- **Versioning Strategy**: Upgraded `SubscriptionPlan` to include an immutable `version` field.
- **Workflow**: Updates to pricing terms retire the active plan and insert a new `v+1` record.
- **Backwards Compatibility**: Contractors retain their original pricing snapshot stored in `SubscriptionPaymentIntent` and their active historical plan logic.
- **UI Element**: "Log" buttons added to display a plan's version history in the admin panel.

### Capability 0.2 — Marketplace Hub Redesign & Pricing Integration

- **Guided Setup Flow**: Restructured the disconnected `/admin/marketplace` grid into a numbered flow: Locations → UOM → Categories → Services.
- **Operational Grid**: Separated daily items (SLA, Jobs, Contractors, Subs, Import Logs) into a lower secondary layout.
- **Integrated Pricing**: Merged `Pricing Configuration` into the `Service Creation Modal`. Creating or editing a service now simultaneously captures and executes mutations for Urban, Semi-Urban, and Rural pricing definitions.

### Capability 0.3 — RO Management & Rating Privacy

- **Validation Resilience**: Reinforced the `createRetailOutlet` flow with robust frontend error handling to capture and display database conflict exceptions (duplicate email/code).
- **Admin Productivity**: Executed a UI cleanup across the Consumers and Sites modules, stripping away non-functional export buttons and redundant action menus to minimize cognitive load.
- **Privacy-First Ratings**: Enhanced the `RatingSummary` query to dynamically aggregate and surface raw review comments while strictly sanitizing the data to remove all rater identifiers.

### Capability 0.5 — Cashfree Payment Lifecycle & Discovery Optimization

- **Payment Gateway Integration**: Transitioned from mock data to real-world payment orchestration via `CashfreeService`. Implemented official SDK integration for Order creation, Payment session management, and Refunds.
- **Asynchronous Settlement**: Instrumented a robust Webhook listener (`CashfreeWebhookController`) with HMAC security to bridge the gap between third-party payment success and internal system state (Subscription upgrades, Job settlements).
- **Discovery Safeguards**: Enhanced the `contractorMarketplaceProfile` schema and `searchContractors` logic to strictly enforce job quotas. Contractors are now automatically retired from search results once their PREMIUM tier limits are exhausted.
- **Data Integrity**: Hardened the contractor onboarding flow with proactive conflict validation (duplicate email/phone checks) and automated organizational hierarchy linking for RO-managed contractors.
- **Tiered Catalog Pricing**: Upgraded the `ServicesService` to fetch and return the complete Rural/Urban/Semi-Urban pricing matrix for the marketplace catalog in a single aliased database hit.

---

## Epic 1 — Authentication & Authorization

### Capability 1.1 — Identity & Credential Management

#### Feature 1.1.1 — Multi-Strategy Login

> **User Story**: As a system user, I want to authenticate using the method appropriate to my role (email/password or phone/OTP) so that I can securely access role-specific functionality.

| Layer        | Tasks                                                                                                                                               |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement email/password authentication flow with role-type (`loginType`) validation against the `users` table via `AuthService.validateUser`       |
|              | Implement OTP-based login for all roles via `requestLoginOtp` / `verifyLoginOtp` mutations using Fast2SMS integration                               |
|              | Implement general-purpose OTP verification (password reset, account verification) via `requestOtp` / `verifyOtp` mutations, separate from login OTP |
|              | Support configurable OTP mode (`DUMMY` vs `ACTUAL`) via `OTP_MODE` environment variable for development/staging                                     |
| **Frontend** | Build login page with dynamic form switching based on selected role (email/password vs phone/OTP)                                                   |
|              | Implement OTP input screen with resend timer and auto-submit on 6-digit entry                                                                       |
|              | Handle login error states with user-friendly messaging for invalid credentials and throttled accounts                                               |
| **Security** | Implement login attempt throttling via `auth_throttle` table with time-based lockout for `PASSWORD_LOGIN` and `OTP_VERIFY` types                    |
|              | Hash passwords using `bcryptjs` before storage; never expose raw credentials                                                                        |
|              | Validate `loginType` matches the user's actual role to prevent cross-role authentication bypass                                                     |
| **Testing**  | Unit test `AuthService.validateUser` with correct/incorrect credentials per role type                                                               |
|              | Unit test OTP generation, storage, expiry, and verification logic                                                                                   |
|              | E2E test full login flow for each role (SUPER_ADMIN, ADMIN, RO, CONTRACTOR, CONSUMER)                                                               |
|              | E2E test throttling: verify lockout after consecutive failed attempts                                                                               |

---

#### Feature 1.1.2 — JWT Token Lifecycle Management

> **User Story**: As an authenticated user, I want my session to persist securely and refresh transparently so that I don't need to log in repeatedly.

| Layer        | Tasks                                                                                                                 |
| ------------ | --------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement JWT access token generation with role, userId, and email claims via `@nestjs/jwt`                           |
|              | Implement refresh token generation, storage in `refresh_tokens` table, and rotation via `refreshAccessToken` mutation |
|              | Implement secure logout by invalidating stored refresh tokens via `AuthService.logout`                                |
|              | Implement `me` query to retrieve current user profile from JWT claims with `GqlAuthGuard` protection                  |
| **Frontend** | Store JWT tokens securely and implement transparent token refresh on 401 responses via Apollo Client link middleware  |
|              | Implement automatic redirect to login on token expiry when refresh also fails                                         |
| **Security** | Enforce JWT expiry (`JWT_EXPIRES_IN`) via environment configuration                                                   |
|              | Implement refresh token rotation — issue new refresh token on each refresh and invalidate the old one                 |
| **Testing**  | Unit test JWT generation with correct claims and expiry                                                               |
|              | E2E test refresh token flow: valid refresh → new tokens, expired refresh → 401                                        |
|              | E2E test logout invalidates refresh token and subsequent refresh attempts fail                                        |

---

### Capability 1.2 — Role-Based Access Control (RBAC)

#### Feature 1.2.1 — GraphQL Guard System

> **User Story**: As an administrator, I want each API endpoint to enforce role-based permissions so that users can only access functionality appropriate to their role in the hierarchy.

| Layer        | Tasks                                                                                                               |
| ------------ | ------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `GqlAuthGuard` extending Passport JWT strategy for GraphQL context extraction                             |
|              | Implement `RolesGuard` that reads `@Roles()` decorator metadata and validates against the JWT-decoded user role     |
|              | Implement `@CurrentUser()` decorator for extracting authenticated user context from GraphQL execution context       |
|              | Implement `@Roles()` decorator for declarative role requirement specification on resolvers                          |
|              | Enforce role hierarchy (SUPER_ADMIN → ADMIN → RO → CONTRACTOR/CONSUMER/SUB_USER) in all creation mutations          |
| **Security** | Ensure every resolver mutation/query has explicit `@UseGuards(GqlAuthGuard, RolesGuard)` + `@Roles()` decorators    |
|              | Implement data isolation — each role only sees data they created or own (enforced at service layer query filtering) |
| **Testing**  | Unit test `RolesGuard` with various role combinations and edge cases                                                |
|              | E2E test that unauthorized roles receive proper 403/unauthorized responses                                          |
|              | E2E test data isolation: RO A cannot see data belonging to RO B                                                     |

---

## Epic 2 — User & Role Management

### Capability 2.1 — User Administration

#### Feature 2.1.1 — Hierarchical User Creation

> **User Story**: As an admin user, I want to create users at the appropriate level of the role hierarchy so that organizational structure is maintained.

| Layer        | Tasks                                                                                                              |
| ------------ | ------------------------------------------------------------------------------------------------------------------ |
| **Backend**  | Implement `createAdmin` mutation (SUPER_ADMIN only) — creates ADMIN users with email/password                      |
|              | Implement `createRetailOutlet` mutation (ADMIN only) — creates RO users with email/password                        |
|              | Implement `createConsumer` mutation (RO/SUB_USER) — creates consumer users with phone, linked to consumer profile  |
|              | Implement `createContractor` mutation (RO) — creates contractor users with company details and license information |
|              | Implement `createSubUser` mutation (RO) — creates sub-user accounts with limited RO permissions                    |
|              | Implement generic `createUser` mutation with dynamic role enforcement based on creator's role                      |
| **Frontend** | Build admin management page at `(admin)/admin/admins` for SUPER_ADMIN to create/manage admin accounts              |
|              | Build RO management page at `(admin)/admin/ros` for ADMIN to create/manage retail outlets                          |
|              | Build contractor management interface at `(ro)/ro/contractors` for RO to onboard contractors                       |
|              | Build staff management interface at `(ro)/ro/staff` for RO to manage sub-users                                     |
| **Security** | Validate that creating user's role is strictly above the target role in the hierarchy                              |
|              | Hash passwords before storage during user creation                                                                 |
| **Testing**  | Unit test role hierarchy enforcement — verify ADMIN cannot create SUPER_ADMIN                                      |
|              | E2E test complete user creation flow for each level of the hierarchy                                               |

---

#### Feature 2.1.2 — User Lifecycle Management

> **User Story**: As an administrator, I want to activate, deactivate, and delete user accounts so that I can control system access.

| Layer        | Tasks                                                                                                     |
| ------------ | --------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `deactivateUser` mutation with role hierarchy validation (can only deactivate lower-role users) |
|              | Implement `activateUser` mutation to re-enable previously deactivated accounts                            |
|              | Implement `deleteUser` mutation with cascading cleanup of related records                                 |
|              | Implement `users` query with optional role filter and data isolation per current user's scope             |
|              | Implement `user` query for single user retrieval (SUPER_ADMIN/ADMIN only)                                 |
| **Frontend** | Build user listing with filtering, search, and status indicators                                          |
|              | Implement activate/deactivate toggle with confirmation dialogs                                            |
| **Security** | Prevent self-deactivation/deletion                                                                        |
|              | Validate hierarchy — users can only manage accounts at lower permission levels                            |
| **Testing**  | Unit test lifecycle transitions and permission enforcement                                                |
|              | E2E test activate/deactivate/delete flows with role verification                                          |

---

## Epic 3 — Consumer & Site Lifecycle

### Capability 3.1 — Consumer Management

#### Feature 3.1.1 — Consumer Profile Management

> **User Story**: As a retail outlet operator, I want to create and manage consumer profiles with complete address and hierarchy information so that sites can be correctly registered.

| Layer        | Tasks                                                                                                                                     |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createConsumer` mutation with consumer record creation in `consumers` table, linked to hierarchy (circle/division/subdivision) |
|              | Implement `updateConsumer` mutation for profile field modifications with audit logging                                                    |
|              | Implement `consumer` / `consumerByConsumerId` queries for record retrieval by internal ID or consumer ID                                  |
|              | Implement `myConsumer` query for consumer-role users to access their own profile (by userId or phone)                                     |
|              | Implement `consumers` query with `ConsumerFiltersInput` (pagination, search, status filter) and data isolation per current user's scope   |
|              | Implement `circleName` / `divisionName` / `subdivisionName` ResolveField resolvers for hierarchy name lookups                             |
| **Frontend** | Build consumer listing page at `(ro)/ro/consumers` with paginated table, search, and status filters                                       |
|              | Build consumer detail page at `(ro)/ro/consumers/[id]` with profile, site, and installation information                                   |
|              | Build consumer registration form with address fields and hierarchy selectors                                                              |
|              | Build consumer self-service dashboard at `(consumer)/consumer` with profile and site status                                               |
| **Security** | Enforce RBAC: ADMIN, RO, SUB_USER can create consumers; consumers can only view their own profile                                         |
| **Testing**  | Unit test consumer creation with hierarchy linkage                                                                                        |
|              | E2E test consumer CRUD flow from RO perspective                                                                                           |
|              | E2E test consumer self-service profile access                                                                                             |

---

#### Feature 3.1.2 — Consumer Lifecycle State Management

> **User Story**: As an administrator, I want to manage consumer account states (activate, suspend, deactivate) so that I can control service access based on compliance.

| Layer        | Tasks                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `activateConsumer` mutation — transitions from REGISTERED/SUSPENDED → ACTIVE (ADMIN only)     |
|              | Implement `suspendConsumer` mutation — transitions to SUSPENDED state with reason tracking (ADMIN only) |
|              | Implement `deactivateConsumer` mutation — transitions to DEACTIVATED state (ADMIN only)                 |
|              | Enforce `consumer_state` enum lifecycle: `REGISTERED → ACTIVE → SUSPENDED → DEACTIVATED`                |
| **Frontend** | Build consumer status management UI with state indicators and action buttons                            |
|              | Show state change confirmation dialogs with reason input for suspension                                 |
| **Security** | Restrict lifecycle mutations to ADMIN role only                                                         |
|              | Audit log every state change with actor, timestamp, and reason                                          |
| **Testing**  | Unit test state transition validation — reject illegal state changes                                    |
|              | E2E test complete consumer lifecycle from registration to deactivation                                  |

---

### Capability 3.2 — Site (Connection) Management

#### Feature 3.2.1 — Site Registration & Configuration

> **User Story**: As a retail outlet operator, I want to create and configure consumer sites with address, location, and technical requirements so that meters can be properly assigned and installed.

| Layer        | Tasks                                                                                                                     |
| ------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createSite` mutation with consumer linkage, address, and hierarchy assignment                                  |
|              | Implement `updateSiteAddress` / `updateSiteLocation` / `updateSiteRequirements` mutations for granular site field updates |
|              | Implement `updateSiteROFields` mutation for RO-specific field management (tariff, requirements, schedule)                 |
|              | Implement `site` / `siteBySiteId` / `sitesByConsumer` / `mySites` queries for various access patterns                     |
|              | Implement `sites` query with `SitesFilterInput` and `SitesPaginationInput` for filtered, paginated listing                |
|              | Implement `siteStatistics` query for dashboard summary metrics (counts by state)                                          |
|              | Implement `sitePhotoUrl` ResolveField for generating signed URLs from Azure Blob storage                                  |
|              | Implement `contractor` / `subdivisionName` / `divisionName` / `circleName` ResolveFields for related entity resolution    |
| **Frontend** | Build site listing at `(ro)/ro/site-review` with state-based filtering and pagination                                     |
|              | Build consumer address update page at `(consumer)/consumer/address`                                                       |
|              | Build consumer site photo upload page at `(consumer)/consumer/photo` with camera integration                              |
|              | Build consumer status tracking page at `(consumer)/consumer/status`                                                       |
| **Security** | Enforce RBAC per mutation: RO/SUB_USER for creation, consumer for self-service updates                                    |
|              | Validate `siteId` uniqueness per consumer                                                                                 |
| **Testing**  | Unit test site creation with all required field validations                                                               |
|              | E2E test site creation → update → photo upload → verification flow                                                        |

---

#### Feature 3.2.2 — Site State Machine Lifecycle

> **User Story**: As a system administrator, I want sites to follow a strict state machine lifecycle so that no installation step is skipped and full traceability is maintained.

| Layer        | Tasks                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Backend**  | Implement site state machine enforcing transitions: `CREATED → SITE_PENDING → SITE_VERIFIED → METER_ASSIGNED → INSTALLATION_SCHEDULED → INSTALLATION_IN_PROGRESS → INSTALLED → VERIFICATION_PENDING → VERIFIED → BILLED` |
|              | Implement `FAILED` state accessible from any stage with reason tracking                                                                                                                                                  |
|              | Lock tariff code at `SITE_VERIFIED` stage — immutable after verification                                                                                                                                                 |
|              | Implement `uploadSitePhoto` / `uploadSitePhotoUrl` mutations for consumer photo submission                                                                                                                               |
|              | Implement `activateConsumerSite` mutation for the full activation workflow                                                                                                                                               |
|              | Implement `siteReadinessStatus` tracking: `PENDING → PHOTO_UPLOADED → LOCATION_VERIFIED → APPROVED → REJECTED`                                                                                                           |
|              | Audit log every state transition with actor, role, old state, new state, and timestamp                                                                                                                                   |
| **Frontend** | Build site verification workflow at `(ro)/ro/site-review/[id]` with approval/rejection actions                                                                                                                           |
|              | Display site state progression UI with visual state indicators                                                                                                                                                           |
| **Security** | Block illegal state transitions at the service layer                                                                                                                                                                     |
|              | Enforce role-appropriate state change permissions (e.g., only RO can verify sites)                                                                                                                                       |
| **Testing**  | Unit test every valid state transition and confirm illegal transitions throw errors                                                                                                                                      |
|              | E2E test complete site lifecycle from CREATED to BILLED                                                                                                                                                                  |

---

## Epic 4 — Smart Meter Inventory Management

### Capability 4.1 — Meter Registration & Tracking

#### Feature 4.1.1 — Meter Inventory Management

> **User Story**: As a retail outlet operator, I want to register, track, and manage smart meter inventory so that meters are accounted for throughout their lifecycle.

| Layer        | Tasks                                                                                                                      |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createMeter` mutation with serial number, QR code, phase type, voltage, network type, and meter spec linkage    |
|              | Implement `meter` / `meterBySerialNumber` / `meterByQrCode` queries for flexible meter lookup                              |
|              | Implement `meters` query with `MeterFiltersInput` (pagination, status, type) and data isolation per role                   |
|              | Implement `inventorySummary` query returning counts by meter state (available, assigned, installed, active, failed) per RO |
|              | Implement meter state machine: `AVAILABLE → RESERVED → ASSIGNED → INSTALLED → ACTIVE → FAILED → DECOMMISSIONED`            |
| **Frontend** | Build inventory dashboard at `(ro)/ro/inventory` with summary cards and detailed meter listing                             |
|              | Build meter lookup interfaces using serial number and QR code scanning                                                     |
|              | Implement QR code scanner using webcam (`jsqr` library) for meter identification                                           |
| **Security** | Enforce RBAC: ADMIN/RO/SUB_USER can create meters; data isolation per RO                                                   |
| **Testing**  | Unit test meter CRUD operations and state machine transitions                                                              |
|              | E2E test meter creation → assignment → installation → verification lifecycle                                               |

---

#### Feature 4.1.2 — Meter Assignment & Lifecycle Operations

> **User Story**: As a retail outlet operator, I want to assign meters to consumer sites, track installations, and handle failures so that every meter is accounted for.

| Layer        | Tasks                                                                                                                                                       |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `assignMeter` mutation — links meter to consumer, validates meter compatibility against site requirements, creates `site_meter_connection` record |
|              | Implement `markMeterInstalled` mutation (CONTRACTOR only) — records initial reading, updates meter state to INSTALLED                                       |
|              | Implement `verifyMeterInstallation` mutation (RO/ADMIN) — approves installation, transitions meter to ACTIVE                                                |
|              | Implement `markMeterFailed` mutation — handles installation failures, transitions meter to FAILED                                                           |
|              | Implement `releaseMeter` mutation — removes meter-site assignment, creates new connection record (preserves history)                                        |
| **Frontend** | Build meter assignment page at `(ro)/ro/assign` with consumer/site selector and meter picker                                                                |
|              | Build assignment workflow with QR scan → consumer selection → meter validation → confirmation                                                               |
| **Security** | Validate meter compatibility (phase, voltage) against site requirements before assignment                                                                   |
|              | Ensure only one active assignment per meter at any time                                                                                                     |
|              | Preserve full assignment history — never overwrite `site_meter_connections`                                                                                 |
| **Testing**  | Unit test assignment validation (compatible vs incompatible meters)                                                                                         |
|              | Unit test state transitions across the complete meter lifecycle                                                                                             |
|              | E2E test assign → install → verify and assign → install → fail → release → reassign flows                                                                   |

---

### Capability 4.2 — Site-Meter Connections

#### Feature 4.2.1 — Connection Domain Entity Management

> **User Story**: As a system, I want site-meter connections to be a first-class domain entity with full history tracking so that meter replacements and audit trails are preserved.

| Layer        | Tasks                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------ |
| **Backend**  | Implement `site_meter_connections` as a domain entity (not a simple join) with assignment lifecycle tracking |
|              | Implement connection status tracking: `ACTIVE → REPLACED → REMOVED → ROLLED_BACK`                            |
|              | Implement tariff snapshot persistence on connection creation (immutable after verification)                  |
|              | Implement meter replacement flow: create new connection, mark old as REPLACED, preserve full history         |
|              | Implement rollback capability for incorrect assignments                                                      |
| **Security** | Prevent direct modification of historical connection records                                                 |
|              | Audit log every connection state change                                                                      |
| **Testing**  | Unit test connection creation, replacement, and rollback flows                                               |
|              | E2E test full meter replacement scenario with history preservation                                           |

---

## Epic 5 — Installation Workflow

### Capability 5.1 — Installation Tracking

#### Feature 5.1.1 — Installation Management

> **User Story**: As a contractor, I want to receive, track, and complete installation assignments so that I can efficiently manage my fieldwork.

| Layer        | Tasks                                                                                                                                                                |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createInstallation` mutation with site, contractor, and meter linkage                                                                                     |
|              | Implement `createMultipleInstallations` mutation for bulk installation scheduling (one per site-meter connection)                                                    |
|              | Implement `updateInstallationStatus` mutation with status transitions: `PENDING → SCHEDULED → IN_PROGRESS → COMPLETED → VERIFIED` or `FAILED → CANCELLED → REJECTED` |
|              | Implement `installations` query with filters (status, date, contractor) and data isolation                                                                           |
|              | Implement `myInstallations` query for contractors to see only their assigned installations                                                                           |
|              | Implement `installationBySite` / `installationStatusByConsumer` queries for cross-entity lookups                                                                     |
|              | Implement `site` / `meter` / `contractor` / `consumer` ResolveField resolvers for nested entity loading                                                              |
| **Frontend** | Build contractor installation list at `(contractor)/contractor/installations` with status filtering                                                                  |
|              | Build installation details page at `(contractor)/contractor/install` with evidence capture workflow                                                                  |
|              | Build contractor job pickup flow at `(contractor)/contractor/pickup`                                                                                                 |
|              | Build pre-installation checklist at `(contractor)/contractor/pre-installation`                                                                                       |
|              | Build RO installation listing at `(ro)/ro/installations` for monitoring                                                                                              |
|              | Build scheduling interface at `(ro)/ro/schedule` for appointment management                                                                                          |
| **Security** | Enforce that contractors can only update installations assigned to them                                                                                              |
|              | Validate status transitions — reject illegal jumps (e.g., PENDING → COMPLETED without IN_PROGRESS)                                                                   |
| **Testing**  | Unit test installation creation and all status transitions                                                                                                           |
|              | E2E test: create → schedule → assign contractor → start → complete → verify flow                                                                                     |

---

#### Feature 5.1.2 — Installation Evidence Collection

> **User Story**: As a contractor, I want to capture and upload geo-tagged photos, seal verification, and meter readings as installation evidence so that installations can be verified remotely.

| Layer        | Tasks                                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------------------ |
| **Backend**  | Implement `installation_evidence` module with evidence type tracking (photos, GPS, readings)           |
|              | Link evidence records to installations with `installation_evidence` resolver                           |
|              | Support multiple evidence types: `SITE_PHOTO`, `INSTALLATION_PHOTO`, `OCR_IMAGE`, `VERIFICATION_PHOTO` |
| **Frontend** | Build evidence capture flow with camera integration, GPS tagging, and seal number input                |
|              | Implement geo-tagged photo capture using browser APIs                                                  |
| **Security** | Validate GPS coordinates are within reasonable range of the site location                              |
|              | Ensure evidence is immutable after submission                                                          |
| **Testing**  | E2E test evidence upload and retrieval with verification workflow                                      |

---

### Capability 5.2 — Installation Verification

#### Feature 5.2.1 — Verification & Approval Workflow

> **User Story**: As a retail outlet operator, I want to review installation evidence and approve or reject installations so that only properly completed work is accepted.

| Layer        | Tasks                                                                                                           |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createVerification` mutation with evidence review (photos, seal, GPS, reading validation)            |
|              | Implement `reviewVerification` mutation for ADMIN/RO approval or rejection with notes                           |
|              | Enforce verification status enum: `PENDING → APPROVED → REJECTED`                                               |
|              | Implement `verifications` query with pagination and filters                                                     |
| **Frontend** | Build verification review page at `(ro)/ro/verify` with evidence gallery, approval/rejection buttons, and notes |
|              | Build site-review verification page at `(ro)/ro/verify/[id]` with detailed evidence comparison                  |
| **Security** | Only ADMIN/RO/SUB_USER can create verifications; only ADMIN/RO can review                                       |
|              | Log verification decisions in audit trail                                                                       |
| **Testing**  | Unit test verification creation and review with status transitions                                              |
|              | E2E test create → review → approve and create → review → reject flows                                           |

---

## Epic 6 — Marketplace Platform

### Capability 6.1 — Service Catalog & Configuration

#### Feature 6.1.1 — Service Catalog Management

> **User Story**: As an administrator, I want to manage a catalog of marketplace services with categories and area-specific pricing so that consumers can browse and order services.

| Layer        | Tasks                                                                                                                   |
| ------------ | ----------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createMarketplaceService` / `updateMarketplaceService` mutations with category, description, and UOM linkage |
|              | Implement `deactivateMarketplaceService` / `activateMarketplaceService` mutations for lifecycle control                 |
|              | Implement `marketplaceServices` query with `ServicesFilterInput` pagination and filtering                               |
|              | Implement `marketplaceServicesWithPrices` query — returns services with current prices for a given `AreaType`           |
|              | Implement `marketplaceServicesByCategory` query — groups services by category for consumer-facing UI                    |
|              | Implement service categories module with CRUD for organizing services                                                   |
|              | Implement UoM (Units of Measurement) module for service quantity management                                             |
| **Frontend** | Build admin marketplace management at `(admin)/admin/marketplace` with service CRUD                                     |
|              | Build consumer marketplace browsing at `(consumer)/consumer/marketplace` with category navigation                       |
| **Security** | Admin-only mutations for catalog management; authenticated access for browsing                                          |
| **Testing**  | Unit test service CRUD with category and pricing linkage                                                                |
|              | E2E test catalog management workflow and consumer browsing queries                                                      |

---

#### Feature 6.1.2 — Pricing & SLA Configuration

> **User Story**: As an administrator, I want to set service prices and SLA targets per area type so that marketplace operations meet quality standards.

| Layer        | Tasks                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement pricing module with area-type-specific pricing (URBAN, RURAL, etc.)                            |
|              | Implement `setMarketplaceSla` mutation for setting response/completion time SLAs per service + area type |
|              | Implement `bulkSetMarketplaceSlas` mutation for bulk SLA configuration                                   |
|              | Implement `marketplaceSlaMatrix` query to retrieve SLA configuration grid per service                    |
|              | Implement `marketplaceSlaBreaches` query to track SLA violations with filtering                          |
| **Frontend** | Build pricing management interface within admin marketplace section                                      |
|              | Build SLA matrix configuration UI with service × area type grid                                          |
| **Security** | Admin-only access for pricing and SLA configuration                                                      |
| **Testing**  | Unit test pricing calculations and SLA breach detection logic                                            |

---

### Capability 6.2 — Job Lifecycle

#### Feature 6.2.1 — Job Creation & Payment

> **User Story**: As a consumer, I want to create service jobs and initiate payment so that I can request marketplace services.

| Layer        | Tasks                                                                                                                                                      |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `createMarketplaceJob` mutation — consumer creates job with service, location, and schedule details; returns `CreateJobResult` with payment info |
|              | Implement `simulatePaymentSuccess` mutation (DEV only) — placeholder for Cashfree payment gateway integration                                              |
|              | Implement `initiateMarketplacePayment` mutation — creates payment intent for job via `PaymentsService`                                                     |
|              | Implement `myMarketplaceJobs` query for consumers to view their job history with filters                                                                   |
| **Frontend** | Build job creation flow at `(consumer)/consumer/marketplace/[serviceId]` with location, schedule, and payment                                              |
|              | Build consumer job history listing at `(consumer)/consumer/marketplace`                                                                                    |
| **Security** | Validate consumer identity and link jobs to correct consumer profile                                                                                       |
|              | Implement payment validation before job confirmation                                                                                                       |
| **Testing**  | Unit test job creation with payment intent generation                                                                                                      |
|              | E2E test complete job placement flow from service selection to payment                                                                                     |

---

#### Feature 6.2.2 — Job Acceptance & Execution

> **User Story**: As a contractor, I want to view, accept, and execute assigned jobs with OTP verification so that I can manage my service work.

| Layer        | Tasks                                                                                                                |
| ------------ | -------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `acceptMarketplaceJob` mutation — contractor accepts job with scheduling details                           |
|              | Implement `rejectMarketplaceJob` mutation — contractor rejects with reason                                           |
|              | Implement `generateMarketplaceOtp` mutation — generates OTP for job start/completion verification (sent to consumer) |
|              | Implement `verifyMarketplaceOtp` mutation — contractor verifies OTP to confirm physical presence at start/completion |
|              | Implement `cancelMarketplaceJob` mutation — consumer cancels job with refund eligibility check                       |
|              | Implement `myContractorMarketplaceJobs` query for contractors to list their assigned jobs                            |
|              | Implement `marketplaceJobs` query (ADMIN) for system-wide job monitoring                                             |
| **Frontend** | Build contractor job dashboard at `(contractor)/contractor/marketplace` with accept/reject actions                   |
|              | Build job execution flow at `(contractor)/contractor/job` with OTP verification screens                              |
|              | Build job issue reporting at `(contractor)/contractor/issues`                                                        |
|              | Build admin job monitoring at `(admin)/admin/marketplace`                                                            |
| **Security** | Validate contractor subscription status before job acceptance                                                        |
|              | OTP verification ensures physical presence at consumer location                                                      |
|              | Prevent contractors from accepting jobs outside their service area                                                   |
| **Testing**  | E2E test: create → pay → accept → start (OTP) → complete (OTP) flow                                                  |
|              | E2E test: create → pay → reject → reassign flow                                                                      |
|              | E2E test: create → pay → cancel → refund flow                                                                        |

---

### Capability 6.3 — Contractor Marketplace Profiles

#### Feature 6.3.1 — Contractor Profiles & Subscription Management

> **User Story**: As a contractor, I want to manage my marketplace profile and subscription tier so that I can receive job assignments.

| Layer        | Tasks                                                                                                          |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement contractor profile module with service area, specializations, and availability                       |
|              | Implement `subscriptionPlans` query returning available subscription tiers                                     |
|              | Implement `mySubscriptionStatus` query for contractor to check active subscription                             |
|              | Implement `initiateSubscriptionUpgrade` mutation — creates payment intent for tier upgrade                     |
|              | Implement `confirmSubscriptionPayment` mutation — confirms payment and activates subscription                  |
|              | Implement `adminSetContractorSubscription` mutation — admin-managed subscription override                      |
|              | Implement subscription history tracking with `mySubscriptionHistory` / `contractorSubscriptionHistory` queries |
| **Frontend** | Build contractor subscription management UI at `(contractor)/contractor/marketplace/subscription`              |
|              | Build admin contractor subscription management at `(admin)/admin/marketplace/contractors`                      |
| **Security** | Verify subscription ownership — contractors can only upgrade their own subscription                            |
|              | Validate payment intent belongs to the requesting contractor                                                   |
| **Testing**  | Unit test subscription plan retrieval and tier upgrade flow                                                    |
|              | E2E test: initiate upgrade → payment → confirmation → active subscription                                      |

---

#### Feature 6.3.2 — Ratings & Reviews

> **User Story**: As a consumer, I want to rate and review contractors after job completion so that service quality is tracked.

| Layer        | Tasks                                                                                            |
| ------------ | ------------------------------------------------------------------------------------------------ |
| **Backend**  | Implement `createMarketplaceRating` mutation with star rating and review text for completed jobs |
|              | Implement `hasRated` boolean resolution across job arrays using `inArray` queries                |
|              | Enforce state machine transition: Consumer Rating forces job from `COMPLETED` → `CLOSED`         |
|              | Implement `marketplaceRatings` query (Admin) with pagination and filters                         |
|              | Implement `marketplaceRatingSummary` query — aggregated rating stats per contractor              |
|              | Implement `marketplaceContractorReviews` query — paginated reviews for a contractor              |
| **Frontend** | Build rating submission UI after job completion in consumer marketplace section                  |
|              | Build contractor reviews page showing aggregated ratings and individual reviews                  |
| **Security** | Ensure only the consumer from a completed job can submit a rating                                |
|              | Prevent duplicate ratings for the same job                                                       |
| **Testing**  | Unit test rating creation, aggregation, and summary calculations                                 |

---

#### Feature 6.3.3 — Service Location Management

> **User Story**: As an administrator, I want to manage service coverage areas so that marketplace services are available in the correct geographic regions.

| Layer       | Tasks                                                                                   |
| ----------- | --------------------------------------------------------------------------------------- |
| **Backend** | Implement service locations module with area type configuration and geographic coverage |
|             | Support geo-spatial queries using `@turf/turf` for location-based contractor matching   |
| **Testing** | Unit test location-based service availability queries                                   |

---

## Epic 7 — Billing & Payments

### Capability 7.1 — Marketplace Payments

#### Feature 7.1.1 — Payment Processing & Refunds

> **User Story**: As a consumer, I want to make secure payments for marketplace services and receive refunds for cancelled jobs so that transactions are handled fairly.

| Layer        | Tasks                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------- |
| **Backend**  | Implement `initiateMarketplacePayment` mutation — creates payment intent linked to job   |
|              | Implement `marketplacePayments` query (Admin) — paginated payment history with filters   |
|              | Implement `marketplacePayment` / `marketplacePaymentByJob` queries for payment retrieval |
|              | Implement `marketplaceRefunds` query (Admin) — paginated refund history                  |
|              | Implement `marketplaceRefund` / `marketplaceRefundByJob` queries for refund retrieval    |
|              | _(Note: Webhook endpoints for payment gateway callbacks are REST, not GraphQL)_          |
| **Frontend** | Build payment flow within job creation process                                           |
|              | Build admin payment/refund monitoring dashboard                                          |
| **Security** | Validate payment amounts against service pricing                                         |
|              | Implement idempotent payment processing to prevent double charges                        |
| **Testing**  | Unit test payment initiation and status tracking                                         |
|              | E2E test payment → confirm → refund workflow                                             |

---

### Capability 7.2 — Utility Billing (Smart Meter)

#### Feature 7.2.1 — Billing Export & Handoff

> **User Story**: As an administrator, I want to generate billing exports from verified installations so that the utility company can begin metering consumers.

| Layer        | Tasks                                                                                                         |
| ------------ | ------------------------------------------------------------------------------------------------------------- |
| **Backend**  | **⚠️ TODO**: Implement `generateBillingExport` mutation — creates billing handoff records from verified sites |
|              | **⚠️ TODO**: Implement `billingExports` query — retrieves billing export history                              |
|              | **⚠️ TODO**: Implement `billingSummary` query — summary statistics of billing-ready sites                     |
|              | _(Note: `BillingResolver` is currently a placeholder with no implemented queries/mutations)_                  |
| **Frontend** | Build billing dashboard with export generation and history                                                    |
| **Testing**  | Unit test billing export generation with data validation                                                      |

---

## Epic 8 — Notifications & Communications

### Capability 8.1 — Multi-Channel Notifications

#### Feature 8.1.1 — SMS & Email Notification System

> **User Story**: As a system operator, I want automated notifications sent to consumers at key lifecycle events so that they are informed of their installation progress.

| Layer              | Tasks                                                                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**        | Implement `sendWelcomeMessage` mutation — sends welcome SMS/email to newly registered consumers with site ID                                  |
|                    | Implement `sendAppointmentNotification` mutation — notifies consumers of scheduled installation appointments                                  |
|                    | Implement `sendInstallationCompleteNotification` mutation — informs consumers of completed installation with meter serial and initial reading |
|                    | Implement Fast2SMS integration with DLT template compliance for SMS delivery                                                                  |
|                    | Implement AWS SES integration for email delivery                                                                                              |
|                    | Support email as optional fallback (`consumerEmail` is nullable in all notification mutations)                                                |
| **Infrastructure** | Configure `SMS_GATEWAY_ENABLED` and `EMAIL_ENABLED` environment toggles                                                                       |
|                    | Register DLT templates for regulatory SMS compliance                                                                                          |
| **Security**       | Restrict notification sending to ADMIN/RO/SUB_USER (and CONTRACTOR for installation complete)                                                 |
| **Testing**        | Unit test notification service with mocked SMS/email providers                                                                                |
|                    | E2E test notification delivery integration                                                                                                    |

---

## Epic 9 — Platform Operations & Infrastructure

### Capability 9.1 — File Management

#### Feature 9.1.1 — Multi-Provider File Upload & Storage

> **User Story**: As a system user, I want to upload files (photos, documents, Excel imports) through a unified interface so that evidence and data imports are stored securely.

| Layer              | Tasks                                                                                                                 |
| ------------------ | --------------------------------------------------------------------------------------------------------------------- |
| **Backend**        | Implement `uploadBase64File` mutation for GraphQL-based file upload (deprecated in favor of REST)                     |
|                    | Implement REST `POST /files/upload/:mode/:context` endpoint for performant file uploads                               |
|                    | Implement `file` / `filesByEntity` queries for retrieving uploaded files by ID or entity linkage                      |
|                    | Support multi-provider storage: Local, Azure Blob, GCP Storage via `STORAGE_MODE` config                              |
|                    | Implement signed URL generation for secure private file access                                                        |
|                    | Support file types: `SITE_PHOTO`, `INSTALLATION_PHOTO`, `OCR_IMAGE`, `DOCUMENT`, `EXCEL_IMPORT`, `VERIFICATION_PHOTO` |
| **Infrastructure** | Configure Azure Blob Storage with `@azure/storage-blob` and `@azure/identity`                                         |
|                    | Configure public vs private container routing                                                                         |
| **Security**       | Validate file types and sizes before storage                                                                          |
|                    | Generate time-limited signed URLs for private file access                                                             |
|                    | Enforce RBAC: all roles can access files; consumers limited to their own entities                                     |
| **Testing**        | Unit test file upload, storage, and retrieval for each provider                                                       |
|                    | E2E test upload → retrieve → signed URL generation flow                                                               |

---

### Capability 9.2 — Bulk Data Import

#### Feature 9.2.1 — Excel Import System

> **User Story**: As an administrator, I want to bulk import consumers, meters, and contractors from Excel files so that large-scale onboarding is efficient.

| Layer        | Tasks                                                                                                                                        |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `importConsumers` mutation — parses Excel, validates rows, creates consumer + site + user records in batch with SHA-256 hash dedup |
|              | Implement `importMeters` mutation — parses Excel, validates meter specs, creates meter records in batch                                      |
|              | Implement `importContractors` mutation — parses Excel, creates contractor user + profile records with password hashing                       |
|              | Implement `importLocations` / `importStates` mutations for geographic data imports                                                           |
|              | Implement `importLogs` query for tracking import history with status (`PENDING → PROCESSING → COMPLETED → FAILED`)                           |
|              | Implement batch hierarchy resolution for consumer rows (circle/division/subdivision by code)                                                 |
|              | Implement row-level error collection and reporting                                                                                           |
| **Frontend** | Build import interface at `(admin)/admin/upload` with file upload, progress indication, and error reporting                                  |
| **Security** | SHA-256 hash check prevents duplicate imports of the same file                                                                               |
|              | Validate all input data before database writes                                                                                               |
| **Testing**  | Unit test Excel parsing with valid and invalid data                                                                                          |
|              | Unit test deduplication via SHA-256 hash                                                                                                     |
|              | E2E test import with mixed valid/invalid rows and verify error reporting                                                                     |

---

### Capability 9.3 — Audit & Compliance

#### Feature 9.3.1 — Comprehensive Audit Logging

> **User Story**: As an administrator, I want a complete audit trail of all system operations so that every action is traceable for compliance and dispute resolution.

| Layer        | Tasks                                                                                                                                                                                                          |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `auditLogs` query with pagination, entity type, and entity ID filtering (ADMIN only)                                                                                                                 |
|              | Record audit entries for all CRUD operations: `CREATE`, `UPDATE`, `DELETE`, `STATE_CHANGE`, `ASSIGN`, `UNASSIGN`, `VERIFY`, `REJECT`, `ACTIVATE`, `LOGIN`, `LOGOUT`, `UPLOAD`, `IMPORT`, `REPLACE`, `ROLLBACK` |
|              | Store actor ID, role, entity type, entity ID, action, old value, new value, and timestamp                                                                                                                      |
| **Frontend** | Build admin analytics dashboard at `(admin)/admin/analytics` with audit log viewer                                                                                                                             |
| **Security** | Audit logs are append-only — no update or delete operations allowed                                                                                                                                            |
|              | Only ADMIN role can query audit logs                                                                                                                                                                           |
| **Testing**  | E2E test various operations and verify corresponding audit log entries                                                                                                                                         |

---

### Capability 9.4 — Geographic Hierarchy Management

#### Feature 9.4.1 — Circle → Division → Subdivision Hierarchy

> **User Story**: As an administrator, I want to manage the geographic organizational hierarchy so that consumers, sites, and ROs are correctly mapped to their regions.

| Layer        | Tasks                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Backend**  | Implement Circle CRUD: `circles` / `circle` / `circleByCode` queries, `createCircle` / `updateCircle` mutations with `CircleResolver`                                    |
|              | Implement Division CRUD: `divisions` / `division` / `divisionByCode` queries, `createDivision` / `updateDivision` mutations with `DivisionResolver`                      |
|              | Implement Subdivision CRUD: `subdivisions` / `subdivision` / `subdivisionByCode` queries, `createSubdivision` / `updateSubdivision` mutations with `SubdivisionResolver` |
|              | Implement `assignRoSubdivisions` mutation for mapping ROs to subdivisions                                                                                                |
|              | Implement nested field resolvers: circle → divisions, division → subdivisions, division → circle, subdivision → division                                                 |
|              | Support `includeInactive` flag for hierarchy queries                                                                                                                     |
| **Frontend** | Build master data management at `(admin)/admin/master` for hierarchy CRUD                                                                                                |
| **Security** | Only ADMIN/SUPER_ADMIN can create/update hierarchy entities                                                                                                              |
| **Testing**  | Unit test hierarchy CRUD and nested resolver chains                                                                                                                      |

---

### Capability 9.5 — Meter Readings

#### Feature 9.5.1 — Meter Reading Collection & Verification

> **User Story**: As a field operator, I want to record and verify meter readings so that accurate consumption data is captured for billing.

| Layer        | Tasks                                                                                                          |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| **Backend**  | Implement `recordMeterReading` mutation with site, meter, reading value, reading type, and photo evidence link |
|              | Implement `verifyMeterReading` mutation (RO/ADMIN) for approving submitted readings                            |
|              | Implement `meterReadings` / `meterReadingsBySite` queries with pagination and filtering                        |
|              | Implement `latestMeterReading` / `latestMeterReadingByMeter` queries for quick current-state lookups           |
|              | Implement `takenByName` / `meterSerialNumber` / `siteIdentifier` ResolveField resolvers for related data       |
|              | Support reading types: `INITIAL`, `PERIODIC`, `FINAL`, `SPECIAL`                                               |
| **Frontend** | Build meter reading capture flow at `(contractor)/contractor/reading` with camera and manual input             |
| **Security** | Enforce RBAC per reading type: contractors can record; only RO/ADMIN can verify                                |
| **Testing**  | Unit test reading creation and verification with type validation                                               |

---

### Capability 9.6 — Background Scheduling

#### Feature 9.6.1 — Scheduled Task Management

> **User Story**: As a system, I want background jobs to run on schedules for automated tasks like SLA breach checking and subscription expiry.

| Layer              | Tasks                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **Backend**        | Implement scheduler module using `@nestjs/schedule` for cron-based background task execution  |
|                    | Implement SLA breach detection job — checks active marketplace jobs against SLA targets       |
|                    | Implement subscription expiry notifications — alerting contractors before subscription lapses |
| **Infrastructure** | Configure cron schedules via environment variables                                            |
| **Testing**        | Unit test scheduled task logic with mocked time                                               |

---

### Capability 9.7 — Consumer Self-Service

#### Feature 9.7.1 — Consumer Help & Support

> **User Story**: As a consumer, I want access to help and support features within my dashboard so that I can resolve issues with my meter or installation.

| Layer        | Tasks                                                         |
| ------------ | ------------------------------------------------------------- |
| **Backend**  | Implement support module for consumer help desk functionality |
| **Frontend** | Build consumer help page at `(consumer)/consumer/help`        |
| **Testing**  | E2E test support ticket creation and retrieval                |

---

## Epic 10 — Admin Panel Enhancements

### Capability 10.1 — Marketplace Admin Configuration

#### Feature 10.1.1 — Global SLA Management

> **User Story**: As an administrator, I want to configure a global fallback SLA so that marketplace jobs always have a deadline even when no service-specific SLA is set.

| Layer        | Tasks                                                                                 |
| ------------ | ------------------------------------------------------------------------------------- |
| **Backend**  | Add `GlobalSla` ObjectType and `UpsertGlobalSlaInput` to `sla.types.ts`               |
|              | Add `getGlobalSla()` to `SlaService` — fetches single active row from `mp_global_sla` |
|              | Add `upsertGlobalSla()` to `SlaService` — insert or update the global SLA row         |
|              | Add `globalSla` query and `upsertGlobalSla` mutation to `SlaResolver` (Admin only)    |
| **Frontend** | Create `/admin/marketplace/global-sla` page with SLA timeline editor                  |
|              | Add Acceptance, Job Start, Completion hour inputs with live visual timeline preview   |
|              | Add active/inactive toggle with warning when disabled                                 |
|              | Display last-updated timestamp from DB                                                |

---

#### Feature 10.1.2 — Subscription Plan Management

> **User Story**: As an administrator, I want to create and edit subscription pricing plans so that contractors can subscribe to the marketplace at different tiers.

| Layer        | Tasks                                                                                           |
| ------------ | ----------------------------------------------------------------------------------------------- |
| **Backend**  | Add `CreateSubscriptionPlanInput` InputType to `subscriptions.types.ts`                         |
|              | Add `createSubscriptionPlan()` to `SubscriptionsService` — inserts to `mp_subscription_plans`   |
|              | Add `createSubscriptionPlan` mutation to `SubscriptionsResolver` (Admin only)                   |
| **Frontend** | Add **Add Plan** button to `/admin/marketplace/subscriptions` page                              |
|              | Build Add Plan modal with duration, months, base price, discount, live final price, description |
|              | Improve Edit modal — show warning and disable Save when plan has no DB record (fallback mode)   |
|              | Add Subscription Plans and Global SLA cards to `/admin/marketplace` configuration section       |

---

#### Feature 10.1.3 — Consumer Single Upload Form Expansion

> **User Story**: As an administrator, I want the single consumer upload form to capture all required fields so that consumers are fully registered without needing a secondary step.

| Layer        | Tasks                                                                                               |
| ------------ | --------------------------------------------------------------------------------------------------- |
| **Frontend** | Expand single consumer form to include Administrative Hierarchy (Circle, Division, Subdivision IDs) |
|              | Add Connection and Meter fields (Tariff Code, Connection Type, Supply Voltage, Meter Type)          |
|              | Add KYC fields (Identity Type, Identity Number, GSTN)                                               |
|              | Add full Billing Address fields (Line, City, District, State, Pincode)                              |
|              | Organise form into labelled sections with a **View Consumers** shortcut button                      |

---

## Identified Bugs & Technical Debt

| ID      | Severity  | Area             | Description                                                                                                                                                                                |
| ------- | --------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| BUG-001 | 🔴 High   | Billing          | `BillingResolver` is a complete placeholder — no queries or mutations implemented. `generateBillingExport`, `billingExports`, and `billingSummary` are listed as TODOs.                    |
| BUG-002 | 🟡 Medium | Auth             | `me` query throws generic `Error('User not found')` instead of using NestJS `NotFoundException` — will not be caught cleanly by global exception filter.                                   |
| BUG-003 | 🟡 Medium | Marketplace Jobs | Multiple resolvers (`createJob`, `acceptJob`, `rejectJob`, `generateOtp`, `verifyOtp`, `cancelJob`) throw raw `Error('Consumer/Contractor not found')` instead of typed NestJS exceptions. |
| BUG-004 | 🟡 Medium | Subscriptions    | `confirmPayment` fetches payment intent synchronously (`getPaymentIntent` is not `async`) but uses it in an `async` method — potential inconsistency if storage becomes async.             |
| BUG-005 | 🟢 Low    | Files            | `uploadBase64File` mutation is deprecated in favor of REST but still active — no deprecation enforcement at runtime.                                                                       |
| BUG-006 | 🟢 Low    | Consumers        | `ConsumersResolver` injects raw `db` for field resolvers (`circleName`, `divisionName`, `subdivisionName`) instead of using a dedicated hierarchy service — violates modular architecture. |
| BUG-007 | 🟢 Low    | Meter Readings   | `MeterReadingsResolver` directly injects and queries `db` for field resolvers instead of delegating to respective services — inconsistent with service-based architecture.                 |
| BUG-008 | 🟡 Medium | Payments         | Payment webhook endpoints are noted as REST but no REST controller implementation is visible in the payments module.                                                                       |
| BUG-009 | 🟢 Low    | Testing          | Only `app.controller.spec.ts` exists as a test file — no unit or E2E tests for any module resolver or service.                                                                             |
| BUG-010 | 🟡 Medium | Support          | `support` module has `imports.resolver.ts` (data imports) rather than a support/helpdesk resolver — module naming mismatch.                                                                |

---

> _This PMS breakdown was generated from analysis of 31 GraphQL resolvers, 16+ database schema files, 178 enum values, and 5 frontend route groups across the SmartMeter codebase._
