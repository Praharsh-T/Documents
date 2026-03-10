#  Smart Meter System - Update Log

**Last Updated:** 10 March 2026 (Cashfree Integration & Contractor Logic Refinement)

---

## 📋 Latest Session Context (10 March 2026 — Cashfree Integration & Contractor Logic Refinement)

### What Was Done This Session:

#### **1. CASHFREE PAYMENT GATEWAY (REAL INTEGRATION)**

- **Cashfree SDK**: Replaced mock payments with real Order and Refund flows using the official `cashfree-pg` SDK.
- **Webhook Security**: Implemented HMAC signature verification for the Cashfree Webhook endpoint to ensure data integrity.
- **Automated Upgrades**: Subscription upgrades now happen instantly upon payment confirmation via webhook background processing.
- **Refund Management**: Enabled Admin-initiated refunds directly from the Job Detail view.

#### **2. DISCOVERY & QUOTA ENFORCEMENT**

- **Search Filtering**: Contractors who hit their job quota (Premium plans) are now automatically hidden from consumer search results.
- **Strict Free Tier**: Re-enforced strict marketplace blocking for FREE tier users while preserving their job counter history.

#### **3. CONTRACTOR CREATION & HIERARCHY**

- **Duplicate Validation**: Added explicit checks for existing emails in the `users` table during contractor creation to prevent generic DB errors.
- **RO Linking**: Automatically set `parentRoId` and `divisionId` for contractors created by Retail Outlets.

#### **4. API CATALOG ENHANCEMENT**

- **Tiered Pricing**: Enhanced `marketplaceServicesByCategory` to return Rural, Urban, and Semi-Urban prices simultaneously for every service.
- **Commission Percent**: Included service commissions in the public catalog API.

#### **5. FINANCIAL TRANSPARENCY**

- **Admin Ledger**: Created `/admin/marketplace/payments` page to track the status of all marketplace financial transactions.

**Files Changed:**

- `user-service/src/modules/payments/cashfree.service.ts`
- `user-service/src/modules/payments/payments.service.ts`
- `user-service/src/modules/marketplace/subscriptions/subscriptions.service.ts`
- `user-service/src/modules/marketplace/contractors/contractor-profile.service.ts`
- `user-service/src/modules/marketplace/services-catalog/services.service.ts`
- `web-frontend/src/app/(admin)/admin/marketplace/payments/page.tsx`

---

## 📋 Latest Session Context (06 March 2026 — Auth Throttle Env-Gating & Ratings Status Rules)

### What Was Done This Session:

#### **1. LOGIN THROTTLE DISABLED IN DEV & STAGING**

- `ThrottleService` now reads `NODE_ENV` at startup via `ConfigService`. In any environment where `NODE_ENV !== 'production'`, `check()` returns immediately (no lock check) and `recordFailure()` returns a no-op response without touching the DB.
- `ConfigModule` added to `AuthModule` module-level imports so `ConfigService` is injectable into `ThrottleService` via NestJS DI.
- Production behaviour is fully preserved — `resetOnSuccess` (record cleanup) still runs everywhere.

**Files Changed:**

- `user-service/src/modules/auth/throttle.service.ts`
- `user-service/src/modules/auth/auth.module.ts`

---

#### **2. MARKETPLACE RATINGS — EXPANDED STATUS RULES**

Old behaviour: `createMarketplaceRating` threw `400 "Can only rate completed jobs"` for any non-`COMPLETED` job.

New behaviour — role-specific status gates:

| Rater                 | Allowed Statuses                     |
| --------------------- | ------------------------------------ |
| Consumer → Contractor | `COMPLETED`, `REJECTED`, `CANCELLED` |
| Contractor → Consumer | `COMPLETED`, `CANCELLED`             |

This enables:

- Consumer rating a contractor who **rejected** their paid job
- Contractor rating a consumer who **cancelled** after the service started
- Both parties rating on normal **COMPLETED** jobs (unchanged)

Side-effects preserved:

- Contractor aggregate stats update whenever consumer rates (all allowed statuses)
- Job auto-close to `CLOSED` only fires on consumer rating a `COMPLETED` job

**File Changed:**

- `user-service/src/modules/marketplace/ratings/ratings.service.ts` — rewrote `createRating()` with role-resolved status gates

---

### ⚠️ Pending / Known Gaps After This Session:

| #   | Action                                                                     | Who       | Blocking?                                                              |
| --- | -------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------- |
| 1   | Investigate `createMarketplaceJob` "request failed" error from mobile app  | Developer | ✅ Yes — root cause unconfirmed (likely `consumers.userId` not linked) |
| 2   | All previously logged pending items (Cashfree, SMS, Redis payment intents) | Developer | Production                                                             |

---

## 📋 Previous Session Context (02 March 2026 — Marketplace Redesign & Plan Versioning)

### What Was Done This Session:

#### **1. SUBSCRIPTION PLAN VERSIONING**

- Introduced immutable versioning to `SubscriptionPlan` to ensure grandfathered pricing for existing Premium contractors.
- `updateSubscriptionPlan` now retires old terms and inserts a `version + 1` record.
- Added a "Log" interface to the Admin Hub to trace the pricing history of Monthly/Quarterly/Yearly plans over time.

#### **2. MARKETPLACE HUB REDESIGN (`/admin/marketplace`)**

