# Session Changes — Marketplace Features (2026-03-09)

## Overview

Four new features were implemented across the backend (`user-service`) and admin frontend (`web-frontend`):

1. **Service Commission** — admin-configurable 0–30% cut per service
2. **Subscription Job Quota** — plan-level job limit per billing cycle with custom plan names
3. **Contractor QR Badge** — public API for mobile QR scan (no auth required)
4. **Go-Live Checklist** — admin dashboard widget tracking marketplace setup completeness

---

## ⚠️ ONE ACTION REQUIRED

```bash
cd user-service
pnpm db:migrate
```

Migration file already generated: `src/database/migrations/0017_dear_mesmero.sql`
Adds 5 new columns to existing tables — no destructive changes.

---

## Backend Changes (`user-service/`)

### Database Schema

#### `src/database/schema/marketplace/services.ts`
- Added `commissionPercent` — `integer`, default `0`, range 0–30

#### `src/database/schema/marketplace/jobs.ts`
- Added `commissionPercent` — snapshot of service commission at job creation time
- Added `commissionAmount` — `numeric(10,2)`, association cut in rupees
- Added `contractorPayout` — `numeric(10,2)`, amount contractor actually receives

#### `src/database/schema/marketplace/subscription-plans.ts`
- Added `planName` — `varchar(100)`, nullable custom display name (e.g. "Starter", "Pro")
- Added `maxJobsPerCycle` — `integer`, nullable job quota per billing cycle (null = unlimited)

#### `src/database/schema/marketplace/contractor-extensions.ts`
- Added `jobsUsedThisCycle` — `integer`, default `0`, tracks accepted jobs in current cycle

---

### Services Catalog

#### `src/modules/marketplace/services-catalog/services.types.ts`
- Added `commissionPercent: number` to `MarketplaceService` ObjectType
- Added `commissionPercent?: number` to `CreateServiceInput` (validator: `@Min(0) @Max(30)`)
- Added `commissionPercent?: number` to `UpdateServiceInput` (same validators)

#### `src/modules/marketplace/services-catalog/services.service.ts`
- `create()`: persists `commissionPercent` (defaults to 0 if not provided)
- `update()`: allows updating `commissionPercent`
- `mapToDto()`: returns `commissionPercent` in service DTO

---

### Jobs

#### `src/modules/marketplace/jobs/jobs.types.ts`
- Added `commissionPercent`, `commissionAmount`, `contractorPayout` to `MarketplaceJob` ObjectType

#### `src/modules/marketplace/jobs/jobs.service.ts`

**`createJob()`:**
- Fetches `commissionPercent` from the associated service
- Computes `commissionAmount = priceSnapshot × commissionPercent / 100`
- Computes `contractorPayout = priceSnapshot − commissionAmount`
- Snapshots all 3 values into the `mp_jobs` row at creation time

**`acceptJob()`:**
- Fetches contractor's current subscription plan (including `maxJobsPerCycle`)
- **Quota gate**: if contractor is PREMIUM, `maxJobsPerCycle` is set, and `jobsUsedThisCycle >= maxJobsPerCycle` → throws `BadRequestException`
- On successful acceptance → increments `jobsUsedThisCycle` by 1

**`mapJobToDto()`:**
- Returns `commissionPercent`, `commissionAmount`, `contractorPayout` in all job query responses

---

### Subscriptions

#### `src/modules/marketplace/subscriptions/subscriptions.types.ts`
- Added `planName?: string` and `maxJobsPerCycle?: number` to:
  - `SubscriptionPlan` ObjectType
  - `CreateSubscriptionPlanInput` (validators: `@IsString @MaxLength(100)` / `@IsInt @Min(1)`)
  - `UpdateSubscriptionPlanInput` (same validators, both optional)

#### `src/modules/marketplace/subscriptions/subscriptions.service.ts`
- `createSubscriptionPlan()`: persists `planName` and `maxJobsPerCycle` (null if omitted)
- `updateSubscriptionPlan()`: persists and returns `planName`/`maxJobsPerCycle`
- `getSubscriptionPlans()`: returns `planName` and `maxJobsPerCycle` in DTOs
- `confirmSubscriptionPayment()`: added `jobsUsedThisCycle: 0` — **resets quota on new cycle**
- `adminSetSubscription()`: added `jobsUsedThisCycle: 0` — **resets quota when admin sets manually**

