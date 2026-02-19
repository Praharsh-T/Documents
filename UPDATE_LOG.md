# MESCOM Smart Meter System - Update Log

**Last Updated:** 19 February 2026 (Backend Restructuring & Access Control Fixes)

---

## 📋 Latest Session Context (19 Feb 2026 - Backend Restructuring)

### What Was Done This Session:

#### **1. BUG FIXES**

**marketplaceJobs Validation Error:**
- Fixed `JobsFilterInput` and `SlaBreachFilterInput` - added `@IsOptional()` to `page` and `limit` fields
- Error was: `property page should not exist, property limit should not exist`

**SUPER_ADMIN Associations Access:**
- Added `UserRole.SUPER_ADMIN` to all association query decorators
- SUPER_ADMIN can now query associations (required for admin creation)

---

#### **2. ACCESS CONTROL UPDATES**

**Admin Creation - Association Now Mandatory:**
- When SUPER_ADMIN creates an admin, association selection is REQUIRED
- Shows validation error if no association selected
- Shows warning if no associations exist

**SUPER_ADMIN Marketplace Restriction:**
- SUPER_ADMIN only sees marketplace stats dashboard (not management UI)
- All marketplace management pages (locations, services, pricing, SLA, jobs, contractors, categories, UOMs) show "Access Restricted" for SUPER_ADMIN
- Only ADMIN role can manage marketplace features

---

#### **3. ASSOCIATIONS MODULE - EXTRACTED FROM MARKETPLACE**

**New Standalone Module Created:**
```
/modules/associations/
  ├── associations.module.ts
  ├── associations.service.ts
  ├── associations.resolver.ts
  ├── associations.types.ts
  └── index.ts
```

**New Database Tables:**
- `associations` (was `mp_admin_associations`)
- `association_states` (was `mp_admin_association_states`)
- `association_users` (was `mp_admin_association_users`)

**New GraphQL API (cleaner names):**
- `associations` / `association` / `myAssociation`
- `createAssociation` / `updateAssociation`
- `assignStatesToAssociation` / `removeStateFromAssociation`
- `addUserToAssociation` / `removeUserFromAssociation`

---

#### **4. LOCATION HIERARCHY - NEW SCHEMA**

**Old Structure (flat):**
```
mp_location_master (single table with repeated state/district names)
```

**New Hierarchical Structure:**
```
service_states
  └── service_districts (FK → state)
      └── service_sub_districts (FK → district)
          └── service_villages (FK → sub_district)
```

**Benefits:**
- No data duplication
- Proper foreign key relationships
- Better query performance for hierarchy navigation
- Easier data integrity

**Migration:** `0009_service_location_hierarchy.sql` creates tables and migrates existing data

---

## 📋 Previous Session Context (18 Feb 2026 - Admin Module Completion)
- `VillageOption` - Added required `category: string` field

**Backend Service Changes (`locations.service.ts`):**
- Updated validation: Now requires `villageCategory` in import
- Error message: `"Row X: Missing required fields (villageCode, villageName, stateCode, districtCode, villageCategory)"`
- Updated `getVillages()` to include `category` field in response
- Updated `mapToDto()` to handle villageCategory with fallback to 'Unknown'

**Frontend Changes:**
- `useMarketplaceLocations.ts` - Added `category: string` to `VillageOption` interface
- `marketplace.ts` GraphQL - Added `category` to `GET_MARKETPLACE_VILLAGES` query
- Villages now display category badge in the locations browser

---

### ✅ COMPLETED COMPONENTS (This Session)

| Component | Location | Status |
|-----------|----------|--------|
| Admin Associations SUPER_ADMIN restriction | Backend resolver | ✅ Done |
| Associations UI removed from Admin | Frontend pages/hooks | ✅ Done |
| Admin creation with Association dropdown | `/admin/admins/page.tsx` | ✅ Done |
| Location import moved to Marketplace | `/admin/marketplace/locations/page.tsx` | ✅ Done |
| Location import removed from Upload | `/admin/upload/page.tsx` | ✅ Done |
| Village Category made required | Backend types & service | ✅ Done |
| Village Category in frontend | Hooks & GraphQL | ✅ Done |

---

### ✅ COMPLETED COMPONENTS (Previous Sessions)

#### **Backend (NestJS + GraphQL)**

**Database Schemas (9 files):**
- `marketplace-locations.ts` - Location master with LGD hierarchy & area classification
- `marketplace-services.ts` - Service catalog
- `marketplace-pricing.ts` - Versioned pricing per service + area type
- `marketplace-slas.ts` - Versioned SLA configurations
- `marketplace-mappings.ts` - Contractor-location & contractor-service mappings
- `marketplace-subscriptions.ts` - Contractor subscription tracking
- `marketplace-jobs.ts` - Core job entity with state machine
- `marketplace-otps.ts` - OTP management for job actions
- `marketplace-ratings.ts` - Dual rating system (consumer ↔ contractor)

**Type Files (9 files):**
- GraphQL types for all marketplace entities
- Input types for mutations
- Enums: `MarketplaceAreaType`, `ServiceCategory`, `UnitOfMeasure`, `JobStatus`, `OtpType`, `SubscriptionType`

**Service Files (8 files):**
- `locations.service.ts` - Excel import, area classification, search
- `services.service.ts` - CRUD, availability checks
- `pricing.service.ts` - Versioned pricing resolution
- `slas.service.ts` - SLA resolution & deadline calculation
- `contractor-extension.service.ts` - Marketplace profile, location/service mappings
- `subscriptions.service.ts` - FREE/PREMIUM subscription management
- `jobs.service.ts` - Full job lifecycle with state machine
- `ratings.service.ts` - Rating submission & average calculation

**Resolver Files (8 files):**
- All GraphQL queries and mutations for each service

---

#### **Frontend (Next.js + Apollo Client)**

**GraphQL Operations (`marketplace.ts`):**
- ~35 queries and mutations matching backend schema
- Full type generation via codegen

**Custom Hooks (6 files):**
- `useMarketplaceServices.ts` - Service listing
- `useMarketplaceJobs.ts` - Job management
- `useMarketplaceContractors.ts` - Contractor search
- `useMarketplaceLocations.ts` - Location management
- `useMarketplaceSLA.ts` - SLA status tracking
- `useMarketplaceRatings.ts` - Rating submission

**Shared Components (10 files):**
- `ServiceCard.tsx`, `PriceDisplay.tsx`, `SLAIndicator.tsx`
- `JobTimeline.tsx`, `OTPModal.tsx`, `RatingStars.tsx`
- `SubscriptionBadge.tsx`, `AreaTypeBadge.tsx`
- `ContractorCard.tsx`, `JobStatusBadge.tsx`

**Consumer Pages (5 pages):**
- `/consumer/marketplace/page.tsx` - Services listing
- `/consumer/marketplace/service/[id]/page.tsx` - Service detail & booking
- `/consumer/marketplace/contractors/page.tsx` - Contractor search
- `/consumer/marketplace/jobs/page.tsx` - My jobs
- `/consumer/marketplace/jobs/[id]/page.tsx` - Job tracking