- Replaced the confusing flat-grid layout with a structured, step-by-step Setup Flow:
  1. Locations
  2. Units of Measurement
  3. Service Categories
  4. Services & Pricing
- Clustered daily-use elements (Jobs, Contractors, SLA Matrices, Global SLA, Subscriptions, Logs) into an "Operations & Rules" section below.

#### **3. INTEGRATED SERVICES & PRICING UI**

- Removed the standalone "Pricing Configuration" router page.
- Injected `Urban`, `Semi-Urban`, and `Rural` base price inputs directly into the Add/Edit Service form.
- The UI now sequences `createMarketplaceService` followed synchronously by `bulkSetMarketplacePrices`, enabling single-step catalog creation.

---

## 📋 Latest Session Context (27 Feb 2026 Evening — Marketplace Flow Testing & Debugging)

### What Was Done This Session:

#### **1. `searchMarketplaceContractors` — ROOT CAUSE IDENTIFIED & RESOLVED (Testing)**

**Symptom:** Search returned empty results `{ items: [], total: 0 }` even after migration was applied and backend restarted.

**Diagnostic Query Run:**

```sql
SELECT
  mp.contractor_id,
  mp.subscription_type,
  mp.is_marketplace_active,
  c.is_active,
  (SELECT COUNT(*) FROM mp_contractor_service_mapping WHERE contractor_id = c.id AND is_active = true) AS services,
  (SELECT COUNT(*) FROM mp_contractor_coverage_selections WHERE contractor_id = c.id AND is_active = true) AS coverage_entries
FROM mp_contractor_profile mp
JOIN contractors c ON c.id = mp.contractor_id;
```

**Result:**

```
contractor_id                          subscription_type  is_marketplace_active  is_active  services  coverage_entries
c9706131-7039-4e61-b598-1890a228feb4   PREMIUM            true                   true       0         2
```

**Root Cause:** `services = 0` — The contractor had coverage entries and PREMIUM status, but **no entries in `mp_contractor_service_mapping`**. The search query uses `INNER JOIN mp_contractor_service_mapping`, so zero service mappings = zero results regardless of everything else.

**Fix Applied:** Used `assignContractorServices` (Admin) / `updateMyContractorServices` (Contractor) to map at least one service to the contractor. Search immediately returned results after this.

---

#### **2. SEARCH INPUT FLOW DOCUMENTED**

**How to get `locationId` (village UUID from `mp_location_master`):**

```
Step 1: marketplaceStates                          → get stateCode
Step 2: marketplaceDistricts(stateCode)            → get districtCode
Step 3: marketplaceVillages(districtCode, classifiedOnly: true)  → get village id = locationId
```

⚠️ `classifiedOnly: true` is required for consumers — only returns villages where admin has set `area_type`. If no villages return, it means admin hasn't classified any villages yet.

**How to get `serviceId`:**

```graphql
marketplaceServices { items { id name isActive } }
# OR
marketplaceServicesWithPrices(areaType: URBAN) { id name price }
```

**5-condition checklist for a contractor to appear in search:**

1. `mp.is_marketplace_active = true` ✅
2. `c.is_active = true` ✅
3. `mp.subscription_type = 'PREMIUM'` ✅ (FREE contractors are permanently excluded)
4. Entry in `mp_contractor_service_mapping` for that `serviceId` with `is_active = true` ← **was missing**
5. Entry in `mp_contractor_coverage_selections` matching the village's district/sub-district/village

---

#### **3. FULL MARKETPLACE JOB LIFECYCLE DOCUMENTED**

Complete end-to-end flow verified and documented:

| Step | Mutation                              | Role           | Notes                                                                                          |
| ---- | ------------------------------------- | -------------- | ---------------------------------------------------------------------------------------------- |
| 1    | `createMarketplaceJob`                | CONSUMER       | Creates job in `PAYMENT_PENDING`; requires pricing + SLA configured                            |
| 2    | `simulatePaymentSuccess`              | Any (DEV ONLY) | Moves `PAYMENT_PENDING → REQUESTED`; replace with Cashfree webhook in prod                     |
| 3    | `acceptMarketplaceJob`                | CONTRACTOR     | Moves to `ACCEPTED`; auto-generates START_JOB OTP; sends OTP to consumer                       |
| 3b   | `rejectMarketplaceJob`                | CONTRACTOR     | Moves to `REJECTED`; refund TODO                                                               |
| 4    | `activeJobOtp`                        | CONSUMER       | Query to get current OTP code (since SMS is disabled in dev)                                   |
| 5    | `verifyMarketplaceOtp` (START_JOB)    | CONTRACTOR     | Consumer reads OTP → contractor enters → `ACCEPTED → STARTED`; COMPLETE_JOB OTP auto-generated |
| 6    | `verifyMarketplaceOtp` (COMPLETE_JOB) | CONTRACTOR     | Consumer reads OTP → `STARTED → COMPLETED`                                                     |
| 7    | `createMarketplaceRating`             | Any            | Consumer rates contractor; contractor stats updated                                            |

**OTP fallback (if expired):**

```graphql
generateMarketplaceOtp(input: { jobId: "<id>", otpType: START_JOB })
```

**Job state machine:**

```
PAYMENT_PENDING → REQUESTED → ACCEPTED → STARTED → COMPLETED → CLOSED
                                       ↘ REJECTED
PAYMENT_PENDING / REQUESTED / ACCEPTED → CANCELLED (consumer only)
```

