# Marketplace Flow Revamp — Session Changelog
**Date:** 2026-03-23  
**Session:** `395779a7-cabc-42d8-a4b7-c73d3c619b21`

---

## Overview
This session implemented the first batch of the 6-feature contractor marketplace revamp. The DB migration was generated and applied successfully.

---

## DB-Level Changes

### Migration Applied
`user-service/src/database/migrations/0023_watery_swarm.sql`

### Modified Tables

#### `mp_contractor_profile`
| Column | Type | Purpose |
|---|---|---|
| `is_available_for_jobs` | `boolean DEFAULT true` | Contractor availability toggle |
| `commission_override_percent` | `decimal(5,2) NULL` | Per-contractor commission override (FREE only) |
| `is_blacklisted` | `boolean DEFAULT false` | Blacklist status flag |
| `blacklisted_at` | `timestamp with tz` | When auto-blacklisted |
| `blacklist_reason` | `text` | Auto-generated reason string |
| `blacklist_revoked_at` | `timestamp with tz` | When admin revoked blacklist |
| `blacklist_revoked_by` | `uuid FK → users.id` | Admin who revoked |
| `blacklist_revocation_reason` | `text` | Admin's revocation reason |

#### `mp_jobs`
| Column | Type | Purpose |
|---|---|---|
| `payout_contractor_id` | `uuid FK → contractors.id NULL` | For subcontractor payout routing to parent |
| `payment_requested_at` | `timestamp with tz NULL` | When payment link was sent (post-acceptance) |
| `payment_due_by` | `timestamp with tz NULL` | Payment expiry deadline |

#### `mp_job_status` enum (renamed values)
- Old `CANCELLED` → renamed to `CANCELLED_BY_CONSUMER`
- Added: `PAID` (payment confirmed, between `PAYMENT_PENDING` and `STARTED`)
- **New flow:** `REQUESTED → ACCEPTED → PAYMENT_PENDING → PAID → STARTED → COMPLETED → CLOSED`
- **Old flow:** `PAYMENT_PENDING → REQUESTED → ACCEPTED → STARTED → COMPLETED → CLOSED`

### New Tables

#### `mp_blacklist_config`
Singleton table storing admin-configurable SLA breach thresholds.
| Column | Default |
|---|---|
| `max_acceptance_breaches` | 3 |
| `max_start_breaches` | 3 |
| `max_completion_breaches` | 3 |
| `total_breach_limit` | 5 |
| `window_days` | 30 |

#### `mp_blacklist_revocations`
Audit trail for every admin blacklist reversal.
| Column | Type |
|---|---|
| `contractor_id` | `uuid FK → contractors.id` |
| `revoked_by` | `uuid FK → users.id` |
| `reason` | `text NOT NULL` |
| `breach_count_at_revocation` | `integer` |
| `revoked_at` | `timestamp with tz` |

#### `mp_subcontractor_mappings`
Links a subcontractor to a parent PREMIUM contractor with an optional region.
| Column | Type |
|---|---|
| `parent_contractor_id` | `uuid FK → contractors.id` |
| `sub_contractor_id` | `uuid FK → contractors.id` |
| `is_active` | `boolean DEFAULT true` |
| `is_suspended_pending_plan` | `boolean DEFAULT false` |

---

## API Changes (GraphQL Mutations & Queries)

### New Mutations

| Mutation | Role | Description |
|---|---|---|
| `toggleMyAvailability(available: Boolean!)` | CONTRACTOR | Toggle availability for new jobs |
| `setContractorCommission(contractorId, commissionPercent?)` | ADMIN | Set/clear commission override for FREE contractor |
| `revokeContractorBlacklist(contractorId, reason)` | ADMIN | Revoke blacklist, reset breach count, log audit |
| `updateBlacklistConfig(...)` | ADMIN | Update SLA breach thresholds & rolling window |
| `createSubContractor(name, phone, coverageAreas)` | CONTRACTOR (PREMIUM) | Provision new subcontractor + assign locations |
| `removeSubContractor(subContractorId)` | CONTRACTOR (PREMIUM) | Soft-remove subcontractor |

---

## Service Methods Added
All in `ContractorProfileService`:
- `toggleAvailability(contractorId, available)`
- `setContractorCommission(contractorId, percent|null)` — enforces mutual exclusivity with PREMIUM subscription
- `checkAndEnforceBlacklisting(contractorId)` — called after each SLA breach; auto-blacklists on threshold breach
- `revokeBlacklist(adminUserId, contractorId, reason)` — transactional: creates audit entry + resets profile
- `getBlacklistConfig()` + `updateBlacklistConfig(adminUserId, config)` — handles "Rolling Window" days.
- `createSubContractor(parentId, input)` — transactional: creates User, Contractor, MP Profile, Mapping, and Locations.
- `removeSubContractor(parentId, subId)` — soft-delete
- `getSubContractors(parentId)`

---

