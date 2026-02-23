# Session Changes Log

**Last Updated:** 23 February 2026

This document tracks all changes made during development sessions. Update this file as changes are made.

---

## Session: 23 February 2026 - Forgot Password, RO Workflows & Admin Multi-Tenancy

### 1. Branding & Login Restructuring

- **KSLECA branding applied Globally:** Fixed 12 instances of "KSLECAR" → "KSLECA" across layouts, headers, footers, and sidebars.
- **Login Strict Mode:** Removed `isProd` environment-based conditional rendering from `/login` page. The application now strictly defaults to the **Retail Outlet Portal** login block in all environments (development and production). Admin login remains accessible only via the `?role=admin` query parameter linked in the footer.
- **Remember Me Functionality:** Implemented dual-storage authentication. `rememberMe=true` utilizes `localStorage`, while `false` defaults to tab-isolated `sessionStorage`.

### 2. Forgot Password Flow Implementation

**Backend (`auth.service.ts`, `auth.resolver.ts`):**

- Added `requestPasswordReset(email)` to generate a 6-digit OTP and send via email.
- Added `resetPassword(email, otp, newPassword)` to verify the OTP and hash the new password.
- Included anti-enumeration (always returns success) and throttle protection.

**Frontend (`forgot-password/page.tsx`):**

- Built a 3-step wizard (Email → OTP Verification → New Password).
- Connected to the new GraphQL reset mutations.

### 3. Consumer & Admin Multi-Tenancy

- **Admin Association Filtering:** Overhauled `getAccessContext()` in `consumers.service.ts` for the `ADMIN` role. Admins no longer see all consumers/sites; their visibility is strictly mapped to the States assigned to their `Association`. Admins with no active association now see exactly zero records.
- **Consumer Import Pipeline:** Prevented unnecessary fields (Meter Number, Sold, Installed, Service) from being processed or throwing errors during the Consumer Bulk Excel Upload, as these fields are strictly tied to backend operational states.

### 4. Retail Outlet Workflows & Cleanups

- **Role Restrictions:** Restricted `SUB_USER` roles from accessing or manipulating the RO Staff creation UI, and removed `importMeters` access from `SUPER_ADMIN`.
- **Meter Bulk Imports:** Migrated the Bulk Meter Upload UI from the Admin Upload page to the RO Inventory screen (`/ro/inventory`).
- **RAPDRP Flag:** Added the "Is RAPDRP?" boolean column to both the Meter bulk import template and the UI single-creation modals.
- **Contractor Detail Views:** Built out the `/ro/contractors/[contractorId]/page.tsx` dedicated detail view and linked it from the main contractor table rows.
- **Site Review Simplification:** Disabled interactions on `requiredPhase` and `requiredVoltage` selects in the site review form (falling back to Excel defaults). Integrated the `activateConsumerSite` logic directly into the `handleAssignAndSendLinks` action so consumer account creation and welcome SMS fire automatically when a contractor is assigned.

---

## Session: 19 February 2026 (Evening) - Consumer Data Isolation & Subscriptions Module

### 1. Consumer `createdBy` Column Added

**Problem:** ADMINs could see ALL consumers regardless of who created them. Required data isolation so each ADMIN only sees consumers they created/imported.

#### Schema Change

**File:** `/user-service/src/database/schema/consumers.ts`

- Added `createdBy: uuid('created_by').references(() => users.id)` column
- Added `index('consumers_created_by_idx').on(table.createdBy)` index

#### Service Change

**File:** `/user-service/src/modules/consumers/consumers.service.ts`

- `create()` now saves `createdBy: createdBy` when inserting
- `createMany()` passes `createdBy` through to each `create()` call

#### Migration

**File:** `/user-service/src/database/migrations/0011_freezing_the_initiative.sql`

```sql
ALTER TABLE "consumers" ADD COLUMN "created_by" uuid;
ALTER TABLE "consumers" ADD CONSTRAINT "consumers_created_by_users_id_fk"
  FOREIGN KEY ("created_by") REFERENCES "users"("id");
CREATE INDEX "consumers_created_by_idx" ON "consumers" ("created_by");
```

> ⚠️ Existing consumers will have `createdBy = NULL`. Only newly created consumers track the creator.

---

### 2. ADMIN Data Isolation - Changed from State-Based to Creator-Based

**Problem:** Previous logic used complex association → state code → state name → consumerSites.state filtering. This was incorrect and overly complex.

**File:** `/user-service/src/modules/consumers/consumers.service.ts`

#### `getAccessContext()` - Simplified Return Type

```typescript
// BEFORE
{ roIds: string[], subdivisionIds: string[], stateNames: string[] }

// AFTER
{ roIds: string[], subdivisionIds: string[], createdByUserId: string | null }
```

#### ADMIN Logic Changed

```typescript
// BEFORE: Complex association → stateCode → stateName lookup (multiple DB queries)
// AFTER: Single return
if (userContext.role === UserRole.ADMIN) {
  return { roIds: [], subdivisionIds: [], createdByUserId: userContext.id };
}
```

#### `findAll()` - New Filter Logic

```typescript
// ADMIN: filter by consumers.createdBy = userId
if (accessContext.createdByUserId) {
  conditions.push(eq(consumers.createdBy, accessContext.createdByUserId));
}
// RO/SUB_USER: unchanged - filter via consumerSites join
```

#### Data Isolation Summary

| Role            | Sees                                               |
| --------------- | -------------------------------------------------- |
| `SUPER_ADMIN`   | All consumers                                      |
| `ADMIN`         | Only consumers they created (`createdBy = userId`) |
| `RETAIL_OUTLET` | Consumers with sites in their subdivision          |
| `SUB_USER`      | Consumers with sites in parent RO's subdivision    |

---

### 3. Removed Stale Imports from Consumers Service

**File:** `/user-service/src/modules/consumers/consumers.service.ts`

- Removed `associationUsers`, `associationStates` imports from `associations` schema
- Removed `stateMaster` import from `marketplace/master-states`
- Removed `inArray` was kept (still used for site-based filtering)

---

### 4. Role Comparison Fix - String Literals → Enum Values

**Problem:** TypeScript error: `The two values in this comparison do not have a shared enum type`

**File:** `/user-service/src/modules/consumers/consumers.service.ts`

Replaced all string literal role comparisons with `UserRole` enum:

```typescript
// BEFORE
if (userContext.role === 'SUPER_ADMIN') { ... }
if (userContext.role === 'ADMIN') { ... }
if (userContext.role === 'RETAIL_OUTLET') { ... }
if (userContext.role === 'SUB_USER') { ... }

// AFTER
if (userContext.role === UserRole.SUPER_ADMIN) { ... }
if (userContext.role === UserRole.ADMIN) { ... }
if (userContext.role === UserRole.RETAIL_OUTLET) { ... }
if (userContext.role === UserRole.SUB_USER) { ... }
```

---

### 5. Minor Lint Fix - `let` → `const` for `total`

**File:** `/user-service/src/modules/consumers/consumers.service.ts`

- Changed `let total: number` declaration + separate assignment to `const total = countResult[0]?.count || 0`

---

### 6. Installed Missing AWS SDK Package

**File:** `/user-service/package.json`

```bash
pnpm add @aws-sdk/client-sesv2
```

Previous `@aws-sdk/client-ses` was replaced with `client-sesv2` (used by `email.service.ts`).

---

### 7. Subscriptions Module - Contractor Premium Upgrade ✅ NEW MODULE

**New files created:**

| File                                                                             | Description                  |
| -------------------------------------------------------------------------------- | ---------------------------- |
| `/user-service/src/modules/marketplace/subscriptions/subscriptions.types.ts`     | GraphQL types, enums, inputs |
| `/user-service/src/modules/marketplace/subscriptions/subscriptions.service.ts`   | Core business logic          |
| `/user-service/src/modules/marketplace/subscriptions/subscriptions.resolver.ts`  | GraphQL queries + mutations  |
| `/user-service/src/modules/marketplace/subscriptions/subscriptions.scheduler.ts` | Daily expiry cron job        |
| `/user-service/src/modules/marketplace/subscriptions/subscriptions.module.ts`    | NestJS module                |
| `/user-service/src/modules/marketplace/subscriptions/index.ts`                   | Barrel exports               |

#### Subscription Plans

| Plan      | Duration  | Price  | Discount | Final  |
| --------- | --------- | ------ | -------- | ------ |
| MONTHLY   | 1 month   | ₹499   | 0%       | ₹499   |
| QUARTERLY | 3 months  | ₹1,497 | 10%      | ₹1,347 |
| YEARLY    | 12 months | ₹5,988 | 25%      | ₹4,491 |