---

#### **4. ONE JOB = ONE SERVICE = ONE PAYMENT (Architecture Clarification)**

**Confirmed:** There is no "add service to existing job" or "multi-service job" concept. Each job is a single-service, single-payment work order with an **immutable price snapshot** locked at creation time (`unit_price_snapshot`, `total_price_snapshot`).

**If consumer wants multiple services:**

- Create 2 separate jobs → 2 separate payments

**If consumer picked wrong service before paying:**

- Cancel job (`PAYMENT_PENDING` → `CANCELLED`, free — no payment yet)
- Create new job with correct service

**If consumer cancels after payment:**

- Job cancellable from `REQUESTED` or `ACCEPTED` status
- **No automatic refund** — manual process until Cashfree is integrated (known TODO)

---

#### **5. PREREQUISITES FOR `createMarketplaceJob` TO SUCCEED**

Two admin-side configurations must exist before any consumer can book:

| Prerequisite                             | Where to configure           | GraphQL mutation           |
| ---------------------------------------- | ---------------------------- | -------------------------- |
| **Pricing** for `serviceId` + `areaType` | `/admin/marketplace/pricing` | `createMarketplacePricing` |
| **SLA** for `serviceId` + `areaType`     | `/admin/marketplace/sla`     | `createMarketplaceSla`     |

If either is missing, `createMarketplaceJob` throws:

- `"No pricing configured for this service and area"`
- `"No SLA configured for this service and area"`

---

### ⚠️ Known Gaps Confirmed During Testing:

| Gap                                                  | Impact                                                                                |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `SMS_GATEWAY_ENABLED=false`                          | OTPs generated but not SMS-delivered; consumer must use `activeJobOtp` query manually |
| `paymentUrl: null` (Cashfree mock)                   | Consumer can't pay via link; use `simulatePaymentSuccess` in dev                      |
| No refund on rejection/cancellation                  | Manual process; `// TODO: Trigger refund process` in jobs.service.ts                  |
| `ratedUserId` must be `users.id` not `contractor.id` | In `createMarketplaceRating` — easy to confuse                                        |

---

## 📋 Previous Session Context (27 Feb 2026 — Contractor Coverage Areas, Search Fix & UI Fixes)

### What Was Done This Session:

#### **1. CONTRACTOR SELF-SERVICE COVERAGE AREAS ✅ NEW FEATURE**

**Architecture Decision:** Replaced admin-assigned location mappings (`mp_contractor_location_mapping`) with contractor self-service coverage selections. Contractors choose their own district/sub-district/village coverage from their marketplace dashboard — PREMIUM-only (FREE tier cannot set coverage or appear in search).

**New DB Table** (`mp_contractor_coverage_selections`):

| Column           | Type                              | Notes                                   |
| ---------------- | --------------------------------- | --------------------------------------- |
| `id`             | uuid PK                           |                                         |
| `contractor_id`  | uuid FK → contractors             |                                         |
| `selection_type` | `mp_coverage_selection_type` enum | `DISTRICT` / `SUB_DISTRICT` / `VILLAGE` |
| `reference_code` | varchar(100)                      | LGD code or village UUID                |
| `display_name`   | varchar(500)                      | Human-readable label                    |
| `is_active`      | boolean                           | default true                            |
| `created_at`     | timestamp                         |                                         |

Unique index on `(contractor_id, selection_type, reference_code)`.

**New Enum** (`enums.ts`):

```typescript
coverageSelectionTypeEnum = pgEnum("mp_coverage_selection_type", [
  "DISTRICT",
  "SUB_DISTRICT",
  "VILLAGE",
]);
```

**Migration:** `0014_sweet_silver_samurai.sql` — generated via `drizzle-kit generate`. **Must be applied to DB.**

**New GraphQL APIs:**

```graphql
Query:    myContractorCoverageAreas          # CONTRACTOR role
Mutation: updateMyContractorCoverageAreas    # CONTRACTOR role, PREMIUM-only gate
```

**New GraphQL Types:**

- `ContractorCoverageSelection` ObjectType
- `CoverageSelectionInput` InputType (`selectionType`, `referenceCode`, `displayName`)
- `UpdateCoverageAreasInput` InputType (`selections: CoverageSelectionInput[]`)
- `CoverageSelectionType` enum registered via `registerEnumType`

**Service Methods (`contractor-profile.service.ts`):**

- `getMyCoverageAreas(contractorId)` — returns all selections ordered by `createdAt`
- `updateMyCoverageAreas(contractorId, selections[])` — PREMIUM check via `ForbiddenException`, delete-all + bulk insert

**`searchContractors()` Rewrite — Hierarchical Coverage Matching:**

Old: `INNER JOIN mp_contractor_location_mapping` (exact village UUID only)

New: `EXISTS` subquery with 3-tier match:

```sql
EXISTS (
  SELECT 1 FROM mp_contractor_coverage_selections mcs
  WHERE mcs.contractor_id = mp.contractor_id AND mcs.is_active = true
  AND (
    (mcs.selection_type = 'VILLAGE'      AND mcs.reference_code = :villageId)
    OR (mcs.selection_type = 'SUB_DISTRICT' AND mcs.reference_code = :subDistrictCode)
    OR (mcs.selection_type = 'DISTRICT'     AND mcs.reference_code = :districtCode)
  )
)
```