---

### Contractor Profile

#### `src/modules/marketplace/contractors/contractor-profile.types.ts`

Added new GraphQL ObjectTypes:

**`ContractorPublicBadge`**
```
contractorId, contractorName, companyName, phone, licenseNumber,
subscriptionType, subscriptionValidTill, isVerified
```

**`GoLiveChecklistStep`**
```
id, label, isDone, link?
```

**`GoLiveChecklist`**
```
steps: [GoLiveChecklistStep], pendingCount, completedCount, isReady
```

#### `src/modules/marketplace/contractors/contractor-profile.service.ts`

**`getContractorPublicBadge(token: string)`** — new method:
- Looks up `mp_contractor_profile` by `qrProfileToken`
- Joins `contractors` + `users` tables
- Returns public-safe fields only (no email, no internal token)
- Throws `NotFoundException` on invalid/missing token

**`getGoLiveChecklist()`** — new method:
- Runs 6 COUNT queries **in parallel** via `Promise.all()`
- Checks: active category, active service, global SLA, active subscription plan, classified location, PREMIUM contractor
- Returns step array with `isDone`, `link` (admin page URL), + `isReady` boolean

#### `src/modules/marketplace/contractors/contractor-profile.resolver.ts`
- Imported `ContractorPublicBadge`, `GoLiveChecklist` from types
- Added `contractorPublicBadge(token: String!)` query — **NO auth guard** (public endpoint)
- Added `marketplaceGoLiveChecklist` query — guarded `@Roles(ADMIN, SUPER_ADMIN)`

---

## Frontend Changes (`web-frontend/`)

### Admin Services Page
`src/app/(admin)/admin/marketplace/services/page.tsx`

- Modal form: added `commissionPercent` state (init from existing service or 0)
- New **Commission Rate** section in Create/Edit modal:
  - Range slider (0–30) + inline number input (clamped 0–30)
  - Live hint: "Contractor receives X%" or "0% — full amount to contractor"
- `handleSave()`: passes `commissionPercent` in both create and update mutations
- Table: added **Commission** column header + amber badge "X% commission" per row
- Empty state `colSpan` fixed 5 → 6

---

### Admin Subscriptions Page
`src/app/(admin)/admin/marketplace/subscriptions/page.tsx`

- `GET_SUBSCRIPTION_PLANS` query: added `planName`, `maxJobsPerCycle` to selection
- `SubscriptionPlan` TypeScript interface: added `planName?`, `maxJobsPerCycle?`
- **AddPlanModal**: new Plan Name + Max Jobs / Cycle inputs; blank → `null` before mutation
- **EditPlanModal**: initialised from existing plan; same two inputs
- **Plan card**: shows planName in violet + quota badge ("50 jobs / cycle" / "Unlimited jobs")

---

### Admin Dashboard
`src/app/(admin)/admin/page.tsx`

- Added `GET_GO_LIVE_CHECKLIST` query
- Added **`GoLiveChecklistWidget`** component:
  - Green `CheckCircle2` for done steps, amber `XCircle` for pending
  - `ExternalLink` icon on pending steps → links to relevant admin page
  - Header badge: "✓ Ready" or "N pending"
- Layout: Billing Overview + Go-Live Checklist now side-by-side in a 2-col grid

---

## GraphQL API Reference

| Query / Mutation | Auth | Description |
|---|---|---|
| `createMarketplaceService` | Admin | + `commissionPercent: Int` (0–30) |
| `updateMarketplaceService` | Admin | + `commissionPercent: Int` (0–30) |
| `createSubscriptionPlan` | Admin | + `planName: String`, `maxJobsPerCycle: Int` |
| `updateSubscriptionPlan` | Admin | + `planName: String`, `maxJobsPerCycle: Int` |
| `subscriptionPlans` | Any auth | Returns `planName`, `maxJobsPerCycle` |
| Any job query | Auth | Returns `commissionPercent`, `commissionAmount`, `contractorPayout` |
| `contractorPublicBadge(token)` | **None** | NEW — mobile QR badge lookup |
| `marketplaceGoLiveChecklist` | Admin | NEW — readiness checklist |