**Contractor Pages (4 pages):**
- `/contractor/marketplace/page.tsx` - Dashboard
- `/contractor/marketplace/jobs/page.tsx` - Job list with SLA indicators
- `/contractor/marketplace/jobs/[id]/page.tsx` - Job detail with OTP verification
- `/contractor/marketplace/profile/page.tsx` - Ratings & performance

**Admin Pages (5 pages):**
- `/admin/marketplace/page.tsx` - Management dashboard
- `/admin/marketplace/services/page.tsx` - Service CRUD
- `/admin/marketplace/pricing/page.tsx` - Pricing matrix editor
- `/admin/marketplace/sla/page.tsx` - SLA configuration
- `/admin/marketplace/locations/page.tsx` - 4-column location hierarchy browser

---

### ⏳ PENDING COMPONENTS (Payment Module)

#### **Backend - Payment Integration (Cashfree)**

**Files to Create:**
1. `src/modules/marketplace/payments/payments.module.ts`
2. `src/modules/marketplace/payments/payments.service.ts`
   - `createPaymentOrder()` - Create Cashfree order
   - `verifyPayment()` - Verify webhook signature
   - `handlePaymentSuccess()` - Update job status
   - `handlePaymentFailure()` - Handle failures
3. `src/modules/marketplace/payments/payments.controller.ts`
   - REST endpoint for Cashfree webhook (`POST /api/payments/webhook`)
4. `src/modules/marketplace/payments/payments.resolver.ts`
5. `src/modules/marketplace/payments/payments.types.ts`

**Refund Handling:**
1. `src/modules/marketplace/refunds/refunds.service.ts`
   - Auto-refund on contractor rejection
   - Auto-refund on consumer cancellation
   - Cashfree refund API integration

**Database Schema:**
- `marketplace-payments.ts` - Already created, needs service implementation
- `marketplace-refunds.ts` - Refund tracking table

#### **Backend - Scheduler Service**

**Files to Create:**
1. `src/modules/marketplace/scheduler/marketplace-scheduler.module.ts`
2. `src/modules/marketplace/scheduler/marketplace-scheduler.service.ts`
   - `@Cron('*/5 * * * *')` - SLA breach detection
   - `@Cron('0 * * * *')` - Subscription expiry check
   - `@Cron('*/15 * * * *')` - Refund status polling

#### **Frontend - Payment Flow**

**Components to Create:**
1. `PaymentPage.tsx` - Cashfree payment redirect
2. `PaymentStatusPage.tsx` - Success/failure display
3. `RefundStatusBadge.tsx` - Refund tracking display

**Integration:**
- Cashfree JS SDK integration
- Payment return URL handling

---

### 📋 ADMIN WORKFLOW (Data Management)

Since no seed data was created, admin will manage all data via UI:

1. **Locations** → `/admin/marketplace/locations`
   - Upload LGD Excel with villages
   - Classify villages as URBAN/SEMI_URBAN/RURAL

2. **Services** → `/admin/marketplace/services`
   - Create service catalog
   - Set premium-only flags

3. **Pricing** → `/admin/marketplace/pricing`
   - Set prices per service + area type
   - Version control with effective dates

4. **SLAs** → `/admin/marketplace/sla`
   - Configure acceptance/start/completion hours
   - Version control with effective dates

---

### 🔧 ENVIRONMENT VARIABLES NEEDED

```env
# Cashfree Payment Gateway (add when implementing payments)
CASHFREE_APP_ID=your_app_id
CASHFREE_SECRET_KEY=your_secret_key
CASHFREE_API_VERSION=2023-08-01
CASHFREE_ENV=TEST
CASHFREE_WEBHOOK_SECRET=your_webhook_secret
CASHFREE_RETURN_URL=https://yourapp.com/payment/return
CASHFREE_NOTIFY_URL=https://yourapi.com/api/payments/webhook

# Scheduler
CRON_SLA_CHECK=*/5 * * * *
CRON_SUBSCRIPTION_CHECK=0 * * * *
CRON_REFUND_CHECK=*/15 * * * *
```

---

### 🧪 TESTING CHECKLIST

**Backend Tests:**
- [ ] Start backend: `cd /Users/praharsh/Projects/user-service && pnpm start:dev`
- [ ] Run migrations if needed: `pnpm drizzle-kit push`
- [ ] Test GraphQL playground: `http://localhost:3001/graphql`

**Frontend Tests:**
- [ ] Start frontend: `cd /Users/praharsh/Projects/web-frontend && pnpm dev`
- [ ] Test consumer marketplace flow
- [ ] Test contractor job management
- [ ] Test admin configuration pages

**Key Flows to Test:**
1. Admin creates locations, services, pricing, SLAs
2. Consumer searches services in their area
3. Consumer views contractor list and profiles
4. Consumer creates job request
5. Contractor accepts/rejects job
6. Contractor generates OTP, starts job
7. Contractor completes job with OTP
8. Consumer rates contractor

---

### 📁 FILE LOCATIONS

**Backend:** `/Users/praharsh/Projects/user-service/src/modules/marketplace/`
**Frontend:** `/Users/praharsh/Projects/web-frontend/src/`
- Pages: `app/(consumer|contractor|admin)/*/marketplace/`
- Components: `components/marketplace/`
- Hooks: `hooks/marketplace/`
- GraphQL: `graphql/marketplace.ts`

---

## 📋 Previous Session Context (12 Feb 2026 - Consumer Import & Authentication Fixes)

### What Was Done This Session:

#### **PART 1: Database Seed Simplification**
- Simplified seed to only create Super Admin from environment variables
- Removed all test data (consumers, sites, meters, contractors)
- Super Admin credentials now from: `SUPER_ADMIN_EMAIL`, `SUPER_ADMIN_PASSWORD`, `SUPER_ADMIN_NAME`, `SUPER_ADMIN_PHONE`

#### **PART 2: Authentication Fixes**

1. **Super Admin Login Fix:**
   - Fixed `LOGIN_TYPE_TO_ROLE` mapping to allow SUPER_ADMIN to login via ADMIN loginType
   - Added `or()` import from drizzle-orm for role lookup

2. **ValidationPipe Fixes:**
   - Added `@IsEnum(LoginType)` decorators to `loginType` fields in `LoginInput`, `RequestLoginOtpInput`, `VerifyLoginOtpInput`
   - Added `@IsString()` and `@IsNotEmpty()` decorators to `ImportFileInput.fileKey`
   - Required because backend uses `forbidNonWhitelisted: true`

3. **CORS Configuration:**
   - Updated backend CORS with explicit origins
   - Changed frontend Apollo Client credentials to `'same-origin'`

#### **PART 3: File Upload Fixes**

1. **Admin Upload Page:**
   - Fixed storageKey extraction: `uploadResult.files?.[0]?.storageKey` (was reading from wrong path)

2. **Import File Input:**
   - Added proper class-validator decorators to `fileKey` field

#### **PART 4: Consumer Import - Hierarchy Resolution (CRITICAL)**

**Problem:** Excel import was storing Circle/Division/Subdivision names as address fields instead of resolving to actual hierarchy IDs.

**Solution:**