Also enforces `mp.subscription_type = 'PREMIUM'` — FREE contractors never appear.

**Frontend Changes:**

| File                                                    | Change                                                                                                                                                         |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `(contractor)/contractor/marketplace/coverage/page.tsx` | **NEW** — PREMIUM gate, cascading district→sub-district→village picker, colour-coded chips (blue=DISTRICT, amber=SUB_DISTRICT, green=VILLAGE), save/cancel bar |
| `(contractor)/contractor/marketplace/page.tsx`          | Added "Coverage Areas" quick-action card with PREMIUM badge                                                                                                    |
| `hooks/marketplace/useMarketplaceContractorProfiles.ts` | Added `useMyContractorCoverageAreas()`, `useUpdateMyCoverageAreas()` hooks + `ContractorCoverageSelection` / `CoverageSelectionInput` TS interfaces            |
| `graphql/marketplace.ts`                                | Added `GET_MY_CONTRACTOR_COVERAGE_AREAS`, `UPDATE_MY_COVERAGE_AREAS` operations                                                                                |

**Backend Files Changed:**

| File                                                             | Change                                                         |
| ---------------------------------------------------------------- | -------------------------------------------------------------- |
| `database/schema/marketplace/enums.ts`                           | Added `coverageSelectionTypeEnum`                              |
| `database/schema/marketplace/contractor-extensions.ts`           | Added `contractorCoverageSelections` table                     |
| `database/migrations/0014_sweet_silver_samurai.sql`              | Auto-generated DDL (CREATE TYPE + CREATE TABLE + FK + indexes) |
| `modules/marketplace/contractors/contractor-profile.types.ts`    | New ObjectType + InputTypes                                    |
| `modules/marketplace/contractors/contractor-profile.service.ts`  | 3 new/updated methods                                          |
| `modules/marketplace/contractors/contractor-profile.resolver.ts` | 1 new Query + 1 new Mutation                                   |

---

#### **2. `searchMarketplaceContractors` INTERNAL_SERVER_ERROR FIX**

**Root Cause:** `${now}` (JavaScript `Date.toString()`) was interpolated directly into a raw SQL string, producing a value PostgreSQL could not parse as a timestamp.

**Fix:** Replaced `${now}` with `NOW()` in the raw SQL expression in `contractor-profile.service.ts`.

---

#### **3. LANDING PAGE & LOGO UI FIXES**

- **Removed** "Government Registered" badge from the landing page hero section
- **Fixed** logo whitespace in Next.js `Image` with `fill` layout: applied two-layer div wrapper (`relative w-10 h-10` outer → `absolute inset-0` inner) across **3 instances** (navbar, footer, association section)

---

#### **4. LOCATION CLASSIFICATION UI FIXES (Admin)**

- **Villages visible on district select** — removed the sub-district gate that was preventing village rows from displaying when only a district was selected (no sub-district chosen)
- **Edit-only classification** — village area classification is now triggered via a pencil icon per row; removed the always-visible inline select that allowed accidental bulk saves

---

### ⚠️ Pending After This Session:

| #   | Action                                                          | Who               | Blocking?                             |
| --- | --------------------------------------------------------------- | ----------------- | ------------------------------------- |
| 1   | Apply `0014_sweet_silver_samurai.sql` to dev DB                 | Developer         | ✅ Yes — new table doesn't exist yet  |
| 2   | Restart backend (`pnpm start:dev`)                              | Developer         | ✅ Yes — new resolver/enum not loaded |
| 3   | Cashfree subscription payment integration                       | Developer         | Production                            |
| 4   | Cashfree job payment (`initiateMarketplacePayment`)             | Developer         | Production                            |
| 5   | Move `paymentIntents` Map → DB/Redis                            | Developer         | Production                            |
| 6   | `confirmSubscriptionPayment` webhook-only guard                 | Developer         | Production                            |
| 7   | Enable SMS (`SMS_GATEWAY_ENABLED=true`) + TRAI DLT registration | Admin + Developer | Production                            |
| 8   | Deprecate `assignContractorLocations` mutation                  | Developer         | Non-blocking                          |

---

## 📋 Previous Session Context (26 Feb 2026 Evening — Subscription Flow & Frontend Env Badge)

### What Was Done This Session:

#### **1. VALIDATIONPIPE "property contractorId should not exist" — ROOT CAUSE & FIX**

**Error:** Every call to `initiateSubscriptionUpgrade` returned:

```json
{ "message": ["property contractorId should not exist"], "statusCode": 400 }
```

**Why it happened (two mechanisms combined):**

- `target: "ES2023"` in `tsconfig.json` → TypeScript emits class fields as `Object.defineProperty` calls, meaning `contractorId?: string` becomes an own enumerable property set to `undefined` on every class instance — even when the client sends no such field.
- `ValidationPipe({ forbidNonWhitelisted: true })` in `main.ts` → any property with no class-validator decorator on the class is rejected.

**Fix:** Added `@IsOptional() @IsString()` to the internal `contractorId` field. This whitelists it without adding `@Field()` so it never enters the GraphQL schema.

---

#### **2. BANK ACCOUNT DETAILS ON SUBSCRIPTION UPGRADE ✅ NEW**

**New DB columns on `mp_contractor_profile`:**