#### Contractor Upgrade Flow

```
1. GET subscriptionPlans              → View pricing
2. GET mySubscriptionStatus           → Check current status + canUpgrade/canRenew
3. MUTATION initiateSubscriptionUpgrade(contractorId, duration)
   → Returns paymentIntent { id, amount, paymentLink, orderId, expiresAt (30 min) }
4. Contractor pays via paymentLink (currently mock URL)
5. MUTATION confirmSubscriptionPayment(paymentIntentId)
   → Sets subscriptionType = PREMIUM
   → Sets subscriptionValidTill = now + duration (or extends from existing expiry)
   → Creates record in mp_contractor_subscriptions
```

#### Admin Override

```graphql
# Admin can set subscription directly without payment
mutation adminSetContractorSubscription(input: AdminSetSubscriptionInput!)
```

#### Expiry Scheduler

- Runs **daily at midnight** via `@Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)`
- Also runs **10 seconds after server startup**
- Finds all `subscriptionType = PREMIUM` where `subscriptionValidTill <= NOW`
- Downgrades each to `FREE`, deactivates old record, creates new FREE record

#### GraphQL API Added

```graphql
# Queries
subscriptionPlans: [SubscriptionPlan!]!                 # Public
mySubscriptionStatus: SubscriptionStatus!               # CONTRACTOR
mySubscriptionHistory: [ContractorSubscription!]!       # CONTRACTOR
contractorSubscriptionStatus(contractorId: ID!): SubscriptionStatus!  # ADMIN
contractorSubscriptionHistory(contractorId: ID!): [ContractorSubscription!]!  # ADMIN
subscriptionPaymentIntent(id: ID!): SubscriptionPaymentIntent  # Auth

# Mutations
initiateSubscriptionUpgrade(input: ...): SubscriptionUpgradeResponse!   # CONTRACTOR
confirmSubscriptionPayment(input: ...): SubscriptionConfirmResponse!     # CONTRACTOR
adminSetContractorSubscription(input: ...): SubscriptionConfirmResponse! # ADMIN
```

#### Marketplace Module Updated

**File:** `/user-service/src/modules/marketplace/marketplace.module.ts`

- Added `SubscriptionsService`, `SubscriptionsResolver`, `SubscriptionsScheduler` to providers
- Added `SubscriptionsService` to exports

#### Types Index Updated

**File:** `/user-service/src/modules/marketplace/types/index.ts`

- Added `export * from '../subscriptions/subscriptions.types'`

#### ⚠️ Production TODOs

- Replace in-memory `paymentIntents` Map with DB table or Redis
- Wire `paymentLink` to real Cashfree payment order creation
- Add Cashfree webhook endpoint to auto-confirm payment

---

### 8. MARKETPLACE_IMPLEMENTATION_PLAN.md Updated

**File:** `/docs/MARKETPLACE_IMPLEMENTATION_PLAN.md`

- Status updated to `In Progress (Subscriptions Module ✅ COMPLETED)`
- `contractor_subscriptions` schema section marked ✅ IMPLEMENTED with pricing table
- Added full `SubscriptionsService` and `SubscriptionsScheduler` documentation in Section 4.2
- Updated GraphQL mutations: replaced `upgradeSubscription` with actual implemented mutations
- Added all new subscription queries to GraphQL queries section
- Added `adminSetContractorSubscription` to admin mutations
- Subscriptions folder marked ✅ IMPLEMENTED in new files section
- Phase 5 checklist + metrics updated
- Added **Appendix D** with full flow documentation, GraphQL types reference, and production TODOs

---

## Session: 19 February 2026 - Backend Restructuring & Fixes

### 1. Bug Fixes

#### marketplaceJobs Query Validation Error

**Problem:** `property page should not exist, property limit should not exist` error when fetching jobs
**File:** `/user-service/src/modules/marketplace/types/jobs.types.ts`
**Fix:** Added `@IsOptional()` decorator to `page` and `limit` fields in `JobsFilterInput` and `SlaBreachFilterInput`

#### adminAssociations SUPER_ADMIN Access

**Problem:** SUPER_ADMIN couldn't access associations queries (403 Forbidden)
**File:** `/user-service/src/modules/marketplace/resolvers/admin-associations.resolver.ts`
**Fix:** Added `UserRole.SUPER_ADMIN` to all query role decorators:

- `adminAssociations`
- `adminAssociation`
- `myAdminAssociation`
- `adminAssociationStates`
- `adminAssociationUsers`
- `myAssignedStateCodes`

### 2. Admin Creation - Association Now Mandatory

**File:** `/web-frontend/src/app/(admin)/admin/admins/page.tsx`

- Association selection is now REQUIRED (not optional)
- Validation error shown if no association selected
- Warning displayed if no associations exist in the system

### 3. SUPER_ADMIN Marketplace Access Restriction

**Problem:** SUPER_ADMIN could access marketplace management pages (should only see stats)

**Files Modified:**

- `/web-frontend/src/app/(admin)/admin/marketplace/page.tsx` - Split view: SUPER_ADMIN sees stats dashboard, ADMIN sees management grid
- `/web-frontend/src/app/(admin)/admin/marketplace/locations/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/services/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/pricing/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/sla/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/contractors/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/jobs/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/categories/page.tsx` - Added SUPER_ADMIN restriction
- `/web-frontend/src/app/(admin)/admin/marketplace/uoms/page.tsx` - Added SUPER_ADMIN restriction

**Behavior:**

- SUPER_ADMIN visiting management pages sees "Access Restricted" message with Go Back button
- Only ADMIN role can access management features

### 4. Associations Module - Moved Out of Marketplace

**Rationale:** Admin Associations are a core feature, not specific to marketplace. Moved to separate module.

#### New Files Created:

**Schema:**

- `/user-service/src/database/schema/associations.ts` - New tables: `associations`, `association_states`, `association_users`

**Module:**

- `/user-service/src/modules/associations/associations.module.ts`
- `/user-service/src/modules/associations/associations.service.ts`
- `/user-service/src/modules/associations/associations.resolver.ts`
- `/user-service/src/modules/associations/associations.types.ts`
- `/user-service/src/modules/associations/index.ts`

**Migration:**

- `/user-service/src/database/migrations/0008_associations_rename.sql` - Renames old `mp_admin_*` tables to `association_*`

#### GraphQL Changes:

Old API (still works via marketplace module):

- `adminAssociations` / `adminAssociation` / `myAdminAssociation`
- `createAdminAssociation` / `updateAdminAssociation`
- `assignStatesToAdminAssociation` / `removeStateFromAdminAssociation`
- `addUserToAdminAssociation` / `removeUserFromAdminAssociation`

New API (from associations module):

- `associations` / `association` / `myAssociation`
- `createAssociation` / `updateAssociation`
- `assignStatesToAssociation` / `removeStateFromAssociation`
- `addUserToAssociation` / `removeUserFromAssociation`

### 5. Marketplace Module - Removed Association References

**File:** `/user-service/src/modules/marketplace/marketplace.module.ts`

- Removed `AdminAssociationsService` and `AdminAssociationsResolver` imports
- Associations now in separate `AssociationsModule`

**File:** `/user-service/src/modules/marketplace/services/index.ts`

- Removed `admin-associations.service` export

**File:** `/user-service/src/modules/marketplace/resolvers/index.ts`

- Removed `admin-associations.resolver` export

**File:** `/user-service/src/database/schema/marketplace/index.ts`

- Removed `admin-associations` export

### 6. Location Hierarchy - New Schema Structure

**Problem:** Original `mp_location_master` table had flat structure with repeated state/district names

**New Hierarchical Structure:**

**File:** `/user-service/src/database/schema/marketplace/service-locations.ts`

```
service_states
    └── service_districts (FK → state)
        └── service_sub_districts (FK → district, state)
            └── service_villages (FK → sub_district, district, state)
```

**Tables:**

1. `service_states` - State code, name, local name
2. `service_districts` - District code, name, foreign key to state
3. `service_sub_districts` - Sub-district code, name, FK to district & state
4. `service_villages` - Village code, name, category, area_type, classification info, FK to all parent levels

**Benefits:**

- No repeated state/district names
- Proper relational structure
- Easier hierarchical queries
- Better data integrity with foreign keys

**Migration:**

- `/user-service/src/database/migrations/0009_service_location_hierarchy.sql`
- Creates new tables
- Migrates data from `mp_location_master` to new hierarchy
- Old table kept for backward compatibility