1. **Updated `findSubdivisionByHierarchy`** in hierarchy.service.ts:
   - Return type changed to `(Subdivision & { circleId: string }) | null`
   - Added `circleId: circles.id` to the select query
   - Now returns circleId along with subdivision data

2. **Updated `findOrCreateHierarchy`** in hierarchy.service.ts:
   - Return type changed to `(Subdivision & { circleId: string }) | null`
   - Returns `{ ...subdivision, circleId: circle.id }` when creating new hierarchy

3. **Updated `imports.resolver.ts`:**
   - Added `ResolvedHierarchy` interface with `circleId`, `divisionId`, `subdivisionId`
   - Updated `PreparedConsumerRow` to include all three hierarchy IDs
   - Updated `resolveHierarchiesBatch` to return full hierarchy info
   - Updated `bulkInsertConsumersWithTransaction` to insert all three hierarchy IDs

4. **Fixed Drizzle null handling:**
   - Changed from passing `userId: null` to not including `userId` field at all
   - Let PostgreSQL use column defaults for nullable fields

#### **PART 5: Import Error Handling**

1. **User-friendly error messages:**
   - "One or more Consumer IDs already exist in the system" (duplicate consumer_id)
   - "One or more phone numbers already exist in the system" (duplicate phone)
   - "Invalid data format in one or more fields" (type errors)
   - "Invalid hierarchy reference" (foreign key violations)

2. **Fixed DrizzleQueryError parsing:**
   - Actual PostgreSQL error is in `error.cause`, not `error.message`
   - Now correctly extracts `pgError.detail` and `pgError.code`

#### **PART 6: Frontend Build Fixes**

1. **Added stub GraphQL queries** to `billing.ts`:
   - `GET_BILLING_SUMMARY` - returns `__typename` only
   - `GET_BILLING_EXPORTS` - returns `__typename` only
   - These are placeholders until billing module is fully implemented

---

## ✅ COMPLETED FEATURES

### Authentication & Authorization
- [x] Email/Password login for Admin/Super Admin via `login` mutation
- [x] Phone + OTP login for all roles via `requestLoginOtp`/`verifyLoginOtp`
- [x] Super Admin can login with ADMIN loginType
- [x] JWT token-based authentication
- [x] Role-based access control (RBAC)

### User Management
- [x] Super Admin creation from environment variables
- [x] Admin user management
- [x] RO (Retail Outlet) user management
- [x] Contractor user management
- [x] Consumer management

### Hierarchy Management
- [x] Circle → Division → Subdivision structure
- [x] RO assignment to subdivisions
- [x] Auto-create hierarchy during import if not exists

### Import System
- [x] Consumer Excel import with hierarchy resolution
- [x] Meter Excel import
- [x] Contractor Excel import
- [x] SHA-256 file hash to prevent duplicate imports
- [x] Transaction-based (all-or-nothing) import
- [x] User-friendly error messages
- [x] Import logging and audit trail

### File Management
- [x] Private file storage (MinIO/S3)
- [x] Public file storage
- [x] Signed URLs for private files
- [x] Installation evidence uploads

### Installation Flow
- [x] Site creation from consumer import
- [x] Meter assignment to sites
- [x] Contractor job assignment
- [x] Installation evidence upload
- [x] RO verification workflow
- [x] State machine for site/meter/installation states

### Frontend
- [x] Admin dashboard
- [x] RO dashboard with consumer list
- [x] RO verification page
- [x] Contractor job list and installation page
- [x] Consumer registration page
- [x] File upload pages

---

## 🔄 PENDING / TODO

### High Priority
- [ ] **SMS Gateway Integration** - Currently disabled (`SMS_SERVICES_DISABLED=true`)
- [ ] **Email Service Integration** - Currently disabled (`EMAIL_ENABLED=false`)
- [ ] **Consumer Activation Flow** - RO "Send Link" button needs to trigger actual SMS/Email
- [ ] **Skip Duplicates Option** - Allow import to skip existing consumers instead of failing entire batch

### Medium Priority
- [ ] **Billing Module** - Full implementation (currently has stub queries)
- [ ] **Notification System** - In-app notifications
- [ ] **Dashboard Statistics** - Proper stats queries for all dashboards
- [ ] **Export Functionality** - Export consumers/meters/reports to Excel

### Low Priority
- [ ] **Audit Log UI** - View audit logs in admin panel
- [ ] **Consumer Self-Service Portal** - Bill viewing, complaint registration
- [ ] **Mobile App** - React Native app for contractors
- [ ] **Reports & Analytics** - Detailed reports with charts

### Known Issues
- [ ] Billing queries return placeholder data only
- [ ] Some dashboard stats may show incorrect counts

---

## 📝 Environment Variables Required

```env
# Database
DATABASE_URL=postgresql://user:pass@host:5432/db

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=7d

# Super Admin (created on seed)
SUPER_ADMIN_EMAIL=admin@example.com
SUPER_ADMIN_PASSWORD=securepassword
SUPER_ADMIN_NAME=Super Admin
SUPER_ADMIN_PHONE=9999999999

# File Storage (MinIO/S3)
STORAGE_ACCESS_KEY=...
STORAGE_SECRET_KEY=...
STORAGE_BUCKET=smartmeter
STORAGE_ENDPOINT=...
STORAGE_REGION=...

# Services (disable for development)
SMS_SERVICES_DISABLED=true
SMS_GATEWAY_ENABLED=false
EMAIL_ENABLED=false
```

---

## 📊 Import Behavior Summary

| Aspect | Behavior |
|--------|----------|
| Transaction | All-or-nothing (1 failure = entire batch fails) |
| Duplicate Consumer ID | Error with specific message |
| Duplicate Phone | Error with specific message |
| Same File Upload | Rejected by SHA-256 hash check |
| Modified File | Allowed (different hash) |
| Missing Hierarchy | Auto-created during import |
| Hierarchy Storage | Stores IDs (circleId, divisionId, subdivisionId), not names |

---

## 🔗 Previous Session (5 Feb 2026 - Verification Flow & Display Fixes)

### What Was Done:

#### **PART 1: RO Verify Page - Human Readable IDs & Details**
   - Added nested `consumer { id, consumerId, name, phone, email }`

2. **RO Verify Page Improvements:**
   - Shows Site ID (SITE-XXXX format) instead of UUID
   - Shows Meter Serial Number instead of meter UUID
   - Shows Consumer details: consumerId, name, phone, email
   - Shows Contractor details: companyName, phone, email
   - Fixed `name` → `companyName` for Contractor type

---

#### **PART 2: Installation Evidence Upload - REST Flow**

**Goal:** Fix evidence upload to use proper REST API with auth token.

**Changes:**

1. **Contractor Install Page:**
   - Updated to use REST endpoint: `POST /files/upload/private/installation-evidence`
   - Fixed auth token retrieval: `storage.get('mescom_auth_token')`
   - Form data includes file with proper headers

2. **Evidence Resolver:**
   - Added `@ResolveField fileUrl` to generate signed URLs for storage keys
   - Evidence now displays correctly in verification page

---

#### **PART 3: Site State Inconsistency Fix (CRITICAL)**

**Problem:** Site showed "VERIFICATION_PENDING" in site-review page but installation showed "VERIFIED".