| Column                     | Type           | Notes                                         |
| -------------------------- | -------------- | --------------------------------------------- |
| `bank_account_number`      | `varchar(30)`  | Account number                                |
| `bank_ifsc_code`           | `varchar(15)`  | IFSC code                                     |
| `bank_account_holder_name` | `varchar(255)` | Account holder name                           |
| `bank_name`                | `varchar(255)` | Bank name (optional)                          |
| `bank_verified`            | `boolean`      | Default false, for future Cashfree verify API |

**Migration:** `0013_contractor_bank_details.sql` (run `pnpm run db:migrate`)

**New GraphQL Input:** `BankAccountInput` — validated with IFSC regex, `@MaxLength`, `@IsNotEmpty`

**Integration:** `initiateSubscriptionUpgrade` accepts optional `bankAccount` field. If provided, saves to profile (or updates if profile already exists, resets `bankVerified = false`).

---

#### **3. MARKETPLACE PROFILE CREATION FLOW (Architecture)**

**Correct flow established:**

```
Excel Import → auto-create FREE profile (isMarketplaceActive: false)
                        ↓
              Contractor purchases PREMIUM
                        ↓
              confirmPayment() → isMarketplaceActive = true
                        ↓
              Contractor appears in consumer search
```

**Changes:**

- `imports.resolver.ts`: `bulkImportContractors` transaction now inserts `mp_contractor_profile` rows (FREE, inactive) with `onConflictDoNothing` safety
- `subscriptions.service.ts`: `confirmPayment` sets `isMarketplaceActive = true`
- `subscriptions.service.ts`: `getSubscriptionStatus` returns graceful FREE default (no 404)
- `subscriptions.service.ts`: `initiateUpgrade` auto-creates profile if missing (safety net)

---

#### **4. CONTRACTOR ID SECURITY FIX**

- **Removed** `contractorId` from `InitiateSubscriptionUpgradeInput` client input
- Resolver derives `contractorId` from JWT (`findByUserId`) and injects it internally
- Eliminates 403 "You can only upgrade your own subscription" confusion where users were passing `userId` instead of `contractor.id`
- Service guards with `if (!input.contractorId) throw new BadRequestException`

---

#### **5. FRONTEND ENV BADGE**

- **New component:** `web-frontend/src/components/shared/EnvBadge.tsx`
- Reads `NEXT_PUBLIC_ENV` (set per `.env.development` / `.env.staging` / `.env.production`)
- `development` → green **DEV** badge at bottom-right
- `staging` → amber **STAGING** badge at bottom-right
- `production` → renders nothing
- Fixed position, `pointer-events-none`, `select-none`, `z-[9999]`

---

## 📋 Previous Session Context (26 Feb 2026 Morning — LGD Import Scaling & 401 Upload Fixes)

### What Was Done This Session:

#### **1. LGD EXCEL IMPORT SCALING & OVERRIDES**

- **Drizzle Batch UPSERTs (`locations.service.ts`):** Ripped out the sequential DB query loop causing application timeouts off 30,000+ row datasets. Re-architected with 1,000-size array chunks running PostgreSQL `onConflictDoUpdate` upserts against the `villageCode` constraint. Drastically increased write scaling.
- **Frontend State Override Pipeline:** Added an explicit State selection UI dropdown over the upload component. Super Admins can map an overarching State (e.g. Karnataka) bridging the payload through the GraphQL `$stateCode` variables, commanding the backend to overwrite whatever corrupt state strings are actually living inside the `.xlsx` file.
- **Header Tolerances & Adjustments:** Swapped logic to `parseExcelRaw` to discard unaccounted Excel columns dynamically. Handled new arrays for `Village Status` and `Village Version`. Mutated template examples to map perfectly against frontend enums (`Urban`, `Semi-Urban`, `Rural`).

#### **2. FILE UPLOAD 401 UNAUTHORIZED BUG**

- **Token Storage Isolation Gap:** Evaluated the REST Guard mechanisms when `POST /files/upload/private/imports` threw 401 Unauthorized returns on logged-in sessions. Noticed that file boundary fetches were strictly pulling authorization bearer fragments from `sessionStorage`, completely breaking the feature for anyone using `localStorage` via the "Remember Me" flag. Standardized all hooks onto the `storage.get('auth_token')` helper utility for parity.

---

## 📋 Previous Session Context (23 Feb 2026 - Branding, Login Overrides, Forgot Password, and Multi-Tenancy)

### What Was Done This Session:

#### **1. BRANDING & LOGIN OVERRIDES**

- Applied "KSLECA" branding fix across 12 instances, replacing outdated "KSLECAR".
- Removed Developer-only overriding parameter `isProd` from the `/login` screen. All environments now strictly present Retail Outlet Portal login directly. The Admin login is correctly hidden and accessible only via query parameter link mapping (`?role=admin`) in the footer.
- Established Remember Me dual storage (`localStorage` for long term retention vs `sessionStorage` for tab isolation).

#### **2. FORGOT PASSWORD WORKFLOW**

- Fully implemented Auth Service Backend mutations: `requestPasswordReset` (OTP Generation) & `resetPassword` (OTP Hash Verification). Included DLT constraints, anti-enumeration, and backend rate throttling.
- Fully implemented 3-Step Frontend form wizard on `/forgot-password` connecting React Hook Forms to Apollo GraphQL endpoints.