---

## Business Logic Rules

| Rule | Enforced In |
|---|---|
| Commission 0–30%, default 0 | `services.service.ts` + DB schema |
| Commission snapshotted at job creation (immutable) | `jobs.service.ts createJob()` |
| `contractorPayout = price − commissionAmount` | `jobs.service.ts createJob()` |
| Quota checked before accepting job | `jobs.service.ts acceptJob()` |
| Job quota counter incremented per acceptance | `jobs.service.ts acceptJob()` |
| Quota resets to 0 on subscription renewal | `subscriptions.service.ts confirmSubscriptionPayment()` |
| Quota resets to 0 on admin-set subscription | `subscriptions.service.ts adminSetSubscription()` |
| QR token auto-generated on profile creation | `contractor-profile.service.ts createProfile()` |
| Go-live checklist has 6 gates (all must pass) | `contractor-profile.service.ts getGoLiveChecklist()` |

---

---

# Feature 5: Meter Replacement Flow

**Session:** 2026-03-09 (same day, later)

Adds a separate import path and contractor workflow for **replacing** an existing meter, distinct from the new installation flow.

## Migrations

| File | Contents |
|---|---|
| `0018_charming_amphibian.sql` | `installation_type` enum + columns on `consumer_sites`; new `old_meter_final_readings` table |

```bash
cd user-service && pnpm db:migrate
```

---

## Backend Changes (`user-service/`)

### Database Schema

#### `src/database/schema/enums.ts`
- Added `installationTypeEnum` — `'NEW' | 'REPLACEMENT'` (pgEnum)
- Added `'replacement_consumers'` to `importEntityTypeEnum`

#### `src/database/schema/consumers.ts` (`consumer_sites` table)
- `installationType` — enum `NEW | REPLACEMENT`, default `NEW`
- `oldMeterNumber` — `varchar(50)` nullable — old meter serial / RO number
- `oldCustomerId` — `varchar(50)` nullable — old consumer reference ID

#### `src/database/schema/installations.ts` — **NEW TABLE: `old_meter_final_readings`**
```
id, siteId, oldMeterNumber?, oldCustomerId?,
finalReading, readingPhotoUrl?,
ocrReading?, ocrConfidence?,
captureLatitude?, captureLongitude?,
capturedBy (FK users), capturedAt, notes?,
createdAt
```

---

### GraphQL Types
#### `src/modules/installations/installations.types.ts`
- Added `RecordOldMeterReadingInput` InputType — contractor submits OCR reading
- Added `OldMeterFinalReading` ObjectType — returned from query/mutation

---

### Installations Service & Resolver
#### `src/modules/installations/installations.service.ts`
- **`recordOldMeterReading(input, contractorUserId)`** — validates site is `REPLACEMENT` type, prevents duplicate recording, inserts into `old_meter_final_readings`
- **`getOldMeterReading(siteId)`** — returns the stored old meter reading or null
- **`create(input)`** — **Added strict validation**: if the associated site's `installationType` is `'REPLACEMENT'`, it aggressively blocks the creation of the installation with a `BadRequestException` unless `getOldMeterReading()` confirms that the old meter final reading was already captured and saved.

#### `src/modules/installations/installations.resolver.ts`
| Operation | Type | Role | Description |
|---|---|---|---|
| `recordOldMeterReading(input)` | Mutation | CONTRACTOR | Record OCR final reading of old meter before replacement |
| `oldMeterReading(siteId)` | Query | All authenticated | Get recorded old meter reading for a site |

---

### Import Resolver
#### `src/modules/support/imports.resolver.ts`
- Extended `ConsumerSiteExcelRow` interface with `oldMeterNumber?` and `oldCustomerId?` fields
- **`importReplacementConsumers(input)`** mutation — Admin only; same flow as `importConsumers` but:
  - Sets `installationType = 'REPLACEMENT'` on all created sites
  - Persists `oldMeterNumber` and `oldCustomerId` from Excel
  - Logs under `replacement_consumers` entity type