**Root Cause:** When verification was approved, the site state transition from current state to `VERIFIED` failed silently because:
- Site was at `INSTALLED` state
- `INSTALLED → VERIFIED` is NOT a valid transition
- Valid path: `INSTALLED → VERIFICATION_PENDING → VERIFIED`

**Fix in `verifications.service.ts`:**
```typescript
// Before: Direct transition (failed for INSTALLED sites)
await this.sitesService.transitionState(installation.siteId, SiteState.VERIFIED, ...);

// After: Handle intermediate states
const currentSite = await this.sitesService.findById(installation.siteId);
if (currentSite.siteState === SiteState.INSTALLED) {
  await this.sitesService.transitionState(..., SiteState.VERIFICATION_PENDING, ...);
}
if (currentSite.siteState === SiteState.INSTALLATION_IN_PROGRESS) {
  await this.sitesService.transitionState(..., SiteState.INSTALLED, ...);
  await this.sitesService.transitionState(..., SiteState.VERIFICATION_PENDING, ...);
}
await this.sitesService.transitionState(..., SiteState.VERIFIED, ...);
```

**Result:** Site state now properly transitions through all required states.

---

#### **PART 4: Meter State Validation Fixes**

**Problem 1:** `markInstalled` failed because it checked only `assignedConsumerId`.

**Fix:** Updated `meters.service.ts` to check `assignedSiteId || assignedConsumerId`:
```typescript
// Before
if (meter.state !== 'ASSIGNED' || !meter.assignedConsumerId) {
  throw new BadRequestException('Meter is not assigned to a consumer');
}

// After  
if (meter.state !== 'ASSIGNED' || (!meter.assignedSiteId && !meter.assignedConsumerId)) {
  throw new BadRequestException('Meter is not assigned to a site or consumer');
}
```

**Problem 2:** Verification approval failed when meter was still `ASSIGNED`.

**Fix:** Added meter state transition in verification approval:
```typescript
if (meter.state === 'ASSIGNED') {
  try {
    await this.metersService.markInstalled(meterId, initialReading, reviewerId);
  } catch (err) {
    console.warn('Failed to mark meter as installed:', err);
  }
}
await this.metersService.markActive(meterId, reviewerId);
```

---

#### **PART 5: Other Fixes This Session**

1. **Network Availability Field:**
   - Added `networkAvailability` to each connection in RO site-review verification form

2. **Schedule Appointments Flow:**
   - Updated `/ro/schedule` to show success message and redirect to `/ro/assign`

3. **Contractor Installation Flow:**
   - Fixed "No installation found" when contractor clicked "Start Installation"
   - Fixed by creating installation record when RO assigns meter in `site-meter-connections.service.ts`
   - Added fallback button for legacy sites without installation records

4. **RO Installations Page Data Isolation:**
   - Fixed `roId` lookup to use `retailOutlets.id` instead of user ID

5. **TypeScript Errors:**
   - Fixed `UserRole` import in various services
   - Fixed nullable `contractorId` validation

---

### Files Modified This Session:

```
Backend:
├── src/modules/installations/
│   ├── installations.resolver.ts (added meter, contractor, consumer ResolveFields)
│   └── installations.module.ts (added ContractorsModule, ConsumersModule imports)
├── src/modules/verifications/
│   └── verifications.service.ts (site state intermediate transitions, meter state handling)
├── src/modules/meters/
│   └── meters.service.ts (fixed markInstalled assignedSiteId check)
├── src/modules/installation-evidence/
│   └── installation-evidence.resolver.ts (added fileUrl signed URL ResolveField)
├── src/modules/site-meter-connections/
│   └── site-meter-connections.service.ts (creates installation on meter assignment)

Frontend:
├── src/graphql/installations.ts (updated query with nested relations)
├── src/app/(ro)/ro/verify/page.tsx (human-readable IDs and details display)
├── src/app/(contractor)/contractor/install/[installationId]/page.tsx (REST upload flow)
```

---

### State Machine Flow (Complete & Verified):

```
CREATED
  ↓ (Consumer uploads photo/address, RO sends links)
SITE_PENDING
  ↓ (RO verifies site requirements)
SITE_VERIFIED
  ↓ (RO schedules appointment)
INSTALLATION_SCHEDULED
  ↓ (RO assigns meter → creates installation)
METER_ASSIGNED
  ↓ (Contractor starts installation)
INSTALLATION_IN_PROGRESS
  ↓ (Contractor completes, uploads evidence)
INSTALLED
  ↓ (RO creates verification request)
VERIFICATION_PENDING
  ↓ (RO approves verification)
VERIFIED
  ↓ (Billing handoff)
BILLED
```

---

### Key Bug Fixes Summary:

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Site stuck at VERIFICATION_PENDING | Direct INSTALLED→VERIFIED transition invalid | Added intermediate state transitions |
| Evidence images not displaying | Storage key not converted to signed URL | Added fileUrl ResolveField |
| Evidence upload 401 error | Wrong auth token key | Fixed to use `mescom_auth_token` |
| Meter state error on complete | Only checked assignedConsumerId | Added assignedSiteId check |
| Meter state error on verify | Meter still ASSIGNED when approving | Added ASSIGNED→INSTALLED transition |
| Installation not found | No record created on meter assign | Creates installation in createConnection |
| RO installations empty | Wrong roId (user ID vs retailOutlets.id) | Fixed lookup to use retailOutlets.id |

---

### Testing Verified:

1. ✅ Consumer can upload site photo and update address
2. ✅ RO can verify site and schedule appointment
3. ✅ RO can assign meter (creates installation automatically)
4. ✅ Contractor can start installation
5. ✅ Contractor can upload evidence photos (REST + GraphQL)
6. ✅ Contractor can complete installation with reading
7. ✅ RO can view completed installations with evidence
8. ✅ RO can approve verification (site → VERIFIED)
9. ✅ All state transitions work correctly

---

### For Existing Sites Stuck at VERIFICATION_PENDING:

Run this GraphQL mutation as Admin/RO:
```graphql
mutation {
  transitionSiteState(
    id: "<site-uuid>", 
    newState: VERIFIED, 
    reason: "Manual fix - verification was already approved"
  ) {
    success
    message
    newState
  }
}
```

---

## 📋 Previous Session Context (3 Feb 2026 - Contractor Installation Flow)

### What Was Done:

1. **Contractor Installation Execution Page** (`/contractor/install/[installationId]`)
   - Start installation with GPS capture
   - Photo evidence upload (meter, seal, reading, location)
   - Meter reading entry and completion

2. **State Machine Fixes:**
   - Auto-transitions through intermediate states
   - `initialReading` must be STRING not number
   - Use `comments` field not `notes`

3. **RO Installations Page** (`/ro/installations`)
   - Tabs: Scheduled, In Progress, Completed, All
   - Stats cards and details modal

### Key Points:
- Installation created when RO assigns meter
- Site transitions: METER_ASSIGNED → IN_PROGRESS → INSTALLED
- Verification transitions: INSTALLED → VERIFICATION_PENDING → VERIFIED

---

## 📋 Previous Session Context (2 Feb 2026 - Contractor Assignment Flow Implementation)

### What Was Done This Session:

#### **PART 1: Contractor Model & Authentication (Completed Earlier)**
- Made contractors use phone (not email) for OTP-based login
- Added locationUpdatedBy flag to prevent double address/location updates
- Added contractor fields to consumer bulk import
- Created contractor bulk upload feature

#### **PART 2: Contractor Assignment Workflow (NEW - Just Completed)**

**Goal:** Enable contractor assignment BEFORE site verification (not after meter assignment)

**Backend Changes:**

1. **Database Schema Updates:**
   - Added `contractorId` field to `consumer_sites` table (FK to contractors)
   - Added `contractorAssignedAt` timestamp field
   - Added `contractorInviteSentAt` timestamp field  
   - Added index on `contractor_id` for query performance
   - Migration: `0002_absurd_stryfe.sql`

2. **GraphQL API Updates:**
   - Added `contractorId`, `contractorAssignedAt`, `contractorInviteSentAt` fields to `ConsumerSite` type
   - Added `assignContractorToSite(siteId, contractorId)` mutation
   - Added `sendContractorInvite(siteId)` mutation
   - Updated `GET_SITE` query to include contractor fields

3. **Service Layer:**
   - Added `assignContractor()` method in `SitesService`
   - Added `markContractorInviteSentAt()` method in `SitesService`
   - Updated `mapToGraphQL()` to include contractor fields
   - Contractor can be assigned at ANY stage (during import, before verification, etc.)

**Frontend Changes:**

1. **New Admin Site Detail Page:**
   - Created `/admin/sites/[siteId]/page.tsx` (NEW FILE)
   - Shows site address, requirements, RAPDRP status, network type
   - **Contractor Assignment Section:**
     - If no contractor: Shows dropdown to select contractor + "Assign" button
     - If contractor assigned: Shows contractor details + "Send Invite" button
     - Displays assignment timestamp and invite sent timestamp
   - Shows site state, readiness status, timeline
   - Links to consumer detail page

2. **Updated Consumer Detail Page:**
   - Made `SiteCard` clickable (links to new site detail page)
   - Added hover effect for better UX

3. **GraphQL Frontend:**
   - Added `ASSIGN_CONTRACTOR_TO_SITE` mutation
   - Added `SEND_CONTRACTOR_INVITE` mutation
   - Updated `GET_SITE` query to fetch contractor fields
   - Used existing `GET_CONTRACTORS` query for dropdown

**Flow Changes:**

**OLD FLOW:**
```
Consumer Import → Site Created → Site Verification → Meter Assigned → Contractor Assigned (during installation)
```

**NEW FLOW:**
```
Consumer Import (optional: contractor pre-assigned from Excel)
  ↓
Site Created
  ↓
Admin Assigns Contractor (if not pre-assigned) ← NEW STEP
  ↓
Admin Sends Contractor Invite (at initial review) ← NEW STEP
  ↓
Site Verification (contractor already knows about the job)
  ↓
Meter Assigned
  ↓
Installation
```

**Key Benefits:**
- Contractors are notified early (during initial review, not at installation)
- Contractors can plan ahead and schedule site visits
- Admin has full visibility of contractor assignments
- Contractor assignment is independent of meter assignment
- Supports both Excel pre-assignment and manual assignment

### Files Modified/Created This Session:

```
Backend (Session Part 2):
├── src/database/schema/index.ts (contractor fields + index in consumer_sites)
├── src/database/migrations/0002_absurd_stryfe.sql (GENERATED)
├── src/modules/sites/
│   ├── sites.types.ts (added contractorId, contractorAssignedAt, contractorInviteSentAt fields)
│   ├── sites.service.ts (added assignContractor, markContractorInviteSent methods)
│   └── sites.resolver.ts (added assignContractorToSite, sendContractorInvite mutations)

Frontend (Session Part 2):
├── src/app/(admin)/admin/sites/[siteId]/page.tsx (NEW - full site detail + contractor assignment UI)
├── src/app/(admin)/admin/consumers/[consumerId]/page.tsx (made SiteCard clickable)
├── src/graphql/sites.ts (added ASSIGN_CONTRACTOR_TO_SITE, SEND_CONTRACTOR_INVITE mutations)
```

### New GraphQL Operations:

```graphql
# Assign contractor to site
mutation AssignContractorToSite($siteId: ID!, $contractorId: ID!) {
  assignContractorToSite(siteId: $siteId, contractorId: $contractorId) {
    id
    contractorId
    contractorAssignedAt
    siteState
  }
}

# Send contractor invite link
mutation SendContractorInvite($siteId: ID!) {
  sendContractorInvite(siteId: $siteId) {
    id
    contractorInviteSentAt
  }
}
```

### Task Status (Full Session):
- ✅ Contractor phone (required) + OTP login
- ✅ Contractor email (optional)
- ✅ Contractor licenseNumber (alphanumeric, required, unique, global)
- ✅ Contractor global scope (all admins can see/assign any contractor)
- ✅ Consumer bulk upload with contractor fields (Excel)
- ✅ Contractor upsert during consumer import
- ✅ Address/location update lock (locationUpdatedBy flag)
- ✅ Contractor bulk upload via admin upload page
- ✅ Frontend login UI updated for contractor OTP login
- ✅ **Contractor assignment UI in admin site detail page (NEW)**
- ✅ **Send contractor invite functionality (NEW)**
- ✅ **Site detail page shows RAPDRP, network type, requirements (NEW)**
- ✅ **Updated site card to link to detail page (NEW)**

### Pending/Future Work:
- 🟡 Location update UI indicator (show who updated: CONSUMER vs CONTRACTOR)
- 🟡 SMS/Email gateway integration for invite sending (currently just marks as sent)
- 🟡 Contractor dashboard to view assigned sites
- 🟡 Push notifications for contractors on assignment

---

## 📋 Previous Session Context (29 Jan 2026 - Evening)

### What Was Done This Session:

#### 1. Contractor Division Fields - Backend & Frontend
**Goal:** Display contractor's division in assign page, allow filtering by division
**Changes:**
- Added `divisionId` and `divisionName` fields to Contractor GraphQL type (`contractors.types.ts`)
- Added field resolvers in `contractors.resolver.ts` to fetch division info from users table
- Updated frontend `GET_CONTRACTORS` query to include `divisionId` and `divisionName`

#### 2. Division Filter in RO Assign Page
**Goal:** Filter contractors by division when assigning meter installation
**Changes:**
- Added `divisionFilter` state and `availableDivisions` memo in `/ro/assign/page.tsx`
- Added filter dropdown in contractor selection step
- Contractors filtered based on selected division
- Each contractor now displays their division name badge

#### 3. Network Type & RAPDRP Display in Assign Page
**Goal:** Show site requirements (network type, RAPDRP) when selecting sites
**Changes:**
- Updated `GET_SITES` query to include `requiredNetworkType` and `isRapdrp` fields
- Site list now shows:
  - RAPDRP badge (purple) if site is RAPDRP
  - Network type (RF, GPRS, etc.) in the site details
- Updated Site interface with `requiredNetworkType` and `isRapdrp` fields