#### **3. CONSUMER & ADMIN MULTI-TENANCY**

- **Admin Association Filtering (`consumers.service.ts`):** `getAccessContext()` dynamically restricts admin visibility to the states explicitly assigned within their `Associations` table row. Unassigned Admins see zero records.
- **Consumer Import Protections (Backend):** Excel imports drop and ignore `Meter Number`, `Sold`, `Installed`, and `Service` fields explicitly to preserve integrity, as these fields must originate strictly from physical RO processes.

#### **4. RO WORKFLOWS & ROUTING CLEANUP**

- **Role Permissions:** Revoked `importMeters` Graph permissions from `SUPER_ADMIN` completely; re-assigned uniquely to `RETAIL_OUTLET`. Removed `SUB_USER` ability to render or navigate to Staff creation endpoints `/ro/staff`.
- **Bulk Imports & RAPDRP:** Migrated the Bulk Meter Upload functionality directly into the `/ro/inventory` space. Added the "Is RAPDRP?" boolean column across the Admin Template, the Single Form inputs, and the GraphQL schema.
- **Site Review Improvements:** Disabled edits to `requiredPhase` and `requiredVoltage` selects in the site-review, relying entirely on the Excel-originated fields. Bundled the consumer account verification execution `activateConsumerSite` directly within the contractor assignment submit handler `handleAssignAndSendLinks` to seamlessly send Registration SMS alongside assignment.
- **Contractors Panel:** Constructed the specific detail view page `/ro/contractors/[contractorId]` and updated the main list rows to establish active router links to these individual profiles.

---

## 📋 Previous Session Context (20 Feb 2026 - Subscriptions, Data Isolation & GraphQL Fixes)

### What Was Done This Session:

#### **1. CONSUMER DATA ISOLATION — Creator-Based (Breaking Change)**

**Problem:** ADMINs were seeing ALL consumers regardless of who created them. Required per-ADMIN isolation.

**Schema Change:**

- Added `createdBy: uuid` column to `consumers` table with FK → `users.id` and an index
- Migration: `0011_freezing_the_initiative.sql`

**Service Change (`consumers.service.ts`):**

- `getAccessContext()` now returns `createdByUserId` instead of state-based lookups
- ADMIN role: `WHERE consumers.created_by = :userId`
- RO/SUB_USER: unchanged (subdivision-based filtering)
- SUPER_ADMIN: sees all

**Data Isolation Matrix:**
| Role | Sees |
|------|------|
| SUPER_ADMIN | All consumers |
| ADMIN | Only consumers they created |
| RETAIL_OUTLET | Consumers with sites in their subdivision |
| SUB_USER | Same as parent RO |

---

#### **2. ROLE ENUM COMPARISON FIX**

**Problem:** `consumers.service.ts` compared `userContext.role` (UserRole enum) with raw string literals.
**Fix:** All comparisons changed to `UserRole.SUPER_ADMIN`, `UserRole.ADMIN`, `UserRole.RETAIL_OUTLET`, `UserRole.SUB_USER`

---

#### **3. CONTRACTOR PREMIUM SUBSCRIPTION MODULE ✅ NEW**

**6 New Files Created:**

| File                                       | Description                  |
| ------------------------------------------ | ---------------------------- |
| `subscriptions/subscriptions.types.ts`     | GraphQL types, enums, inputs |
| `subscriptions/subscriptions.service.ts`   | Business logic               |
| `subscriptions/subscriptions.resolver.ts`  | GraphQL queries + mutations  |
| `subscriptions/subscriptions.scheduler.ts` | Daily expiry cron job        |
| `subscriptions/subscriptions.module.ts`    | NestJS module                |
| `subscriptions/index.ts`                   | Barrel exports               |

**Subscription Plans:**
| Plan | Duration | Price | Discount | Final |
|------|----------|-------|----------|-------|
| MONTHLY | 1 month | ₹499 | 0% | ₹499 |
| QUARTERLY | 3 months | ₹1,497 | 10% | ₹1,347 |
| YEARLY | 12 months | ₹5,988 | 25% | ₹4,491 |

**Flow:** `subscriptionPlans` → `initiateSubscriptionUpgrade` → pay → `confirmSubscriptionPayment`

**Scheduler:** `@Cron(EVERY_DAY_AT_MIDNIGHT)` + runs 10s after server startup. Downgrades expired PREMIUM → FREE.

**Payment intents stored in-memory Map** (TODO: move to DB/Redis for production)

**GraphQL API Added:**

```
Queries:
  subscriptionPlans                               # Public
  mySubscriptionStatus                            # CONTRACTOR
  mySubscriptionHistory                           # CONTRACTOR
  contractorSubscriptionStatus(contractorId)      # ADMIN
  subscriptionPaymentIntent(id)                   # Auth

Mutations:
  initiateSubscriptionUpgrade(input)              # CONTRACTOR
  confirmSubscriptionPayment(input)               # CONTRACTOR
  adminSetContractorSubscription(input)           # ADMIN
```

---

#### **4. GRAPHQL TYPE FIXES — Service Category & UOM as Objects**

**Problem:** `category` and `uom` on `MarketplaceService` are object types (`ServiceCategoryType` / `ServiceUomType`), not scalars. `isPremiumOnly` does not exist on `MarketplaceService` — the correct field is `isFreeAvailable` on `ServiceCategoryType`.