- **`bulkInsertReplacementConsumersWithTransaction()`** private method — transactional bulk insert with replacement-specific fields

---

## Frontend Changes (`web-frontend/`)

### GraphQL Operations
#### `src/graphql/imports.ts`
- Added `IMPORT_REPLACEMENT_CONSUMERS` mutation
- Added `ImportReplacementConsumersResponse` TypeScript interface

### Admin Upload Page
#### `src/app/(admin)/admin/upload/page.tsx`
- Extended `ImportType` union: `'consumers' | 'replacement_consumers' | 'contractors'`
- Import type selector changed from **2-column grid → 3-column grid**
- Added **Replacement Consumers** card (orange theme, distinct from violet)
- Template download for `replacement_consumers` includes extra columns: `Old Meter Number`, `Old Customer ID`
- `handleUpload()` wires `importReplacementConsumers` mutation for the replacement tab

---

## Contractor Flow (Mobile App — Backend APIs)

```
Site with installationType = REPLACEMENT:
  1. GET oldMeterReading(siteId) → check if already recorded
  2. Capture old meter photo → extract reading via OCR
  3. MUTATION recordOldMeterReading(siteId, finalReading, ocrReading, ...) 
  4. Proceed with standard createInstallation() flow
```

---
---

# Feature 6: Consumer Self-Registration

**Session:** 2026-03-09

Enables consumers **not** in the meter installation flow to self-register on the mobile app and access marketplace services. Implements smart phone-based account linking if the consumer is later imported by the admin.

## Migrations

| File | Contents |
|---|---|
| `0019_gigantic_agent_brand.sql` | `self_registered` boolean column on `consumers` table |

```bash
cd user-service && pnpm db:migrate
```

---

## Backend Changes (`user-service/`)

### Database Schema

#### `src/database/schema/consumers.ts` (`consumers` table)
- `selfRegistered` — `boolean`, default `false`
  - `true` = registered via mobile app
  - `false` = imported by admin (legacy default)

---

### Auth Types, Service & Resolver

#### `src/modules/auth/auth.types.ts`
- Added `SelfRegisterConsumerInput` InputType — `{ name: string, phone: string }`

#### `src/modules/auth/auth.service.ts`
- Added **`selfRegisterConsumer(input)`** method:
  1. Validates phone not already registered as CONSUMER
  2. Creates `users` row — `role=CONSUMER`, `isActive=true`, **no password** (OTP-only login)
  3. Creates `consumers` row — `selfRegistered=true`, auto `consumerId` prefix `SR-<timestamp>`
  4. Returns `AuthPayload` (access + refresh tokens) — can login immediately

#### `src/modules/auth/auth.resolver.ts`
- Added **`selfRegisterConsumer(input)`** mutation — **public, no auth guard**

---

### Sites Service — "Send Link" Phone Merge Fix

#### `src/modules/sites/sites.service.ts` — `activateConsumerSite()`

**Before:** Always created a new `users` row if `consumer.userId` was null.

**After:** Before creating a new user, checks if a `CONSUMER` user with the same phone already exists:
- **Found (self-registered):** Links existing user to consumer record — no duplicate created
- **Not found:** Creates new user (standard activation, unchanged)

SMS is only sent when a genuinely new user is created.

---

## GraphQL API Reference

| Operation | Type | Auth | Description |
|---|---|---|---|
| `selfRegisterConsumer(input)` | Mutation | **None (public)** | Self-register + get auth tokens immediately |
| `requestLoginOtp(phone, CONSUMER)` | Mutation | None | Request OTP for consumer login |
| `verifyLoginOtp(phone, otp, CONSUMER)` | Mutation | None | Verify OTP and get auth tokens |
| `myConsumer` | Query | CONSUMER | Get consumer profile + linked sites |

---

# Feature 7: RO Creation & Validation Fix

**Session:** 2026-03-09

Resolved a critical bug where creating a Retail Outlet with an existing email or code would show a success message but fail to create the record due to unhandled GraphQL errors on the frontend.