---

## Session: 18 February 2026 - Admin Module Completion

### 1. Admin Associations - SUPER_ADMIN Only

#### Backend Access Control

**File:** `/user-service/src/modules/marketplace/resolvers/admin-associations.resolver.ts`

Changed all mutations from `@Roles(UserRole.ADMIN)` to `@Roles(UserRole.SUPER_ADMIN)`:

- `createAdminAssociation` - SUPER_ADMIN only
- `updateAdminAssociation` - SUPER_ADMIN only
- `deleteAdminAssociation` - SUPER_ADMIN only
- `addUserToAdminAssociation` - SUPER_ADMIN only
- `removeUserFromAdminAssociation` - SUPER_ADMIN only

#### Frontend - Removed from Regular Admin UI

**Deleted Files:**

- `/web-frontend/src/app/(admin)/admin/marketplace/associations/page.tsx`
- `/web-frontend/src/hooks/marketplace/useAdminAssociations.ts`

**Modified Files:**

- `/web-frontend/src/hooks/marketplace/index.ts` - Removed associations exports
- `/web-frontend/src/app/(admin)/admin/marketplace/page.tsx` - Removed Associations card from dashboard

#### Admin Creation with Association Assignment

**File:** `/web-frontend/src/app/(admin)/admin/admins/page.tsx`

Added functionality:

- Fetches associations using `GET_ADMIN_ASSOCIATIONS` query
- Shows dropdown to select association when creating new admin
- After admin creation, calls `ADD_USER_TO_ASSOCIATION` mutation
- Helper text: "Assign to an admin group for access control"

---

### 2. Location Import - Moved to Marketplace Locations Page

#### Removed from Upload Page

**File:** `/web-frontend/src/app/(admin)/admin/upload/page.tsx`

- Removed `locations` import type
- Changed grid from `md:grid-cols-4` to `md:grid-cols-3`
- Removed `MapPin` icon import
- Removed `IMPORT_LOCATIONS` mutation import
- Removed `locationResult` state and related code
- Updated instructions to remove location-specific info

#### Added to Marketplace Locations Page

**File:** `/web-frontend/src/app/(admin)/admin/marketplace/locations/page.tsx`

Complete rewrite with:

- Import functionality with `IMPORT_LOCATIONS` mutation
- Download template button (CSV with all required columns)
- File upload area with drag-and-drop style
- Progress indicators: uploading → processing → done
- Result display: totalRows, inserted, updated, skipped
- Error display with scrollable list
- NO manual one-by-one creation (Plus buttons removed)
- 4-column location browser (States → Districts → Sub-Districts → Villages)
- Villages show category badge

---

### 3. Village Category - Made Required

#### Backend Type Changes

**File:** `/user-service/src/modules/marketplace/types/locations.types.ts`

```typescript
// LgdExcelRow - villageCategory now required
export interface LgdExcelRow {
  // ... other fields
  villageCategory: string; // Was: villageCategory?: string
}

// MarketplaceLocation - villageCategory non-nullable
@Field({ description: 'Village category - REQUIRED during import' })
villageCategory: string; // Was: @Field({ nullable: true }) villageCategory?: string

// VillageOption - added category field
@ObjectType()
export class VillageOption {
  // ... other fields
  @Field({ description: 'Village category' })
  category: string; // NEW FIELD
}
```

#### Backend Service Changes

**File:** `/user-service/src/modules/marketplace/services/locations.service.ts`

```typescript
// Validation now requires villageCategory
if (
  !row.villageCode ||
  !row.villageName ||
  !row.stateCode ||
  !row.districtCode ||
  !row.villageCategory
) {
  result.errors.push(
    `Row ${i + 1}: Missing required fields (villageCode, villageName, stateCode, districtCode, villageCategory)`,
  );
}

// getVillages() now includes category
const result = await this.db.select({
  // ... other fields
  category: locationMaster.villageCategory,
});

return result.map((r) => ({
  // ... other fields
  category: r.category || "Unknown",
}));
```

#### Frontend Changes

**File:** `/web-frontend/src/hooks/marketplace/useMarketplaceLocations.ts`

```typescript
export interface VillageOption {
  id: string;
  code: string;
  name: string;
  nameLocal?: string;
  category: string; // NEW - required
  areaType?: "URBAN" | "SEMI_URBAN" | "RURAL";
  isClassified: boolean;
}
```

**File:** `/web-frontend/src/graphql/marketplace.ts`

```graphql
query GetMarketplaceVillages($districtCode: String!, $subDistrictCode: String) {
  marketplaceVillages(
    districtCode: $districtCode
    subDistrictCode: $subDistrictCode
  ) {
    id
    code
    name
    nameLocal
    category # NEW FIELD
    areaType
    isClassified
  }
}
```

---

## Session: February 2026 (Previous)

### 1. Build System Migration

#### npm → pnpm Migration

- **Backend:** Migrated `/smartmeter-backend` from npm to pnpm
- **Frontend:** Migrated `/smartmeter-frontend` from npm to pnpm
- Deleted `package-lock.json` files, using `pnpm-lock.yaml` instead

---

### 2. Package Updates (Frontend)

Updated to latest versions:

- React 19.2.4
- Next.js 16.1.6
- Apollo Client 3.14.0
- TypeScript 5.9.3
- ESLint 9.x

---

### 3. Storage Migration (Frontend)

Changed from `localStorage` to `sessionStorage` for tab isolation:

- Renamed storage keys for consistency
- Sessions are now isolated per browser tab

---

### 4. GraphQL Codegen Setup (Frontend)

**File:** `/smartmeter-frontend/codegen.ts`

- Added GraphQL Code Generator configuration
- Output location: `src/graphql/__generated__/`
- Generated files: `graphql.ts`, `hooks.ts`, `gql.ts`, `index.ts`
- Run with: `pnpm codegen`

---

### 5. CORS Configuration (Backend)

**File:** `/smartmeter-backend/src/main.ts`

- CORS is now conditional based on `ENABLE_CORS=true` environment variable
- Default: CORS disabled

---

### 6. GraphQL Upload Removal (Backend)

**File:** `/smartmeter-backend/src/main.ts`

- Removed `graphqlUploadExpress` middleware completely
- All file uploads now use REST API (`POST /files/upload/:mode/:context`)
- GraphQL mutations receive `fileKey` (storage key) instead of file streams

---

### 7. Import System Overhaul (Backend)

#### New Module: Import Logs

**Location:** `/smartmeter-backend/src/modules/import-logs/`

Files created:

- `import-logs.service.ts` - Core service with hash checking
- `import-logs.types.ts` - GraphQL types
- `import-logs.module.ts` - NestJS module
- `index.ts` - Barrel exports

Features:

- SHA-256 hash-based duplicate file detection
- Import logging with status tracking
- Stores import history in `import_logs` table
- Stores file hashes in `file_hashes` table

#### New Database Tables

**File:** `/smartmeter-backend/src/database/schema/index.ts`

```typescript
// import_logs - Tracks all import operations
export const importLogs = pgTable("import_logs", {
  id,
  fileKey,
  fileName,
  fileHash,
  entityType,
  status,
  totalRows,
  successCount,
  failedCount,
  errors,
  importedBy,
  startedAt,
  completedAt,
});

// file_hashes - Prevents duplicate imports
export const fileHashes = pgTable("file_hashes", {
  id,
  hash,
  fileKey,
  entityType,
  uploadedBy,
  createdAt,
});
```

#### Imports Resolver Refactored

**File:** `/smartmeter-backend/src/modules/support/imports.resolver.ts`

Changes:

- Now uses file key from REST upload (not GraphQL Upload)
- Hash check before processing
- Bulk insert with database transactions (all-or-nothing)
- Validates all rows before any DB writes
- Uses proper enum constants instead of string literals

---

### 8. Bulk Transaction Inserts (Backend)

#### Consumers Import

- Batch insert all users, consumers, and sites in single transaction
- All-or-nothing behavior - if any fails, all roll back

#### Contractors Import

- Changed from row-by-row to bulk transaction insert
- Validates all rows first, then bulk inserts
- Pre-hashes passwords before transaction

---

### 9. Database Enum Constants (Backend)

**File:** `/smartmeter-backend/src/common/enums/index.ts`

Added type-safe enum constants for database inserts:

```typescript
export const DbUserRole = {
  SUPER_ADMIN: "SUPER_ADMIN",
  ADMIN: "ADMIN",
  CONSUMER: "CONSUMER",
  RETAIL_OUTLET: "RETAIL_OUTLET",
  CONTRACTOR: "CONTRACTOR",
  SUB_USER: "SUB_USER",
} as const;

export const DbConsumerState = {
  REGISTERED: "REGISTERED",
  ACTIVE: "ACTIVE",
  SUSPENDED: "SUSPENDED",
  DEACTIVATED: "DEACTIVATED",
} as const;

export const DbSiteState = {
  CREATED: "CREATED",
  SITE_PENDING: "SITE_PENDING",
  // ... all states
} as const;
```

**Usage in imports.resolver.ts:**

```typescript
role: DbUserRole.CONSUMER,      // instead of 'CONSUMER' as const
state: DbConsumerState.REGISTERED,
currentState: DbSiteState.CREATED,
```

---

### 10. Excel File Size Limit (Backend)

**File:** `/smartmeter-backend/src/modules/files/files.controller.ts`

Added configurable size limit for Excel/CSV imports:

```typescript
const EXCEL_IMPORT_MAX_SIZE_MB = parseInt(
  process.env.EXCEL_IMPORT_MAX_SIZE_MB || "3",
  10,
);
```

**Environment Variable:**

```env
EXCEL_IMPORT_MAX_SIZE_MB=3  # Default 3MB
```

---

### 11. Installation Evidence - fileKey Support (Backend)

**File:** `/smartmeter-backend/src/modules/installation-evidence/installation-evidence.types.ts`

Added `fileKey` field to `CreateInstallationEvidenceInput`:

```typescript
@Field({ nullable: true, description: 'Storage key from REST file upload (preferred method)' })
fileKey?: string;
```

**File:** `/smartmeter-backend/src/modules/installation-evidence/installation-evidence.service.ts`

Updated to handle fileKey with priority:

```typescript
// Priority: fileKey > fileUrl > base64Data
if (input.fileKey) {
  fileUrl = await generateUrl(input.fileKey, "private");
  fileName = input.fileName || input.fileKey.split("/").pop();
}
```

---

### 12. Deprecated GraphQL Mutations (Backend)

Marked as deprecated (still functional for backwards compatibility):

| Mutation                | File              | Deprecation Reason                |
| ----------------------- | ----------------- | --------------------------------- |
| `uploadBase64File`      | files.resolver.ts | Use REST upload API               |
| `uploadSitePhotoBase64` | sites.resolver.ts | Use REST upload + uploadSitePhoto |

---

### 13. Files Service Extensions (Backend)

**File:** `/smartmeter-backend/src/modules/files/files.service.ts`

Added methods:

- `downloadAsBuffer(fileKey, mode)` - Download file as Buffer
- `calculateHash(buffer)` - Calculate SHA-256 hash
- `uploadImportFile(file, context, mode, userId)` - Upload import files

---

### 14. JWT Default Secret (Backend)

**Files:**

- `/smartmeter-backend/src/modules/auth/auth.module.ts`
- `/smartmeter-backend/src/modules/auth/jwt.strategy.ts`

Changed default JWT secret from empty to `'secret'` for development.

---

## Pending / TODO

- [ ] Chunked Excel processing for large files (memory efficiency)
- [ ] Update frontend to use `fileKey` instead of `fileUrl` for storage keys
- [ ] Regenerate frontend GraphQL types after backend schema changes
- [ ] Add more comprehensive error handling for imports
- [ ] Run migration for auth_throttle table: `pnpm drizzle-kit generate --name add_auth_throttle && pnpm drizzle-kit push`

---

### 15. Schema Split into Separate Files (Backend)

**Location:** `/smartmeter-backend/src/database/schema/`

The monolithic 1200+ line `index.ts` was split into domain-specific modules:

| File                        | Tables/Exports                                                             |
| --------------------------- | -------------------------------------------------------------------------- |
| `enums.ts`                  | All pgEnum definitions (userRoleEnum, meterStateEnum, siteStateEnum, etc.) |
| `users.ts`                  | users table                                                                |
| `auth-throttle.ts`          | authThrottle table, THROTTLE_RULES constant                                |
| `hierarchy.ts`              | circles, divisions, subdivisions, retailOutlets, roSubdivisions            |
| `contractors.ts`            | contractors table                                                          |
| `consumers.ts`              | consumers, consumerSites tables                                            |
| `meters.ts`                 | meterSpecs, meters tables                                                  |
| `site-meter-connections.ts` | siteMeterConnections table                                                 |
| `files.ts`                  | files table                                                                |
| `installations.ts`          | installations, installationEvidence, verifications tables                  |
| `audit.ts`                  | auditLogs table                                                            |
| `imports.ts`                | excelImports, importLogs, fileHashes tables                                |
| `billing.ts`                | stateTransitions, billingExports, billingExportItems tables                |
| `index.ts`                  | Barrel exports all modules                                                 |

---

### 16. Auth Throttle System (Backend)

**New Table:** `auth_throttle`

Tracks failed login/OTP attempts with time-based lockouts:

```typescript
// /src/database/schema/auth-throttle.ts
export const authThrottle = pgTable("auth_throttle", {
  id,
  identifier, // "login:email@example.com" or "otp:+919876543210"
  throttleType, // PASSWORD_LOGIN or OTP_VERIFY
  failureCount,
  lockedUntil,
  lastFailureAt,
  createdAt,
  updatedAt,
});

export const THROTTLE_RULES = {
  PASSWORD_LOGIN: [
    { failures: 3, lockMs: 5 * 60 * 1000 }, // 5 min
    { failures: 6, lockMs: 30 * 60 * 1000 }, // 30 min
    { failures: 10, lockMs: 2 * 60 * 60 * 1000 }, // 2 hr
    { failures: 15, lockMs: 24 * 60 * 60 * 1000 }, // 24 hr
    { failures: 20, lockMs: 48 * 60 * 60 * 1000 }, // 48 hr
  ],
  OTP_VERIFY: [
    { failures: 5, lockMs: 5 * 60 * 1000 }, // 5 min
    { failures: 10, lockMs: 30 * 60 * 1000 }, // 30 min
    { failures: 15, lockMs: 2 * 60 * 60 * 1000 }, // 2 hr
  ],
};
```

**New Service:** `/src/modules/auth/throttle.service.ts`

Methods:

- `check(identifier, type)` - Throws 429 if locked
- `recordFailure(identifier, type)` - Increment failure count, apply lock if threshold
- `resetOnSuccess(identifier, type)` - Delete record on successful auth

**Behavior:**

- Time-based locks only (no permanent disable)
- Successful auth = full reset (record deleted)
- HTTP 429 response includes: `retryAfter`, `lockedUntil`

---

### 17. Role-Based Login (Backend)

**New Enum:** `LoginType`

```typescript
// /src/modules/auth/auth.types.ts
export enum LoginType {
  ADMIN = "ADMIN",
  RO = "RO",
  CONTRACTOR = "CONTRACTOR",
  CONSUMER = "CONSUMER",
}
```

**Updated GraphQL Input:**

```graphql
input LoginInput {
  email: String!
  password: String!
  loginType: LoginType! # NEW - must match user's role
}
```

**Behavior:**