**Files Fixed:**

**Frontend (`web-frontend`):**

- `src/graphql/marketplace.ts`
  - `MARKETPLACE_SERVICE_FRAGMENT`: expanded `category` → `{ id name code isFreeAvailable }`, `uom` → `{ id name code requiresQuantity }`, removed `isPremiumOnly`
  - `GET_MARKETPLACE_SERVICES_WITH_PRICES`: same + replaced `isPremiumOnly` with `isFreeAvailable`
  - `GET_MARKETPLACE_SERVICES_BY_CATEGORY`: `category` is now object, removed `isPremiumOnly`
- `src/hooks/marketplace/useMarketplaceServices.ts`
  - `MarketplaceService.category` changed from `ServiceCategory` string to `ServiceCategoryObject { id name code isFreeAvailable }`
  - `MarketplaceService.uom` changed from `ServiceUom` string to `ServiceUomObject { id name code requiresQuantity }`
  - `isPremiumOnly` removed; `isFreeAvailable` added to `ServiceWithPrice`
- `src/app/(admin)/admin/marketplace/services/page.tsx` — Uses `category?.code` for filter, `category?.name` for display, `!category?.isFreeAvailable` for Premium badge; removed `isPremiumOnly` checkbox from create modal
- `src/app/(admin)/admin/marketplace/pricing/page.tsx` — Uses `category?.name` and `uom?.name` instead of `.replace(/_/g, ' ')`
- `src/app/(admin)/admin/marketplace/sla/page.tsx` — Same fix

**Backend (`user-service`):**

- `contractor-profile.types.ts` — Added `@IsOptional()`, `@IsInt()`, `@Min(1)` to `page` and `limit` fields in `ContractorProfileFilterInput`. This fixed: `"property page should not exist, property limit should not exist"` BadRequestException.

---

#### **5. DOCUMENTATION UPDATES**

- `CONSUMER_API.md` → v3.3.0: Added full Marketplace section (browse, search, book job, track jobs, cancel, rate)
- `CONTRACTOR_API.md` → v3.3.0: Added full Marketplace section (profile, jobs, OTP flow, subscription upgrade)
- `MARKETPLACE_IMPLEMENTATION_PLAN.md` → Updated status, Appendix D added
- `SESSION_CHANGES.md` → Updated with both 19 Feb (evening) and 20 Feb sessions

---

## 📋 Previous Session Context (19 Feb 2026 - Backend Restructuring)

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

| Component                                  | Location                                | Status  |
| ------------------------------------------ | --------------------------------------- | ------- |
| Admin Associations SUPER_ADMIN restriction | Backend resolver                        | ✅ Done |
| Associations UI removed from Admin         | Frontend pages/hooks                    | ✅ Done |
| Admin creation with Association dropdown   | `/admin/admins/page.tsx`                | ✅ Done |
| Location import moved to Marketplace       | `/admin/marketplace/locations/page.tsx` | ✅ Done |
| Location import removed from Upload        | `/admin/upload/page.tsx`                | ✅ Done |
| Village Category made required             | Backend types & service                 | ✅ Done |
| Village Category in frontend               | Hooks & GraphQL                         | ✅ Done |

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

| Aspect                | Behavior                                                    |
| --------------------- | ----------------------------------------------------------- |
| Transaction           | All-or-nothing (1 failure = entire batch fails)             |
| Duplicate Consumer ID | Error with specific message                                 |
| Duplicate Phone       | Error with specific message                                 |
| Same File Upload      | Rejected by SHA-256 hash check                              |
| Modified File         | Allowed (different hash)                                    |
| Missing Hierarchy     | Auto-created during import                                  |
| Hierarchy Storage     | Stores IDs (circleId, divisionId, subdivisionId), not names |

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
if (meter.state !== "ASSIGNED" || !meter.assignedConsumerId) {
  throw new BadRequestException("Meter is not assigned to a consumer");
}