## Backend Changes (`user-service/`)
- **`RetailOutletsService.create`**: Refined the `ConflictException` handling to ensure duplicate `email` or `code` is caught and thrown correctly with a user-friendly message.

## Frontend Changes (`web-frontend/`)
- **`src/app/(admin)/admin/ros/page.tsx`**: Updated `handleCreateRO` to correctly parse `graphQLErrors` from the Apollo Client response. Error messages (e.g., "User email already exists") are now displayed in a red alert box above the form, rather than being swallowed.

---

# Feature 8: Admin Consumers & Site UI Cleanup

**Session:** 2026-03-09

Streamlined the Admin interface by removing non-functional, redundant, or confusing UI elements from the Consumers and Site management views.

## Frontend Changes (`web-frontend/`)

### Consumers Listing (`/admin/consumers`)
- Removed the **Export** button (non-functional).
- Removed the **3-dots (MoreVertical)** action menu from table rows to simplify the layout.

### Consumer Details (`/admin/consumers/[id]`)
- Removed the **Edit** button from the header.
- Removed **View Documents** and **Add New Site** buttons from the bottom action bar.

### Site Details (`/admin/sites/[id]`)
- **Contractor Assignment**: Removed the interactive assignment form. It now displays a clean "No contractor assigned" or lists the assigned contractor in a read-only format.
- Removed the redundant **View Consumer** button from the bottom action bar.

---

# Feature 9: Marketplace Rating Summary Enhancement

**Session:** 2026-03-09

Enhanced the `marketplaceRatingSummary` query to provide raw review comments while maintaining strict reviewer anonymity.

## Backend Changes (`user-service/`)

### GraphQL Schema
- **`src/modules/marketplace/ratings/ratings.types.ts`**: Added `anonymousReviews: [String!]!` to the `RatingSummary` ObjectType.

### Ratings Service
- **`src/modules/marketplace/ratings/ratings.service.ts`**: Updated `getRatingSummary` to fetch all non-null comments for a user.
- Uses PostgreSQL `ARRAY_AGG` and `ARRAY_REMOVE` to aggregate comments into a single array, ordered by most recent first.
### [REFINEMENT] Marketplace Go-Live Checklist Transparency
**Goal:** Provide unmistakable feedback to admins about readiness.

**Backend Changes:**
- Enhanced `getGoLiveChecklist` with gates for **Marketplace Pricing (Base Rates)** and **Service-specific SLAs (Recommended)**.
- Refined check labels: "Global Fallback SLA (Essential)" vs "Service-specific SLAs (Recommended)".
- Removed the "Live Contractors" requirement to allow registration *after* readiness.

**Frontend Changes:**
- Integrated the `GoLiveChecklistWidget` directly into the Marketplace Management page dashboard.
- Added dynamic checkmark icons to the **Setup Flow Guide Cards** linked to backend status.
- Implemented a high-priority gradient alert on the main Admin Dashboard for pending setup steps.

### [REFINEMENT] Subscription Plans UI Refactor
**Goal:** Highlight plan names as the primary identifier.

**Frontend Changes:**
- Redesigned plan cards in `/admin/marketplace/subscriptions`.
- Introduced a **Bold Violet Anchor Box** on the left for the **Plan Name** (e.g., PRO, ENTERPRISE).
- Relegated duration and version to secondary status for faster visual recognition.
- **Privacy Rule**: Only the comment text is returned. All rater identifiers (ID, Name, Phone) are strictly excluded from this query.

## Frontend Usage
The frontend can now fetch `anonymousReviews` within the `marketplaceRatingSummary` query to display a "Wall of Love" or feedback list for any user in the marketplace.

---

## Account Flow Summary

| Scenario | What happens at "Send Link" |
|---|---|
| Consumer self-registers → admin imports same phone → RO clicks Send Link | Existing user linked to imported consumer, `isActivated=true`. No duplicate user. |
| Admin imports consumer → RO clicks Send Link | New user created, SMS sent (unchanged). |
| Consumer self-registers only (no import) | Uses marketplace immediately. No site until admin imports. |