### Files Modified This Session:
```
Backend:
├── src/modules/contractors/contractors.types.ts (added divisionId, divisionName)
├── src/modules/contractors/contractors.resolver.ts (field resolvers for division)

Frontend:
├── src/graphql/contractors.ts (added divisionId, divisionName to query)
├── src/graphql/sites.ts (added requiredNetworkType, isRapdrp to query)
├── src/app/(ro)/ro/assign/page.tsx:
    - Added divisionFilter state
    - Added availableDivisions memo
    - Added division filter dropdown UI
    - Added contractor division badge display
    - Added RAPDRP badge on site list
    - Added network type display on site list
```

### Updated Task Status:
- ✅ Consumer import with user account creation (Session 1)
- ✅ Subdivision-based RO data isolation (Session 1)
- ✅ Demo consumer creation (Rajesh Kumar, phone: 9876500001)
- ✅ Demo site creation (SITE-DEMO-001)
- ✅ Super Admin dashboard (already exists at /admin/admins)
- ✅ Consumer welcome link (already implemented in site-review flow)
- ✅ **Division filter in assign page** (NEW - this session)
- ✅ **Network type & RAPDRP UI in assign page** (NEW - this session)

---

## 📋 Previous Session Context (29 Jan 2026 - Afternoon)

### What Was Done This Session:

#### 1. Consumer Import - User Account Creation Fix
**Problem:** Imported consumers couldn't login with OTP
**Root Cause:** Import only created `consumers` record, but OTP login checks `users` table
**Fix:**
- Updated `imports.resolver.ts` to create user account with CONSUMER role before creating consumer
- Added `UsersService` dependency to ImportsResolver
- Updated `support.module.ts` to import UsersModule
- Consumer record now linked to user via `userId` field

#### 2. RO Data Isolation - Subdivision-Based Filtering
**Problem:** RO saw 0 consumers/sites even after import
**Root Cause:** 
- RO filtering was based on `consumerSites.roId`, but imported sites have `roId = null`
- Sites should be visible based on subdivision, not just explicit RO assignment
**Fix:**
- Updated `consumers.service.ts`:
  - Renamed `getAccessibleRoIds()` → `getAccessContext()`
  - Now returns both `roIds` AND `subdivisionIds` for RO users
  - `findAll()` filters by either roId OR subdivisionId (OR condition)
- RO can now see all consumers whose sites are in their subdivision

#### 3. Site Creation - Missing subdivisionId Fix
**Problem:** Sites created via import had `subdivision_id = NULL` in database
**Root Cause:** `sites.service.ts` `create()` method wasn't saving `subdivisionId` field
**Fix:**
- Added `subdivisionId: input.subdivisionId` to insert values in `sites.service.ts`

#### 4. Seed Data - RO Subdivision Assignment
**Problem:** Seeded RO had no subdivision assigned, couldn't see any data
**Fix:**
- Updated `seed.ts` to assign first subdivision to RO on creation
- Manually updated existing database: ROs and sites now linked to "Mysuru North-1 Subdivision"

#### 5. Migration Consolidation
**Done:** Consolidated 3 migration files into single `0000_flowery_sharon_ventura.sql`
- Includes all hierarchy fields (circleId, divisionId, subdivisionId) on consumers table

### Files Modified This Session:
```
Backend:
├── src/modules/support/imports.resolver.ts (creates user account during import)
├── src/modules/support/support.module.ts (added UsersModule import)
├── src/modules/consumers/consumers.service.ts (subdivision-based RO filtering)
├── src/modules/sites/sites.service.ts (save subdivisionId on create)
├── src/database/seeders/seed.ts (assign subdivision to RO)
├── src/database/migrations/0000_flowery_sharon_ventura.sql (consolidated)
```

### Database Changes Applied:
```sql
-- Assign subdivision to all ROs
UPDATE retail_outlets SET subdivision_id = (SELECT id FROM subdivisions LIMIT 1) WHERE subdivision_id IS NULL;

-- Assign subdivision to all sites  
UPDATE consumer_sites SET subdivision_id = (SELECT id FROM subdivisions LIMIT 1) WHERE subdivision_id IS NULL;
```

### Current Data State:
- **ROs:** 2 (ro@mescom.gov, ro08@gmail.com) → Both assigned to "Mysuru North-1 Subdivision"
- **Consumers:** 10+ (1 seed + imports)
- **Sites:** 10+ (all linked to subdivision)
- **Circles/Divisions/Subdivisions:** Auto-created from import

---

## 📋 Previous Session Context (27 Jan 2026 - Evening)

### What Was Done That Session:

#### 1. Role Hierarchy Implementation
**Goal:** SUPER_ADMIN → ADMIN → RO → (Contractors, Sub-users, Consumers)

- Added `SUPER_ADMIN` and `SUB_USER` roles to `userRoleEnum`
- Updated `users` table with new fields:
  - `otp_code`, `otp_expires_at` (for consumer OTP login)
  - `created_by` (tracks who created the user)
  - `parent_ro_id` (links contractors/sub-users to their RO)
  - `division_id` (for contractor division assignment)
- Made `email` and `password_hash` nullable (consumers use phone+OTP only)
- Created role creation permission matrix in `UsersService`

#### 2. Consumer OTP-Based Login
**Goal:** Consumers login via Phone + OTP (not email/password)

- Added `requestOtp` mutation - sends OTP to phone (dummy: 123456)
- Added `verifyOtp` mutation - validates OTP and returns JWT
- Updated frontend login page with dual mode:
  - Email/Password for Admin, RO, Contractor
  - Phone/OTP for Consumer
- Added `loginWithToken` method to AuthProvider

#### 3. Meter Assignment Validation Updates
**Goal:** Add network type and RAPDRP validation

- Added `required_network_type` and `is_rapdrp` to `consumer_sites` table
- Updated `SiteMeterConnectionsService` to validate:
  1. Phase type matching (existing)
  2. Network type matching (new) - RF, GPRS, NB_IOT, LORA, DLMS, MODBUS
  3. RAPDRP status matching (new)

#### 4. Contractor Management by RO
**Goal:** RO creates and manages contractors

- Created `/ro/contractors` page for contractor management
- RO can create contractors with: name, email, password, company, license, division
- Added `myContractors` query to get contractors under current RO
- Added `contractorsByDivision` query for filtering

#### 5. Contractor Pickup Removal
**Goal:** Skip pickup step - direct to installation

- Removed "Pickup" link from contractor navigation
- Contractor flow now: Job Assigned → Start Installation → Complete

#### 6. RO Verify Page Improvements
- Added success message after approve/reject
- Returns to list view (not dashboard) after verification
- Added toggle between "Pending" and "Verified" installations list
- Verified count now shows real data (was showing "-")
- Hides approve/reject buttons for already verified installations