## Bug Fixes
- `CANCELLED` enum value renamed to `CANCELLED_BY_CONSUMER` across:
  - `jobs.service.ts` (`cancelJob`)
  - `payments.service.ts` (webhook failure handler)
  - `payment-expiry.service.ts` (cron scheduler)
  - `ratings.service.ts` (rating eligibility checks)

---

## Frontend Changes
- `web-frontend/src/app/(contractor)/contractor/marketplace/page.tsx`  
  Added availability toggle card (tap to toggle online/offline). Loads current state from profile, calls `toggleMyAvailability` mutation, shows real-time visual feedback.

---

## Later Session Updates (2026-03-23)

### Backend API Changes
- **`searchContractors` (Query):** 
  - Rewritten as a 3-way `UNION` to return `PREMIUM` contractors, `FREE` (commission-based) contractors, and `Subcontractors` mapped to `PREMIUM` parents. 
  - **Individual Coverage:** Subcontractors use their own explicitly assigned coverage areas (set by Parent) for search matching, rather than inheriting the entire Parent territory.
  - Subcontractors are excluded from search results if `is_suspended_pending_plan` is true.
  - Added new fields `payoutContractorId` (ID of the parent for subcontractors) and `commissionPercent` to the `ContractorSearchResult` GraphQL type.
- **`createMarketplaceJob` (Mutation):** 
  - Shifted from a payment-first model to a **post-acceptance payment flow**. 
  - Jobs now start in the `REQUESTED` state without a `paymentSessionId` or `paymentUrl`.
  - Amount is still snapshotted at the time of creation (includes commission calculations for `FREE` contractors).
- **`acceptJob` (Mutation):** 
  - Now handles payment initialization. When a contractor accepts a job, its state transitions to `PAYMENT_PENDING` and the payment session is created explicitly for the consumer to act on.
- **`nearestVillage` (Query):** 
  - Added a new public `nearestVillage(lat, lng, radiusKm)` query implementing the Haversine formula directly in PostgreSQL to fetch the closest classified village within a specified radius (default 50km).
- **`initiateMarketplacePayment` (Mutation):**
  - Added a new mutation for consumers to retry or re-initiate payment when a job is in the `PAYMENT_PENDING` state. 
- **`SubscriptionsService` updates:**
  - `checkExpiredSubscriptions()` was updated to automatically suspend a PREMIUM contractor's subcontractors (`is_suspended_pending_plan = true`) when their parent's plan expires.
  - `finalizeSubscriptionUpgrade()` was updated to clear `is_suspended_pending_plan` flags automatically upon plan renewal.

### Frontend Changes
- **Contact Us Page (`/contact`):**
  - Fully re-designed with a premium aesthetic, featuring information cards and an interactive form.
  - Form schema was synced with the backend API `/api/contact` structure (`name`, `email`, `company`, `mobile`, `purpose` enum, `message`, `token`).
  - Added invisible Google reCAPTCHA v2 execution prior to form submission.
  - Re-wired the endpoint to dynamically use the `NEXT_PUBLIC_API_URL` environment variable.
  - "Contact Us" link added to the main landing page footer alongside "Terms" and "Privacy".
- **reCAPTCHA Integration in Auth Flow:**
  - Added `react-google-recaptcha` package and implemented a standard, reusable `useRecaptcha` hook.
  - Embedded invisible reCAPTCHA on the **Login Form**, intercepting execution before Email/Password submission as well as Phone OTP requests.
  - Embedded invisible reCAPTCHA on the **Forgot Password Form**, intercepting execution before submitting an email for a reset code.
  - `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` values embedded correctly into `.env.development`, `.env.staging`, and `.env.production`.

### Admin Feature Completions
- **Subscription Plans Settings:** Added `maxSubcontractors` input field mapping to the backend, enabling Admins to define limits per subscription blueprint.
- **Contractors Management (`/admin/marketplace/contractors`):**
  - Integrated `useSetContractorCommission` to allow setting a percentage override on FREE tier contractors.
  - Implemented `useRevokeContractorBlacklist` to display current blacklist flags with reasons and allow Admins to easily revoke them with an audit reason.
- **Blacklist Settings Page (`/admin/marketplace/blacklist`):**
  - Built a standalone settings page dynamically fetching and updating `GET_BLACKLIST_CONFIG` thresholds (window days, SLA limits).

---

## Still Pending (Next Session)

**Frontend — Consumer:**
- Consumer "Pay Now" screen (appears after contractor accepts, when job is `PAYMENT_PENDING`)
- GPS village selection integrating `nearestVillage` GraphQL query + "Use Location" logic
- Consumer UI: "Blacklisted" or "Not Available" badge in search results

**Frontend — Contractor:**
- Subcontractor management page (`/contractor/marketplace/subcontractors`)

**Infrastructure & Automation:**
- **Webhook Dynamic Notification (`PENDING`):** Integration to dynamically pass `notify_url` to Cashfree. Currently requires manual Dashboard configuration. Requires `BACKEND_URL` in `.env` for future activation.