// After
if (
  meter.state !== "ASSIGNED" ||
  (!meter.assignedSiteId && !meter.assignedConsumerId)
) {
  throw new BadRequestException("Meter is not assigned to a site or consumer");
}
```

**Problem 2:** Verification approval failed when meter was still `ASSIGNED`.

**Fix:** Added meter state transition in verification approval:

```typescript
if (meter.state === "ASSIGNED") {
  try {
    await this.metersService.markInstalled(meterId, initialReading, reviewerId);
  } catch (err) {
    console.warn("Failed to mark meter as installed:", err);
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

| Issue                              | Root Cause                                   | Fix                                      |
| ---------------------------------- | -------------------------------------------- | ---------------------------------------- |
| Site stuck at VERIFICATION_PENDING | Direct INSTALLED→VERIFIED transition invalid | Added intermediate state transitions     |
| Evidence images not displaying     | Storage key not converted to signed URL      | Added fileUrl ResolveField               |
| Evidence upload 401 error          | Wrong auth token key                         | Fixed to use `mescom_auth_token`         |
| Meter state error on complete      | Only checked assignedConsumerId              | Added assignedSiteId check               |
| Meter state error on verify        | Meter still ASSIGNED when approving          | Added ASSIGNED→INSTALLED transition      |
| Installation not found             | No record created on meter assign            | Creates installation in createConnection |
| RO installations empty             | Wrong roId (user ID vs retailOutlets.id)     | Fixed lookup to use retailOutlets.id     |

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
    id: "<site-uuid>"
    newState: VERIFIED
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
- Seeded with  circles (MYS, BGM, MNG, DVG, SHP)

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

| Priority  | Feature                       | Notes                                                    |
| --------- | ----------------------------- | -------------------------------------------------------- |
| 🔴 High   | **Consumer Welcome Link**     | Send link with welcome chat status before initial review |
| 🔴 High   | **Division Filter in Assign** | Filter contractors by division when assigning jobs       |
| 🔴 High   | **File Storage Service**      | Needed for photo uploads (S3/local)                      |
| 🟡 Medium | **SMS Gateway Integration**   | Placeholder ready for MSG91/Twilio                       |
| 🟡 Medium | **Email Gateway Integration** | Placeholder ready for SendGrid/SES                       |
| 🟡 Medium | **Network Type UI in Assign** | Site requirement selection for network type/RAPDRP       |
| 🟡 Medium | **Super Admin Dashboard**     | UI for creating admins                                   |
| 🟢 Low    | **OCR for Meter Reading**     | Deferred per user request                                |
| 🟢 Low    | **QR Scanner Backend**        | Frontend component exists                                |

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

| Role        | Email/Phone            | Password/OTP            |
| ----------- | ---------------------- | ----------------------- |
| Super Admin | superadmin@mescom.gov  | admin123                |
| Admin       | admin@mescom.gov       | admin123                |
| RO          | ro@mescom.gov          | retail123               |
| Contractor  | contractor@example.com | contractor123           |
| Consumer    | (any phone number)     | OTP: **123456** (dummy) |

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
  schema: "./src/database/schema/index.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    url: "postgres://YOUR_USERNAME@localhost:5432/mescom",
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

| Role          | Can Create                     | Uses             |
| ------------- | ------------------------------ | ---------------- |
| SUPER_ADMIN   | ADMIN                          | Email + Password |
| ADMIN         | RETAIL_OUTLET                  | Email + Password |
| RETAIL_OUTLET | CONTRACTOR, SUB_USER, CONSUMER | Email + Password |
| CONTRACTOR    | -                              | Email + Password |
| SUB_USER      | -                              | Email + Password |
| CONSUMER      | -                              | Phone + OTP      |

---

## 🔄 GraphQL Queries Reference

### Authentication

```graphql
# Login with email/password
mutation {
  login(input: { email: "admin@mescom.gov", password: "admin123" }) {
    token
    user {
      id
      name
      role
    }
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
    user {
      id
      name
      role
    }
  }
}
```

### User Creation (Role-based)

```graphql
# Create Contractor (RO only)
mutation {
  createContractor(
    input: {
      name: "John Doe"
      email: "john@company.com"
      password: "password123"
      phone: "9876543210"
      company: "ABC Electric"
      licenseNumber: "LIC-2024-001"
    }
  ) {
    id
    name
    role
  }
}

# Create Consumer (RO only)
mutation {
  createConsumer(
    input: {
      name: "Consumer Name"
      phone: "9876543210"
      rrNumber: "RR123456"
      address: "123 Main St"
    }
  ) {
    id
    name
    role
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
import { MeterNetworkType } from "../meter-specs/meter-specs.types";
// NOT: export enum MeterNetworkType { ... }
```

### Issue: Email required but consumers use OTP

**Fix:** Made `email` and `password_hash` nullable in users table (migration 0003)

### Issue: Verify page not showing success state

**Fix:** Added success message state and proper navigation back to list

---

## 📊 Project Analysis - Pending Items (30 Jan 2026)

### ✅ Core Features - COMPLETED

| Module                                | Status | Notes                             |
| ------------------------------------- | ------ | --------------------------------- |
| Consumer Management                   | ✅     | CRUD, state machine, OTP login    |
| Site Management                       | ✅     | Full lifecycle (CREATED → BILLED) |
| Meter Registry                        | ✅     | Inventory, specs, QR scanning     |
| Site-Meter Connections                | ✅     | Domain entity with history        |
| Installations                         | ✅     | Contractor job flow               |
| Verifications                         | ✅     | RO approval workflow              |
| Audit Logging                         | ✅     | All state changes tracked         |
| Circle/Division/Subdivision Hierarchy | ✅     | Full API                          |
| Consumer Import with User Account     | ✅     | OTP login works                   |
| RO Data Isolation (Subdivision)       | ✅     | Fixed 29 Jan                      |
| Division filter in assign page        | ✅     | Added 29 Jan                      |
| Network type & RAPDRP display         | ✅     | Added 29 Jan                      |

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

| Issue                                    | Location            | Priority |
| ---------------------------------------- | ------------------- | -------- |
| `<img>` instead of Next.js `<Image>`     | Multiple pages      | Low      |
| useMemo dependency warnings              | assign/page.tsx     | Low      |
| Pickup folder exists but feature removed | contractor/pickup   | Low      |
| Dev-mode notification logging            | notification module | Medium   |

### 📋 Recommended Next Steps

1. **Test the full E2E flow** with demo data (consumer login → photo upload → RO review → meter assign → contractor install → RO verify → billing)
2. **Remove or hide contractor pickup page** if the flow skips it
3. **Add real SMS integration** for production (replace console.log)
4. **Run the app on mobile** to identify responsive design issues