### Files Modified This Session:
```
Backend:
├── src/common/enums/index.ts (added SUPER_ADMIN, SUB_USER)
├── src/database/schema/index.ts (updated users table, added site fields)
├── src/database/migrations/0003_add_role_hierarchy_and_otp.sql (NEW)
├── src/modules/auth/auth.service.ts (added OTP methods)
├── src/modules/auth/auth.resolver.ts (added OTP mutations)
├── src/modules/auth/auth.types.ts (added OTP types)
├── src/modules/auth/jwt.strategy.ts (made email optional)
├── src/modules/users/users.service.ts (role hierarchy logic)
├── src/modules/users/users.resolver.ts (new mutations)
├── src/modules/users/users.types.ts (new input types)
├── src/modules/sites/sites.types.ts (added MeterNetworkType import)
├── src/modules/site-meter-connections/site-meter-connections.service.ts (validation)

Frontend:
├── src/app/(auth)/login/page.tsx (OTP login for consumers)
├── src/app/(ro)/layout.tsx (added Contractors nav)
├── src/app/(ro)/ro/contractors/page.tsx (NEW)
├── src/app/(ro)/ro/verify/page.tsx (improvements)
├── src/app/(contractor)/layout.tsx (removed Pickup nav)
├── src/lib/auth/AuthProvider.tsx (added loginWithToken)
├── src/types/auth.ts (updated types)
├── src/graphql/auth.ts (OTP mutations)
```

### What's Still Pending:
1. **Additional site validation fields** - Consider adding more validation for meter-site compatibility
2. **Enhanced contractor filtering** - Add filters by license expiry, active jobs count
3. **Bulk meter assignment** - Assign multiple meters to sites in one operation
4. **Mobile responsiveness improvements** - Test and fix mobile layout issues

---

## ✅ Completed Features (Updated 29 Jan)

### Core Modules
- [x] Consumer Management (CRUD, state machine)
- [x] Site Management (lifecycle: CREATED → VERIFIED → BILLED)
- [x] Meter Registry & Inventory
- [x] Meter Specifications (phase, voltage, RAPDRP, AEH)
- [x] Site-Meter Connections (domain entity with history)
- [x] Installations Module
- [x] Verifications Module
- [x] Audit Logging (all state changes tracked)
- [x] **Consumer Import with User Account** (NEW - 29 Jan) - Imported consumers can now login with OTP
- [x] **Subdivision-based RO Data Isolation** (NEW - 29 Jan) - RO sees sites/consumers in their subdivision
- [x] **Auto-create Hierarchy on Import** (NEW - 29 Jan) - Circle/Division/Subdivision created if not exists

### Recently Implemented

#### 1. Circle/Division/Subdivision Hierarchy
- New tables: `circles`, `divisions`, `subdivisions`
- ROs and Sites linked to subdivisions
- Full GraphQL API with nested queries
- Seeded with MESCOM circles (MYS, BGM, MNG, DVG, SHP)

#### 2. Notification Service
- `sendWelcomeMessage` mutation
- `sendAppointmentNotification` mutation
- `sendInstallationCompleteNotification` mutation
- Dev mode logging (ready for SMS gateway)

#### 3. Meter Reservation Timeout
- Hourly cron job releases expired reservations
- Uses `@nestjs/schedule`

#### 4. Site Verification Enhancements
- Network provider field (AIRTEL/JIO/BSNL/VODAFONE/NONE)
- Electrical readiness checkbox
- Site accessibility notes
- Appointment scheduling

#### 5. Installation Evidence
- Photo upload tracking
- Initial meter reading storage
- Seal/LED verification fields

#### 6. Billing Export Module
- Generate billing exports for verified sites
- Handoff to external billing system
- Confirmation workflow

#### 7. Excel Upload
- Consumer bulk import (`importConsumers`)
- Meter bulk import (`importMeters`)
- Validation and error reporting

---

## 🔧 Pending Work

| Priority | Feature | Notes |
|----------|---------|-------|
| 🔴 High | **Consumer Welcome Link** | Send link with welcome chat status before initial review |
| 🔴 High | **Division Filter in Assign** | Filter contractors by division when assigning jobs |
| 🔴 High | **File Storage Service** | Needed for photo uploads (S3/local) |
| 🟡 Medium | **SMS Gateway Integration** | Placeholder ready for MSG91/Twilio |
| 🟡 Medium | **Email Gateway Integration** | Placeholder ready for SendGrid/SES |
| 🟡 Medium | **Network Type UI in Assign** | Site requirement selection for network type/RAPDRP |
| 🟡 Medium | **Super Admin Dashboard** | UI for creating admins |
| 🟢 Low | **OCR for Meter Reading** | Deferred per user request |
| 🟢 Low | **QR Scanner Backend** | Frontend component exists |

---

## 📁 New Files Created

```
mescom-backend/src/
├── modules/
│   ├── hierarchy/
│   │   ├── hierarchy.module.ts
│   │   ├── hierarchy.service.ts
│   │   ├── hierarchy.resolver.ts
│   │   ├── hierarchy.types.ts
│   │   └── index.ts
│   ├── notification/
│   │   ├── notification.module.ts
│   │   ├── notification.service.ts
│   │   ├── notification.resolver.ts
│   │   └── index.ts
│   └── scheduler/
│       ├── scheduler.module.ts
│       ├── meter-timeout.service.ts
│       └── index.ts
└── database/
    └── migrations/
        ├── 0002_add_requirement_fields.sql
        ├── 0003_add_hierarchy_tables.sql
        └── 0003_add_role_hierarchy_and_otp.sql (NEW - Jan 27)
```

---

## 🧪 Test Credentials

| Role | Email/Phone | Password/OTP |
|------|-------------|--------------|
| Super Admin | superadmin@mescom.gov | admin123 |
| Admin | admin@mescom.gov | admin123 |
| RO | ro@mescom.gov | retail123 |
| Contractor | contractor@example.com | contractor123 |
| Consumer | (any phone number) | OTP: **123456** (dummy) |

### Imported Consumer Phones (can login with OTP 123456):
- **Demo Consumer:** Phone `9876500001` (Rajesh Kumar) - Consumer ID: CONS-DEMO-001
- Check database for other imported consumer phone numbers
- All imported consumers have user accounts created for OTP login

### To Create Demo Site (if not exists):
```bash
cd mescom-backend
psql postgres://praharsh@localhost:5432/mescom_new -f scripts/create-demo-site.sql
```

---

## 🚀 Quick Start

```bash
# Backend
cd mescom-backend
npm run start:dev

# Frontend
cd Mescom
npm run dev
```

**Backend:** http://localhost:3001/graphql  
**Frontend:** http://localhost:3000

---

## 🗄️ Database Setup & Migration Guide

### Prerequisites
- PostgreSQL 17 installed
- Node.js 18+ and npm

### Initial Setup (New Machine)

```bash
# 1. Create the database
psql -U postgres
CREATE DATABASE mescom;
\q

# 2. Update connection string in mescom-backend/drizzle.config.ts
# Format: postgres://USERNAME@localhost:5432/mescom
# Example: postgres://praharsh@localhost:5432/mescom

# 3. Install dependencies
cd mescom-backend
npm install

# 4. Push schema to database (creates all tables)
npx drizzle-kit push

# 5. Run seed data (creates test users, circles, meters, etc.)
npx tsx src/database/seeders/seed.ts
```

### Running Migrations

Migrations are SQL files in `mescom-backend/src/database/migrations/`:
- `0001_create_tables.sql` - Initial tables
- `0002_add_requirement_fields.sql` - Meter requirements
- `0003_add_hierarchy_tables.sql` - Circles/Divisions/Subdivisions
- `0003_add_role_hierarchy_and_otp.sql` - Role hierarchy & OTP