- Login now validates user's role matches the selected `loginType`
- Prevents cross-role authentication (e.g., contractor can't login via admin portal)
- Same generic "Invalid email or password" error for security

---

### 18. OTP Mode Configuration (Backend)

**Environment Variables:**

```env
# OTP_MODE: 'DUMMY' = always use dummy code, 'ACTUAL' = generate real OTP (default)
OTP_MODE=ACTUAL

# Custom dummy OTP code (only used when OTP_MODE=DUMMY)
OTP_DUMMY_CODE=123456
```

**Behavior:**

- `ACTUAL` mode: Generates random 6-digit OTP (requires SMS integration)
- `DUMMY` mode: Always uses `OTP_DUMMY_CODE` (for development/testing)
- Default is `ACTUAL` for production safety

---

## API Flow Summary

### Excel/CSV Import Flow

```
1. POST /files/upload/private/imports  →  { storageKey: "private/.../file.xlsx" }
2. GraphQL: importConsumers(input: { fileKey: "private/.../file.xlsx" })
3. Backend: Download → Hash check → Validate → Bulk insert (transaction)
```

### Photo Upload Flow

```
1. POST /files/upload/private/installation  →  { storageKey: "private/.../photo.jpg" }
2. GraphQL: uploadInstallationEvidence(input: { fileKey: "private/.../photo.jpg", ... })
```

### Site Photo Flow

```
1. POST /files/upload/private/site-photos  →  { storageKey, url }
2. GraphQL: uploadSitePhoto(id, input: { photoUrl: url })
```

---

## Environment Variables Added

```env
# CORS
ENABLE_CORS=true

# Excel Import Size Limit (MB)
EXCEL_IMPORT_MAX_SIZE_MB=3

# OTP Mode: 'DUMMY' or 'ACTUAL' (default: ACTUAL)
OTP_MODE=ACTUAL

# Custom dummy OTP code (only used when OTP_MODE=DUMMY)
OTP_DUMMY_CODE=123456
```

---

### 19. Apollo Upload Client Removal (Frontend)

**Removed packages:**

- `apollo-upload-client`
- `@types/apollo-upload-client`

**File:** `/src/lib/apollo/client.ts`

- Replaced `createUploadLink` with standard Apollo `HttpLink`
- File uploads now use REST API, not GraphQL

**Deleted file:** `/src/types/apollo-upload-client.d.ts`

---

### 20. Frontend Excel Upload Flow Update (Frontend)

**File:** `/src/graphql/imports.ts`

Updated mutations to use `fileKey` instead of `Upload` scalar:

```graphql
mutation ImportConsumers($fileKey: String!) {
  importConsumers(input: { fileKey: $fileKey }) {
    success
    importLogId
    message
    totalRows
    successCount
    failedCount
    errors
  }
}
```

**File:** `/src/app/(admin)/admin/upload/page.tsx`

New two-step upload flow:

1. Upload file via `fetch()` to REST endpoint: `POST /files/upload/private/imports`
2. Call GraphQL mutation with returned `storageKey`

---

### 21. Frontend Auth LoginType Support (Frontend)

**File:** `/src/graphql/auth.ts`

- Added `LoginType` enum matching backend
- Updated `LOGIN_MUTATION` to include `loginType` parameter

**File:** `/src/lib/auth/AuthProvider.tsx`

- Added `roleToLoginType()` helper function
- Updated `login()` to pass `loginType` to mutation
- Removed redundant post-login role validation (backend now handles it)

---

### 22. MESCOM → SmartMeter Branding Update

**Backend (113+ files):**

- `MESCOM Smart Meter - ` → `SmartMeter - ` (comment headers)
- `MESCOM - ` → `SmartMeter - ` (comment headers)
- `MESCOM Backend` → `SmartMeter Backend` (main.ts startup message)
- `MESCOM Circle/Division/Subdivision` → `Circle/Division/Subdivision`
- Notification messages updated to `SmartMeter` branding
- `MESCOM office` → `support team`

**Frontend (20+ files):**

- `MESCOM Frontend - ` → `SmartMeter - ` (comment headers)
- `MESCOM consumer ID` → `Consumer ID`
- Demo emails: `*@mescom.gov` → `*@smartmeter.app`

---

## TODO / Pending Work

### Billing Export Implementation (REMOVED - TO BE IMPLEMENTED LATER)

**Current Status:** Module removed. Database tables remain for future implementation.

**Removed Files:**

- `/src/modules/billing/billing.service.ts`
- `/src/modules/billing/billing.resolver.ts`
- `/src/modules/billing/billing.types.ts`
- `/src/modules/billing/billing.module.ts`
- `/src/modules/billing/index.ts`

**Database Tables (still exist, can be reused):**

- `billing_exports` - Main export record
- `billing_export_items` - Individual sites/meters in export
- `state_transitions` - Audit trail

**Proposed Flow (for future implementation):**

```
1. ADMIN requests export
2. System queries VERIFIED sites with ACTIVE meter connections
3. Generate CSV/Excel file with billing data:
   - Consumer ID, Site ID, Tariff Code
   - Meter Serial, Initial Reading, Verification Date
   - Consumer Name, Address, Phone
4. Upload file to storage (private)
5. Return download URL to admin
6. ADMIN downloads and sends to external billing system
7. (Optional) ADMIN confirms handoff with external reference
8. Sites remain in VERIFIED state (NOT auto-changed to BILLED)
   - BILLED state should only be set when actually billed externally
```

---

### Other Pending Items

- [ ] Run migration for `auth_throttle` table: `pnpm drizzle-kit generate --name add_auth_throttle && pnpm drizzle-kit push`
- [ ] Chunked Excel processing for large files (memory efficiency)
- [ ] Regenerate frontend GraphQL types after backend schema changes
- [ ] Integrate SmsService into NotificationService (currently separate)
- [ ] Integrate EmailService into NotificationService (currently separate)
- [ ] Register DLT templates with TRAI for Fast2SMS (OTP, welcome, installation)
- [ ] Set up AWS SES and verify domain for production emails

---

## Session: 11 February 2026

### 23. SMS Service Created (Backend)

**File:** `/src/modules/notification/sms.service.ts`

NestJS service for sending SMS via Fast2SMS DLT API:

- Uses axios with authorization header
- Supports DLT template-based messaging
- Can be disabled via `SMS_SERVICES_DISABLED=true`

```typescript
async sendSMS(phone: string, variablesArray: string[], templateId: string): Promise<any>
```

---

### 24. Email Service Created (Backend)

**File:** `/src/modules/notification/email.service.ts`

NestJS service for sending emails via AWS SES + Nodemailer:

- Transporter configured with AWS SES
- Supports attachments, CC, HTML body

```typescript
interface SendEmailOptions {
  sendTo: string;
  cc?: string;
  subject: string;
  body: string;
  html?: string;
  attachments?: EmailAttachment[];
}

async sendMail(options: SendEmailOptions): Promise<string>
```

---

### 25. Notification Config Created (Backend)

**File:** `/src/modules/notification/notification.config.ts`

Centralized configuration for SMS/Email:

- `smsProvider` - SMS provider name (from env)
- `isSmsDisabled` - Flag to disable SMS
- `templateIds` - DLT template IDs (to be registered with TRAI)
- `fast2smsMessageIds` - Fast2SMS message IDs
- `emailConfig` - Default from address

---

### 26. Environment Variables Added (SMS/Email)

**File:** `.env.example`

```env
# SMS Configuration
SMS_PROVIDER=fast2sms
SMS_SERVICES_DISABLED=false
FAST2SMS_API_KEY=your-fast2sms-api-key
SMS_GATEWAY_ENABLED=false

# Email Configuration (AWS SES)
EMAIL_ENABLED=false
EMAIL_FROM=do-not-reply@smartmeter.app
AWS_REGION=ap-south-1
# AWS_ACCESS_KEY_ID=your-access-key
# AWS_SECRET_ACCESS_KEY=your-secret-key
```

---

### 27. NotificationService Async Fixes (Backend)

**File:** `/src/modules/notification/notification.service.ts`

Fixed linter errors - removed `async` from methods that don't use `await`:

- `sendWelcomeMessage()` - now sync
- `sendAppointmentNotification()` - now sync
- `sendInstallationCompleteNotification()` - now sync
- `sendSms()` - now sync (placeholder, logs only)
- `sendEmail()` - now sync (placeholder, logs only)

**Note:** These are placeholder methods. Actual SMS/Email sending via `SmsService` and `EmailService` is NOT YET integrated.

---

### 28. NotificationResolver Async Fixes (Backend)

**File:** `/src/modules/notification/notification.resolver.ts`

Removed `async` from resolver methods to match service:

- `sendWelcomeMessage()` - now sync
- `sendAppointmentNotification()` - now sync
- `sendInstallationCompleteNotification()` - now sync

---

### 29. Contractor Name Field Added (Backend)

**File:** `/src/modules/contractors/contractors.types.ts`

Added `name` field to Contractor ObjectType:

```typescript
@Field({ nullable: true, description: 'Contractor name (from linked user)' })
name?: string;
```

**File:** `/src/modules/contractors/contractors.resolver.ts`

Added field resolver to fetch name from linked User record:

```typescript
@ResolveField(() => String, { nullable: true })
async name(@Parent() contractor: Contractor): Promise<string | null> {
  const user = await this.db
    .select({ name: users.name })
    .from(users)
    .where(eq(users.id, contractor.userId))
    .limit(1);
  return user[0]?.name || null;
}
```

---

### 30. Notification Module Updated (Backend)

**File:** `/src/modules/notification/notification.module.ts`

Added new services to providers and exports:

- `SmsService`
- `EmailService`

---

### 31. Dependencies Installed (Backend)

```bash
pnpm add nodemailer @aws-sdk/client-ses axios
pnpm add -D @types/nodemailer
```

---

---

### 32. Billing Module Removed from Frontend Codegen (Frontend)

**File:** `/smartmeter-frontend/src/graphql/billing.ts`

Commented out all queries and mutations - billing module was removed from backend:

```typescript
// BILLING MODULE REMOVED - These queries/mutations are no longer available in the backend
// The billing module has been removed and will be implemented later.
```

This fixes the 10 codegen errors that were blocking `pnpm codegen`.

---

### 33. Frontend Contractor Queries Updated (Frontend)

**File:** `/smartmeter-frontend/src/graphql/contractors.ts`

Added `name` field to all contractor queries:

- `GET_CONTRACTORS` - now includes `name` in response
- `GET_CONTRACTOR` - now includes `name` in response
- `GET_MY_CONTRACTOR` - now includes `name` in response

---

### 34. Frontend Types Updated (Frontend)

**File:** `/smartmeter-frontend/src/types/graphql.ts`

Added `name` field to Contractor interface:

```typescript
export interface Contractor {
  // ... existing fields
  name?: string; // Added - from linked user
}
```

---

### 35. SMS & Email Service Integration (Backend)

**File:** `/src/modules/notification/notification.service.ts`

Fully integrated `SmsService` and `EmailService`:

```typescript
// Now injects both services
constructor(
  private readonly smsService: SmsService,
  private readonly emailService: EmailService,
) {}

// sendSms() now calls SmsService when enabled
async sendSms(to: string, message: string, templateId?: string): Promise<void> {
  if (process.env.SMS_GATEWAY_ENABLED === 'true' && templateId) {
    await this.smsService.sendSMS(to, [/* variables extracted */], templateId);
  } else {
    console.log(`[SMS - DEV MODE] To: ${to}, Message: ${message}`);
  }
}

// sendEmail() now calls EmailService when enabled
async sendEmail(to: string, subject: string, body: string, html?: string): Promise<void> {
  if (process.env.EMAIL_ENABLED === 'true') {
    await this.emailService.sendMail({ sendTo: to, subject, body, html });
  } else {
    console.log(`[EMAIL - DEV MODE] To: ${to}, Subject: ${subject}`);
  }
}
```

**File:** `/src/modules/notification/notification.resolver.ts`

Made all mutations async to match service:

```typescript
async sendWelcomeMessage(...): Promise<NotificationResult>
async sendAppointmentNotification(...): Promise<NotificationResult>
async sendInstallationCompleteNotification(...): Promise<NotificationResult>
```

---

### 36. DLT Template Environment Variables (Backend)

**File:** `/src/modules/notification/notification.config.ts`

Added environment variables for DLT template IDs:

```typescript
templateIds: {
  otp: process.env.DLT_TEMPLATE_OTP,
  welcome: process.env.DLT_TEMPLATE_WELCOME,
  appointment: process.env.DLT_TEMPLATE_APPOINTMENT,
  installationComplete: process.env.DLT_TEMPLATE_INSTALLATION_COMPLETE,
  consumerAppLink: process.env.DLT_TEMPLATE_CONSUMER_APP_LINK,
  contractorAppLink: process.env.DLT_TEMPLATE_CONTRACTOR_APP_LINK,
  contractorPassword: process.env.DLT_TEMPLATE_CONTRACTOR_PASSWORD,
},
```

**File:** `.env.example`

Added DLT template placeholders:

```env
# DLT Template IDs (must be registered with TRAI)
DLT_TEMPLATE_OTP=
DLT_TEMPLATE_WELCOME=
DLT_TEMPLATE_APPOINTMENT=
DLT_TEMPLATE_INSTALLATION_COMPLETE=
DLT_TEMPLATE_CONSUMER_APP_LINK=
DLT_TEMPLATE_CONTRACTOR_APP_LINK=
DLT_TEMPLATE_CONTRACTOR_PASSWORD=
```

---

### 37. DLT Templates Documentation Created (Docs)

**File:** `/docs/DLT_TEMPLATES.md`

Comprehensive documentation for registering SMS templates with TRAI DLT:

**Templates included (7 total):**
| # | Template Name | Variables | Purpose |
|---|--------------|-----------|---------|
| 1 | OTP | `{#var1#}` | Login verification |
| 2 | Welcome | `{#var1#}`, `{#var2#}` | New consumer welcome |
| 3 | Appointment | `{#var1#}`, `{#var2#}` | Contractor scheduling |
| 4 | Installation Complete | `{#var1#}`, `{#var2#}` | Meter installed notification |
| 5 | Consumer App Link | `{#var1#}`, `{#var2#}` | Send app download link |
| 6 | Contractor App Link | `{#var1#}`, `{#var2#}` | Send app download link |
| 7 | Contractor Password | `{#var1#}`, `{#var2#}`, `{#var3#}` | New contractor credentials |

**Includes:**

- Registration process with DLT operator (Jio, Airtel, Vodafone, BSNL)
- Fast2SMS template approval process
- Environment variable configuration
- Testing checklist

---

### 38. API Documentation Updated (Docs)

**File:** `/docs/CONTRACTOR_API.md`

- Updated to version 3.1.0
- Added `name` field to Contractor type documentation

**File:** `/docs/CONSUMER_API.md`

- Updated to version 3.1.0
- Date updated to reflect current session

---

### 39. EmailService Lazy Initialization (Backend)

**File:** `/src/modules/notification/email.service.ts`

Fixed crash on startup when `EMAIL_ENABLED=false`:

- AWS SES client now initialized lazily (only when first email is sent)
- Prevents errors when AWS credentials are not configured

```typescript
private getTransporter(): nodemailer.Transporter {
  if (!this.transporter) {
    const sesClient = new SESClient({ region: process.env.AWS_REGION || 'ap-south-1' });
    this.transporter = nodemailer.createTransport({ SES: { ses: sesClient, aws: require('aws-sdk') } });
  }
  return this.transporter;
}
```

---

## Notification System Status Summary (UPDATED)

| Component              | Status        | Notes                              |
| ---------------------- | ------------- | ---------------------------------- |
| `NotificationService`  | ✅ Integrated | Now uses SmsService & EmailService |
| `NotificationResolver` | ✅ Working    | GraphQL mutations (async)          |
| `SmsService`           | ✅ Integrated | Fast2SMS DLT API                   |
| `EmailService`         | ✅ Integrated | AWS SES + Nodemailer               |

**Current Architecture:**

- DEV mode (`SMS_GATEWAY_ENABLED=false`, `EMAIL_ENABLED=false`): Logs to console
- PROD mode: Sends via Fast2SMS (SMS) and AWS SES (Email)
- DLT templates must be registered with TRAI before SMS production use

---

## Pending / TODO

### Completed This Session ✅

- [x] Integrate SmsService into NotificationService
- [x] Integrate EmailService into NotificationService
- [x] Create DLT_TEMPLATES.md documentation
- [x] Update frontend contractor queries with `name` field
- [x] Update frontend types with `name` field
- [x] Fix billing.ts codegen errors (commented out)
- [x] Update CONTRACTOR_API.md and CONSUMER_API.md

### Still Pending

- [ ] Run frontend codegen: `cd smartmeter-frontend && pnpm codegen` (requires backend running)
- [ ] Register DLT templates with TRAI (external, 1-7 days approval)
- [ ] Set up AWS SES and verify domain (external)
- [ ] Get Fast2SMS API key and configure (external)
- [ ] Run migration for `auth_throttle` table if not done
- [ ] Run database migration for new schema changes

---

## Session: 12 February 2026

### 40. Refresh Tokens Moved to Separate Table (Backend)

**New File:** `/src/database/schema/refresh-tokens.ts`

Moved refresh tokens from `users` table to dedicated `refresh_tokens` table:

```typescript
export const refreshTokens = pgTable("refresh_tokens", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  token: varchar("token", { length: 512 }).notNull(),
  expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true })
    .notNull()
    .defaultNow(),
  userAgent: varchar("user_agent", { length: 500 }),
  ipAddress: varchar("ip_address", { length: 45 }),
});
```

**Benefits:**

- Multiple sessions per user (different devices)
- Easier token revocation
- Better security auditing
- Users table is cleaner

**Files Modified:**

- `/src/database/schema/users.ts` - Removed `refreshToken` and `refreshTokenExpiresAt` columns
- `/src/database/schema/index.ts` - Added export for `refresh-tokens`
- `/src/modules/auth/auth.service.ts` - Updated to use new table

---

### 41. Delayed Consumer Registration Flow (Backend)

**Problem:** Previously, when admin imported consumers, user entries were created immediately, allowing consumers to login before RO review.

**Solution:** Consumer registration is now delayed until RO activates the site.

#### Schema Changes

**File:** `/src/database/schema/consumers.ts`

```typescript
// consumers table
userId: uuid('user_id').references(() => users.id)  // Now NULLABLE
// NULL = Not yet activated (can't login)
// Has value = Activated (can login)

// consumer_sites table - NEW FIELDS
isActivated: boolean('is_activated').notNull().default(false),
activatedAt: timestamp('activated_at', { withTimezone: true }),
activatedBy: uuid('activated_by').references(() => users.id),
```

#### Import Flow Changed

**File:** `/src/modules/support/imports.resolver.ts`

```
BEFORE:
  Import → Create users + consumers + sites (all at once)
  Consumer can login immediately ❌

AFTER:
  Import → Create ONLY consumers + sites (userId = NULL)
  Consumer cannot login until activated ✓
```

#### New Activation Mutation

**File:** `/src/modules/sites/sites.resolver.ts`

```graphql
mutation ActivateConsumerSite($input: ActivateConsumerSiteInput!) {
  activateConsumerSite(input: $input) {
    success
    message
    site {
      id
      siteId
      isActivated
    }
    userCreated # true if first activation
    smsSent # true if SMS sent
  }
}
```

**Flow:**

1. RO reviews site
2. RO assigns contractor
3. RO clicks "Send Link" → calls `activateConsumerSite`
4. If consumer.userId is NULL:
   - Create user entry in `users` table
   - Link to consumer
   - Send registration SMS
5. Set site.isActivated = true
6. Consumer can now login via OTP

#### Auth Service Updated

**File:** `/src/modules/auth/auth.service.ts`

OTP request now checks:

1. Consumer exists by phone
2. Consumer has userId (activated)
3. User is active

```typescript
async requestOtp(phone: string) {
  const consumer = await findConsumerByPhone(phone);

  if (!consumer) {
    throw new BadRequestException('Phone number not registered');
  }

  if (!consumer.userId) {
    throw new BadRequestException(
      'Your account is not yet activated. You will receive an SMS when your site is ready.'
    );
  }

  // Continue with OTP...
}
```

#### Consumer Sites Query Updated

**File:** `/src/modules/sites/sites.service.ts`

New method `findActivatedSitesByUserId()` - returns only activated sites for consumer.

```typescript
async findActivatedSitesByUserId(userId: string): Promise<ConsumerSite[]> {
  return db.select()
    .from(consumerSites)
    .innerJoin(consumers, eq(consumers.id, consumerSites.consumerId))
    .where(
      and(
        eq(consumers.userId, userId),
        eq(consumerSites.isActivated, true),  // Only activated sites
      ),
    );
}
```

---

### 42. DLT Templates Simplified (Docs)

**File:** `/docs/DLT_TEMPLATES.md`

Reduced from 7 templates to 2:

| #   | Template     | Status     | Purpose                                        |
| --- | ------------ | ---------- | ---------------------------------------------- |
| 1   | OTP          | ✅ Active  | Login verification (ID: `1107171707202035772`) |
| 2   | Registration | 🔄 Pending | Welcome message for consumer/contractor        |

**Reason:** Both consumer and contractor use OTP-based login (no password), so one registration template suffices.

---

## Flow Summary: Consumer Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. IMPORT PHASE                                                      │
│    Admin uploads Excel                                               │
│    → consumers table: Created (userId = NULL)                        │
│    → consumer_sites: Created (isActivated = FALSE)                   │
│    → users table: NOTHING                                            │
│                                                                      │
│    Consumer tries to login → "Account not yet activated"             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 2. RO REVIEW PHASE                                                   │
│    RO views site → Verifies details → Assigns contractor             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 3. ACTIVATION PHASE                                                  │
│    RO clicks "Send Link" → activateConsumerSite mutation             │
│                                                                      │
│    IF first activation:                                              │
│    → users table: Create entry                                       │
│    → consumers.userId: Set to new user ID                            │
│    → Send registration SMS                                           │
│                                                                      │
│    ALWAYS:                                                           │
│    → consumer_sites.isActivated = TRUE                               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 4. CONSUMER LOGIN PHASE                                              │
│    Consumer requests OTP → Success (userId exists)                   │
│    Consumer logs in → Sees ONLY activated sites                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Database Migration Required

Run these commands after pulling changes:

```bash
cd smartmeter-backend

# Generate migration
pnpm drizzle-kit generate --name delayed_consumer_registration

# Review the generated SQL in /drizzle folder

# Apply migration
pnpm drizzle-kit push
```

**Tables affected:**

- `users` - Remove `refresh_token`, `refresh_token_expires_at` columns
- `refresh_tokens` - New table
- `consumer_sites` - Add `is_activated`, `activated_at`, `activated_by` columns
- `consumers.user_id` - Make nullable (if not already)

---

## Pending / TODO

### Completed This Session ✅

- [x] Move refresh tokens to separate table
- [x] Make consumers.userId nullable
- [x] Add isActivated to consumer_sites
- [x] Update import resolver - don't create users
- [x] Update auth service - check userId before OTP
- [x] Create activateConsumerSite mutation
- [x] Update mySites query - filter by isActivated
- [x] Simplify DLT templates (7 → 2)

### Still Pending

- [ ] Run database migration
- [ ] Run frontend codegen
- [ ] Register DLT registration template with TRAI
- [ ] Integrate NotificationService with activateConsumerSite

---

### 43. OTP Login with loginType for Consumer & Contractor (Backend + Frontend)

**Problem:** OTP login (`requestOtp`, `verifyOtp`) only checked the `consumers` table. Contractors couldn't use OTP login even though frontend expected it.

**Solution:** Added `loginType` parameter to OTP mutations to support both Consumer and Contractor.

#### Backend Changes

**File:** `/src/modules/auth/auth.types.ts`

```typescript
@InputType()
export class RequestOtpInput {
  @Field()
  phone: string;

  @Field(() => LoginType)
  loginType: LoginType; // NEW - CONSUMER or CONTRACTOR
}

@InputType()
export class VerifyOtpInput {
  @Field()
  phone: string;

  @Field()
  otp: string;

  @Field(() => LoginType)
  loginType: LoginType; // NEW - CONSUMER or CONTRACTOR
}
```

**File:** `/src/modules/auth/auth.service.ts`

```typescript
async requestOtp(phone: string, loginType: LoginType) {
  if (loginType === LoginType.CONSUMER) {
    // Check consumers table
    // Verify consumer.userId exists (activated)
  } else if (loginType === LoginType.CONTRACTOR) {
    // Check contractors table
    // Verify contractor.userId exists
  }
  // Generate and store OTP in users table
}

async verifyOtp(phone: string, otp: string, loginType: LoginType) {
  // Get userId from consumers or contractors table based on loginType
  // Verify OTP against users table
}
```

#### Frontend Changes

**File:** `/src/graphql/auth.ts`

```graphql
mutation RequestOtp($phone: String!, $loginType: LoginType!) {
  requestOtp(input: { phone: $phone, loginType: $loginType }) {
    success
    message
  }
}

mutation VerifyOtp($phone: String!, $otp: String!, $loginType: LoginType!) {
  verifyOtp(input: { phone: $phone, otp: $otp, loginType: $loginType }) {
    accessToken
    refreshToken
    user {
      id
      name
      role
    }
  }
}
```

**File:** `/src/app/(auth)/login/page.tsx`

```typescript
const onRequestOtp = async (data: OtpFormData) => {
  const loginType =
    selectedRole === "consumer" ? LoginType.CONSUMER : LoginType.CONTRACTOR;
  await requestOtp({ variables: { phone: data.phone, loginType } });
};

const onVerifyOtp = async (data: VerifyOtpFormData) => {
  const loginType =
    selectedRole === "consumer" ? LoginType.CONSUMER : LoginType.CONTRACTOR;
  await verifyOtp({
    variables: { phone: phoneNumber, otp: data.otp, loginType },
  });
};
```

#### Login Flow Summary

| Role       | Login Method     | Table Checked           | How                                                     |
| ---------- | ---------------- | ----------------------- | ------------------------------------------------------- |
| ADMIN      | Email + Password | `users`                 | `login` mutation with `loginType: ADMIN`                |
| RO         | Email + Password | `users`                 | `login` mutation with `loginType: RO`                   |
| CONSUMER   | Phone + OTP      | `consumers` → `users`   | `requestOtp` + `verifyOtp` with `loginType: CONSUMER`   |
| CONTRACTOR | Phone + OTP      | `contractors` → `users` | `requestOtp` + `verifyOtp` with `loginType: CONTRACTOR` |

---

## Pending / TODO

### Completed This Session ✅

- [x] Move refresh tokens to separate table
- [x] Delayed consumer registration (userId nullable, isActivated field)
- [x] Update import resolver - don't create users
- [x] Update auth service - check userId before OTP
- [x] Create activateConsumerSite mutation
- [x] Update mySites query - filter by isActivated
- [x] Simplify DLT templates (7 → 2)
- [x] Add loginType to OTP mutations (backend + frontend)
- [x] **OPTIMIZED:** Separate login OTP mutations (requestLoginOtp, verifyLoginOtp)
- [x] Run database migration
- [x] Run frontend codegen
- [x] Simplify seed.ts (super admin only from env)
- [x] Update CONSUMER_API.md and CONTRACTOR_API.md

### Still Pending - External

- [ ] Register DLT registration template with TRAI

### Still Pending - Backend

- [ ] Integrate NotificationService with activateConsumerSite (send SMS on activation)
- [ ] Add consumer registration endpoint (for SMS link to complete registration)

### Still Pending - Frontend

- [ ] **RO Dashboard - "Send Link" Button:** Add UI for RO to activate consumer sites
  - Location: Sites list / Site detail page
  - Call `activateConsumerSite` mutation
  - Show success message with SMS status
- [ ] **Consumer Registration Page:** Create page for consumers to complete registration
  - Route: `/register?token=xxx` (from SMS link)
  - Verify phone, set password (optional), confirm details
- [ ] **Site Activation Status:** Show `isActivated` status in RO site list
  - Badge: "Pending Activation" vs "Activated"
  - Filter sites by activation status
- [ ] **Update Site List UI:** Add activation-related columns/filters
  - `activatedAt` timestamp
  - `activatedBy` user info

---

### 29. Optimized OTP Login - Separate Mutations

#### Overview

Created dedicated login OTP mutations that query `users` table directly by phone + role (single optimized query), while keeping original `requestOtp`/`verifyOtp` for general verification purposes.

#### Why This Optimization?

The previous implementation queried `consumers` or `contractors` table first to get `userId`, then queried `users` table. The optimized version directly queries `users` table with `phone` + `role` in a single query.

#### Backend Changes

**File:** `/src/modules/auth/auth.types.ts`

```typescript
// General verification (password reset, etc.) - simple, no loginType
@InputType()
export class RequestOtpInput {
  @Field()
  phone: string;
}

@InputType()
export class VerifyOtpInput {
  @Field()
  phone: string;

  @Field()
  otp: string;
}

// LOGIN OTP - with loginType for role verification
@InputType()
export class RequestLoginOtpInput {
  @Field()
  phone: string;

  @Field(() => LoginType)
  loginType: LoginType; // ADMIN, RO, CONSUMER, CONTRACTOR
}

@InputType()
export class VerifyLoginOtpInput {
  @Field()
  phone: string;

  @Field()
  otp: string;

  @Field(() => LoginType)
  loginType: LoginType;
}
```

**File:** `/src/modules/auth/auth.service.ts`

```typescript
// General verification - simple
async requestOtp(phone: string) {
  const user = await db.select().from(users)
    .where(eq(users.phone, phone)).limit(1);
  // Generate OTP...
}

async verifyOtp(phone: string, otp: string) {
  // Returns { success, message } - not AuthPayload
}

// LOGIN OTP - optimized single query with role
async requestLoginOtp(phone: string, loginType: LoginType) {
  const expectedRole = LOGIN_TYPE_TO_ROLE[loginType];
  // Single query - users table directly
  const user = await db.select().from(users)
    .where(and(
      eq(users.phone, phone),
      eq(users.role, expectedRole)
    )).limit(1);
  // Generate OTP...
}

async verifyLoginOtp(phone: string, otp: string, loginType: LoginType) {
  const expectedRole = LOGIN_TYPE_TO_ROLE[loginType];
  // Single query with phone + role + otp
  const user = await db.select().from(users)
    .where(and(
      eq(users.phone, phone),
      eq(users.role, expectedRole),
      eq(users.otpCode, otp),
      gt(users.otpExpiresAt, new Date())
    )).limit(1);
  // Returns user for token generation
}
```

**File:** `/src/modules/auth/auth.resolver.ts`

```typescript
// General verification (NOT login)
@Mutation(() => OtpResponse)
async requestOtp(@Args('input') input: RequestOtpInput) {
  return this.authService.requestOtp(input.phone);
}

@Mutation(() => OtpResponse)
async verifyOtp(@Args('input') input: VerifyOtpInput) {
  return this.authService.verifyOtp(input.phone, input.otp);
}

// LOGIN OTP
@Mutation(() => OtpResponse)
async requestLoginOtp(@Args('input') input: RequestLoginOtpInput) {
  return this.authService.requestLoginOtp(input.phone, input.loginType);
}

@Mutation(() => AuthPayload)
async verifyLoginOtp(@Args('input') input: VerifyLoginOtpInput) {
  const user = await this.authService.verifyLoginOtp(input.phone, input.otp, input.loginType);
  return this.authService.login(user);
}
```

#### Frontend Changes

**File:** `/src/graphql/auth.ts`

```typescript
// General verification (NOT for login)
export const REQUEST_OTP_MUTATION = gql`
  mutation RequestOtp($phone: String!) {
    requestOtp(input: { phone: $phone }) {
      success
      message
    }
  }
`;

export const VERIFY_OTP_MUTATION = gql`
  mutation VerifyOtp($phone: String!, $otp: String!) {
    verifyOtp(input: { phone: $phone, otp: $otp }) {
      success
      message
    }
  }
`;

// LOGIN OTP
export const REQUEST_LOGIN_OTP_MUTATION = gql`
  mutation RequestLoginOtp($phone: String!, $loginType: LoginType!) {
    requestLoginOtp(input: { phone: $phone, loginType: $loginType }) {
      success
      message
    }
  }
`;

export const VERIFY_LOGIN_OTP_MUTATION = gql`
  mutation VerifyLoginOtp(
    $phone: String!
    $otp: String!
    $loginType: LoginType!
  ) {
    verifyLoginOtp(input: { phone: $phone, otp: $otp, loginType: $loginType }) {
      accessToken
      refreshToken
      user {
        id
        name
        role
        phone
      }
    }
  }
`;
```

**File:** `/src/app/(auth)/login/page.tsx`

```typescript
import {
  REQUEST_LOGIN_OTP_MUTATION,
  VERIFY_LOGIN_OTP_MUTATION,
  LoginType,
} from "@/graphql/auth";

const [requestLoginOtp] = useMutation(REQUEST_LOGIN_OTP_MUTATION);
const [verifyLoginOtp] = useMutation(VERIFY_LOGIN_OTP_MUTATION);

const onRequestOtp = async (data) => {
  const loginType =
    selectedRole === "consumer" ? LoginType.CONSUMER : LoginType.CONTRACTOR;
  const result = await requestLoginOtp({
    variables: { phone: data.phone, loginType },
  });
  if (result.data?.requestLoginOtp?.success) {
    /* ... */
  }
};

const onVerifyOtp = async (data) => {
  const loginType =
    selectedRole === "consumer" ? LoginType.CONSUMER : LoginType.CONTRACTOR;
  const result = await verifyLoginOtp({
    variables: { phone: phoneNumber, otp: data.otp, loginType },
  });
  if (result.data?.verifyLoginOtp?.accessToken) {
    loginWithToken(
      result.data.verifyLoginOtp.accessToken,
      result.data.verifyLoginOtp.user,
    );
  }
};
```

#### Updated Login Flow Summary

| Role       | Login Method     | Mutations Used                       | Query Pattern              |
| ---------- | ---------------- | ------------------------------------ | -------------------------- |
| ADMIN      | Email + Password | `login`                              | `users` WHERE email + role |
| ADMIN      | Phone + OTP      | `requestLoginOtp` + `verifyLoginOtp` | `users` WHERE phone + role |
| RO         | Email + Password | `login`                              | `users` WHERE email + role |
| RO         | Phone + OTP      | `requestLoginOtp` + `verifyLoginOtp` | `users` WHERE phone + role |
| CONSUMER   | Phone + OTP      | `requestLoginOtp` + `verifyLoginOtp` | `users` WHERE phone + role |
| CONTRACTOR | Phone + OTP      | `requestLoginOtp` + `verifyLoginOtp` | `users` WHERE phone + role |

**Key Benefit:** All roles now use the same optimized query pattern - single query to `users` table with `phone` + `role` filter.