**To run a migration manually:**
```bash
# Option 1: Using psql directly
psql -U USERNAME -d mescom -f src/database/migrations/MIGRATION_FILE.sql

# Option 2: Using Drizzle push (applies schema changes automatically)
npx drizzle-kit push
```

### Cloning to Different Device

1. **Pull latest code from git**
2. **Install dependencies in both folders:**
   ```bash
   cd mescom-backend && npm install
   cd ../Mescom && npm install
   ```
3. **Create database and run schema:**
   ```bash
   psql -U postgres -c "CREATE DATABASE mescom;"
   cd mescom-backend
   npx drizzle-kit push
   ```
4. **Run seed to create test data:**
   ```bash
   npx tsx src/database/seeders/seed.ts
   ```
5. **Start servers:**
   ```bash
   # Terminal 1
   cd mescom-backend && npm run start:dev
   
   # Terminal 2
   cd Mescom && npm run dev
   ```

### Connection String

Update `mescom-backend/drizzle.config.ts`:
```typescript
export default defineConfig({
  schema: './src/database/schema/index.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: 'postgres://YOUR_USERNAME@localhost:5432/mescom',
  },
});
```

---

## 📊 Role Hierarchy Reference

```
SUPER_ADMIN
    └── Creates → ADMIN
                    └── Creates → RETAIL_OUTLET (RO)
                                    ├── Creates → CONTRACTOR
                                    ├── Creates → SUB_USER
                                    └── Creates → CONSUMER (via site)
```

### Role Permissions:
| Role | Can Create | Uses |
|------|------------|------|
| SUPER_ADMIN | ADMIN | Email + Password |
| ADMIN | RETAIL_OUTLET | Email + Password |
| RETAIL_OUTLET | CONTRACTOR, SUB_USER, CONSUMER | Email + Password |
| CONTRACTOR | - | Email + Password |
| SUB_USER | - | Email + Password |
| CONSUMER | - | Phone + OTP |

---

## 🔄 GraphQL Queries Reference

### Authentication
```graphql
# Login with email/password
mutation {
  login(input: { email: "admin@mescom.gov", password: "admin123" }) {
    token
    user { id name role }
  }
}

# Consumer OTP Login
mutation {
  requestOtp(input: { phone: "9876543210" }) {
    success
    message
  }
}

mutation {
  verifyOtp(input: { phone: "9876543210", otp: "123456" }) {
    token
    user { id name role }
  }
}
```

### User Creation (Role-based)
```graphql
# Create Contractor (RO only)
mutation {
  createContractor(input: {
    name: "John Doe"
    email: "john@company.com"
    password: "password123"
    phone: "9876543210"
    company: "ABC Electric"
    licenseNumber: "LIC-2024-001"
  }) {
    id name role
  }
}

# Create Consumer (RO only)
mutation {
  createConsumer(input: {
    name: "Consumer Name"
    phone: "9876543210"
    rrNumber: "RR123456"
    address: "123 Main St"
  }) {
    id name role
  }
}
```

---

## 📝 Known Issues & Fixes

### Issue: Duplicate enum registration error
**Error:** `Enum "MeterNetworkType" was already registered`
**Fix:** Import from existing module instead of redefining:
```typescript
// In sites.types.ts
import { MeterNetworkType } from '../meter-specs/meter-specs.types';
// NOT: export enum MeterNetworkType { ... }
```

### Issue: Email required but consumers use OTP
**Fix:** Made `email` and `password_hash` nullable in users table (migration 0003)

### Issue: Verify page not showing success state
**Fix:** Added success message state and proper navigation back to list

---

## 📊 Project Analysis - Pending Items (30 Jan 2026)

### ✅ Core Features - COMPLETED

| Module | Status | Notes |
|--------|--------|-------|
| Consumer Management | ✅ | CRUD, state machine, OTP login |
| Site Management | ✅ | Full lifecycle (CREATED → BILLED) |
| Meter Registry | ✅ | Inventory, specs, QR scanning |
| Site-Meter Connections | ✅ | Domain entity with history |
| Installations | ✅ | Contractor job flow |
| Verifications | ✅ | RO approval workflow |
| Audit Logging | ✅ | All state changes tracked |
| Circle/Division/Subdivision Hierarchy | ✅ | Full API |
| Consumer Import with User Account | ✅ | OTP login works |
| RO Data Isolation (Subdivision) | ✅ | Fixed 29 Jan |
| Division filter in assign page | ✅ | Added 29 Jan |
| Network type & RAPDRP display | ✅ | Added 29 Jan |

### ⚠️ Pending Items - Prioritized

#### **High Priority (Core Flow Gaps)**

1. **Consumer App - Welcome Link Flow**
   - Per requirements: "RO sends welcome message with customer app link"
   - Backend: `markWelcomeSent` mutation exists
   - **Missing:** Actual SMS/notification trigger is dev-mode logging only
   - **Action:** Consider adding real SMS gateway integration (Twilio/MSG91)

2. **Contractor Pickup Flow Confusion**
   - Pickup was "removed" per update log, but `/contractor/pickup/` folder still exists
   - Contractor flow is now: "Job Assigned → Start Installation → Complete"
   - **Action:** Verify if pickup page should be deleted or kept as optional

3. **Meter Reservation Timeout Testing**
   - Cron job exists in scheduler module
   - **Action:** Need to test the hourly job actually releases expired reservations

#### **Medium Priority (UX Enhancements)**

4. **Bulk Meter Assignment**
   - Assign multiple meters to multiple sites in one operation
   - Currently: One-by-one assignment only

5. **Enhanced Contractor Filtering**
   - Filters by: license expiry, active jobs count, workload
   - Currently: Only division filter exists

6. **OCR for Meter Reading**
   - Mentioned in requirements: "Initial meter reading using OCR"
   - **Status:** UI exists but OCR integration is manual entry

7. **Mobile Responsiveness**
   - Most pages are desktop-optimized
   - Need testing on mobile devices

#### **Low Priority (Nice to Have)**

8. **Admin Master Data Pages**
   - `/admin/master` exists but may need:
     - Meter type management
     - Tariff code management
     - Network type configuration

9. **Reports & Analytics**
   - `/admin/analytics` exists
   - Consider adding:
     - Installation success rate by contractor
     - Average installation time
     - Failure reason analysis

10. **Consumer Notification History**
    - Track all SMS/notifications sent to consumer
    - Currently: Welcome message tracking only

### 🔧 Technical Debt

| Issue | Location | Priority |
|-------|----------|----------|
| `<img>` instead of Next.js `<Image>` | Multiple pages | Low |
| useMemo dependency warnings | assign/page.tsx | Low |
| Pickup folder exists but feature removed | contractor/pickup | Low |
| Dev-mode notification logging | notification module | Medium |

### 📋 Recommended Next Steps

1. **Test the full E2E flow** with demo data (consumer login → photo upload → RO review → meter assign → contractor install → RO verify → billing)
2. **Remove or hide contractor pickup page** if the flow skips it
3. **Add real SMS integration** for production (replace console.log)
4. **Run the app on mobile** to identify responsive design issues

