# SmartMeter Project — Master Context Reference

> **Sources analysed:** PROJECT_BRAIN.md, PROJECT_DESCRIPTION.txt, REQUIREMENT.md, SESSION_CHANGES.md, UPDATE_LOG.md, state.md, FLOW_DOCUMENTATION.md (all as of 06 March 2026)

---

## 1. Project Identity

**Name:**  Smart Meter Installation Management System  
**Client:**  (MSma) / **KSLECA** branding in production  
**Type:** Government-grade, lifecycle-governed, multi-role enterprise platform  
**Purpose:** Digitise the complete lifecycle of smart meter installation — consumer onboarding → site verification → meter assignment → contractor installation → RO verification → billing handoff

**Key principle:** Backend is the ONLY authority. Frontend NEVER enforces business logic.

---

## 2. Tech Stack

| Layer | Tech |
|---|---|
| Frontend | Next.js 16 (App Router, Turbopack), React 19.2, Tailwind CSS, Apollo Client, React Hook Form + Zod |
| Backend | NestJS 11, GraphQL (code-first), PostgreSQL 17, Drizzle ORM, JWT + Passport |
| File Storage | Local ✅ · Azure Blob ✅ · GCP ❌ (not implemented — throws runtime error) |
| SMS | Fast2SMS (DLT templates) — infrastructure built, partially connected |
| Email | AWS SES — infrastructure built, partially connected |
| Payments | Cashfree — integrated (Order creation, webhook verification, refunds). Ensure `CASHFREE_APP_ID` / `CASHFREE_SECRET` set in production env. |
| Package Manager | pnpm throughout |

**Dirs:** `user-service/` (NestJS backend) · `web-frontend/` (Next.js frontend)

---

## 3. Roles & Authentication

| Role | Login | Created by |
|---|---|---|
| SUPER_ADMIN | Email + Password (loginType=ADMIN) | Seeded from env |
| ADMIN | Email + Password | SUPER_ADMIN |
| RETAIL_OUTLET (RO) | Email + Password | ADMIN |
| SUB_USER | Email + Password | RO |
| CONTRACTOR | Phone + OTP | RO (or Excel import) |
| CONSUMER | Phone + OTP (dummy: `123456`) | RO (or Excel import) |

**Data isolation:** SUPER_ADMIN → all; ADMIN → consumers they created (createdBy FK); RO → their subdivision; CONTRACTOR → their jobs; CONSUMER → their sites.

**Auth flow:** `rememberMe=true` → `localStorage`; `false` → `sessionStorage`. Storage utility `storage.get('auth_token')` normalises both.

**Throttling:** `auth_throttle` table; time-based lockouts (5min/30min/2hr/24hr/48hr on failures). **Disabled in non-production** via `NODE_ENV` check in `ThrottleService`.

---

## 4. Core Domain Model

```
Consumer (1) ─── has many ──► ConsumerSite (1) ─── tracked by ──► SiteMeterConnection
                                                                           │
Meter (1) ──────── assigned via ─────────────────────────────────────── (many)
                                                                           │
                                                                     Installation ──► InstallationEvidence
                                                                           │
                                                                     Verification
```

### Site State Machine (CRITICAL — no steps skippable)
```
CREATED → SITE_PENDING → SITE_VERIFIED → INSTALLATION_SCHEDULED
       → METER_ASSIGNED → INSTALLATION_IN_PROGRESS → INSTALLED
       → VERIFICATION_PENDING → VERIFIED → BILLED
       
FAILED ← (any stage)
```
Tariff code is **locked/immutable** at `SITE_VERIFIED`.

### Meter State Machine
```
AVAILABLE → RESERVED → ASSIGNED → INSTALLED → ACTIVE → FAILED → DECOMMISSIONED
RESERVED → TIMEOUT/FAILURE → AVAILABLE (hourly cron)
```

---

## 5. Backend Modules — Status

### ✅ Fully Implemented (18)
Auth · Users · Consumers · Sites · Meters · MeterSpecs · SiteMeterConnections · Installations · InstallationEvidence · Verifications · Hierarchy · Audit · Scheduler · MeterReadings · RetailOutlets · Contractors · Associations · Marketplace (Services, Pricing, Ratings, Categories, UoMs, ContractorProfiles, Locations, Subscriptions)

### ⚠️ Partially Implemented (4)
| Module | Gap |
|---|---|
| **Payments** | Mock Cashfree (returns `MOCK_SESSION_*`). No real API calls, no webhook signature verification, no refunds. |
| **Jobs (Marketplace)** | 7 SMS TODOs + 2 refund trigger TODOs in `jobs.service.ts`. OTP generated but not SMS-delivered. |
| **SLA** | Breach detection runs (every 5 min) but 3 notification TODOs — no alerts sent and no auto-cancel/refund on breach. |
| **Files/Storage** | Local + Azure ✅. GCP throws `Error('GCP storage not yet implemented')` at `storage/index.ts:28`. |

### ❌ Not Implemented (1)
| Module | Detail |
|---|---|
| **Billing** | Module is an empty placeholder (`billing.service.ts` has only `TODO` comments). DB tables (`billing_exports`, `billing_export_items`) exist. Full flow must be implemented. |

---

## 6. Frontend Pages (59 total)

| Route Group | Pages | Key Areas |
|---|---|---|
| `(auth)` | 2 | Login, Register, Forgot Password (3-step wizard) |
| `(admin)/admin` | 18 | Dashboard, Admins, ROs, Consumers, Hierarchy, Upload, Analytics, Marketplace mgmt (Services, Categories, UOMs, Pricing, SLA, Locations, Contractors, Jobs, Subscriptions) |
| `(ro)/ro` | 14 | Dashboard, Site Review, Assign, Schedule, Inventory, Verify, Consumers, Contractors, Installations, Staff |
| `(consumer)/consumer` | 10 | Dashboard, Address, Photo, Status, Help, Marketplace (Browse, Services, Book, Jobs, Job Detail) |
| `(contractor)/contractor` | 15 | Dashboard, Installations, Install Detail, Pickup, Pre-Install, Issues, Reading, Marketplace (Overview, Profile, Jobs, Job Detail, Coverage Areas) |

---

## 7. Marketplace System (Major Feature)

### Job Lifecycle
```
PAYMENT_PENDING → REQUESTED → ACCEPTED → STARTED → COMPLETED → CLOSED
                                        ↘ REJECTED
PAYMENT_PENDING / REQUESTED / ACCEPTED → CANCELLED
```

### Key Rules
- 1 job = 1 service = 1 payment (immutable price snapshot at creation)
- PREMIUM contractors only appear in search (`subscription_type = 'PREMIUM'`, `is_marketplace_active = true`)
- Contractor must have: service mapping + coverage area selection + PREMIUM subscription
- Coverage areas: hierarchical (DISTRICT > SUB_DISTRICT > VILLAGE) — contractor self-selects

### Subscription Plans
| Plan | Duration | Price |
|---|---|---|
| MONTHLY | 1 month | ₹499 |
| QUARTERLY | 3 months | ₹1,347 (10% off) |
| YEARLY | 12 months | ₹4,491 (25% off) |

Daily cron downgrades expired PREMIUM → FREE.

### Marketplace Profile Creation Flow
```
Excel Import → auto-creates FREE profile (isMarketplaceActive: false)
             → Contractor pays PREMIUM → confirmPayment() → isMarketplaceActive: true
             → Contractor sets coverage areas + services → appears in consumer search
```

### Rating Rules (06 Mar 2026)
| Rater | Allowed Statuses |
|---|---|
| Consumer → Contractor | COMPLETED, REJECTED, CANCELLED |
| Contractor → Consumer | COMPLETED, CANCELLED |

---

## 8. Known Bugs & Anti-Patterns

### Critical
| # | Issue | File | Fix |
|---|---|---|---|
| 1 | Billing module is empty | `billing.service.ts`, `billing.resolver.ts` | Implement full billing export flow |
| 2 | Cashfree not integrated | `payments.service.ts` | Replace mocks with Cashfree API |
| 3 | Zero test coverage | Entire codebase | Only boilerplate spec files exist |

### High
| # | Issue | File | Fix |
|---|---|---|---|
| 4 | 32+ raw `throw new Error()` in resolvers | `jobs.resolver.ts`, `subscriptions.resolver.ts`, `imports.resolver.ts`, `auth.resolver.ts` | Replace with `NotFoundException`, `BadRequestException`, `ForbiddenException` |
| 5 | 11 notification TODOs | `jobs.service.ts` (7), `sla-scheduler.service.ts` (3), `sites.service.ts` (1) | Wire `NotificationService` to all call sites |
| 6 | No refund trigger on rejection/cancellation | `jobs.service.ts:349,391` | Connect `PaymentsService.initiateRefund()` |
| 7 | GCP storage throws runtime | `storage/index.ts:28` | Implement or remove GCP option |
| 8 | `me` query throws raw `Error` | `auth.resolver.ts:91` | Use `NotFoundException` |

### Medium
| # | Issue | Fix |
|---|---|---|
| 9 | Direct `db` injection in resolvers | `consumers.resolver.ts`, `meter-readings.resolver.ts` — use services instead |
| 10 | Placeholder meter UUID `000...000` | `installations.service.ts:114` — validate before insert |
| 11 | No global rate limiting | Add `@nestjs/throttler` |
| 12 | Missing `class-validator` on GraphQL inputs | Add decorators per user rules |
| 13 | Marketplace job OTP not SMS-delivered | `jobs.service.ts:446` — wire SMS |
| 14 | `support/` module contains import logic | Rename to `imports/` or `data-import/` |

---

## 9. Pending TODO List (All Files)

| File | Line | TODO |
|---|---|---|
| `billing.service.ts` | 5, 16 | Implement billing export (query VERIFIED sites → CSV → upload → return URL) |
| `billing.resolver.ts` | 5, 18 | Add `generateBillingExport`, `billingExports`, `billingSummary` |
| `storage/index.ts` | 28 | Implement GCP Cloud Storage provider |
| `sites.service.ts` | 1689 | Call `NotificationService.sendRegistrationSms()` |
| `jobs.service.ts` | 248, 304, 349, 350, 391, 392, 446, 565 | SMS to contractor/consumer on each job event + refund triggers |
| `sla-scheduler.service.ts` | 43, 44, 82, 121, 122 | Notify admin/consumer on SLA breach + auto-cancel + partial refund |
| `payments.service.ts` | (all) | Replace all mock Cashfree implementations with real API |

---

## 10. Database — Pending Migrations

| Migration | Status |
|---|---|
| `0014_sweet_silver_samurai.sql` | **Generated, NOT yet applied to DB** (adds `mp_contractor_coverage_selections` table + `mp_coverage_selection_type` enum) |

Run: `cd user-service && pnpm db:migrate`

---

## 11. Environment Variables (Key)

```env
# Runtime
NODE_ENV=development          # 'production' enables auth throttling
OTP_MODE=DUMMY                # DUMMY=123456 fixed; ACTUAL=real SMS
STORAGE_MODE=local            # local | azure | gcp (gcp not impl.)

# External services (disabled by default in dev)  
SMS_GATEWAY_ENABLED=false
EMAIL_ENABLED=false
EXCEL_IMPORT_MAX_SIZE_MB=3

# Pending (production)
CASHFREE_APP_ID=
CASHFREE_SECRET_KEY=
CASHFREE_WEBHOOK_SECRET=
```

---

## 12. Recent Session Changes (Chronological — Most Recent First)

| Date | Session | Key Changes |
|---|---|---|
| **06 Mar 2026** | Auth Throttle Env-Gating & Ratings Status Rules | `ThrottleService` disabled in non-prod; Ratings now allow REJECTED/CANCELLED jobs for consumers |
| **02 Mar 2026** | Marketplace UI Redesign & Subscription Versioning | `SubscriptionPlan` versioning (immutable history); Marketplace Hub redesigned with step-by-step setup flow; Pricing UI merged into service modal |
| **27 Feb 2026 (eve)** | Marketplace Flow Testing & Debugging | `searchMarketplaceContractors` root cause found (missing service mapping); full job lifecycle documented |
| **27 Feb 2026** | Contractor Coverage Areas, Search Fix & UI Fixes | Self-service coverage areas (DISTRICT/SUB_DISTRICT/VILLAGE); `mp_contractor_coverage_selections` table; hierarchical search rewrite; landing page logo fixes |
| **26 Feb 2026 (eve)** | Subscription Flow Fixes, Bank Account & Frontend Env Badge | Bank account fields on contractor profile; `contractorId` removed from client input (security fix); `EnvBadge.tsx` component |
| **26 Feb 2026** | LGD Location Imports & Upload Auth Fixes | Batch 1000-row UPSERT for 30K+ rows; State override in UI; file upload 401 fix (localStorage vs sessionStorage) |
| **23 Feb 2026** | Branding, Login, Forgot Password & Multi-Tenancy | KSLECA branding fix (12 instances); RO-only login default; 3-step forgot password wizard; Admin data isolation (createdBy-based); Bulk meter upload moved to /ro/inventory |
| **19 Feb 2026 (eve)** | Consumer Data Isolation & Subscriptions Module | `createdBy` FK on consumers; ADMIN sees only own consumers; Subscriptions module (6 new files); AWS SDK updated |
| **19 Feb 2026** | Backend Restructuring & Fixes | Associations module extracted from Marketplace; Location hierarchy (4 relational tables vs flat); SUPER_ADMIN marketplace restriction |
| **18 Feb 2026** | Admin Module Completion | Village category required; Location import moved to Marketplace Locations page |

---

## 13. Overall Readiness

| Domain | Score | Notes |
|---|---|---|
| Core Infrastructure (Auth, Users, RBAC) | 95% | Minor error handling issues |
| Smart Meter Domain (Sites → Verified) | 90% | State machines complete, audit logging working |
| Marketplace Platform | 70% | Coverage areas + search complete; payments mock; SMS unconnected |
| External Integrations | 40% | SMS/Email infrastructure built but under-connected; Cashfree not integrated |
| Testing | 5% | No meaningful tests |
| **Overall Production Readiness** | **~68%** | Marketplace payments, billing, notifications, and testing are the blockers |

---

## 14. Quick Reference — Test Credentials

```
Super Admin:  superadmin@mescom.gov / admin123  (loginType: ADMIN)
Admin:        admin@mescom.gov / admin123        (loginType: ADMIN)
RO:           ro@mescom.gov / retail123          (loginType: RO)
Contractor:   contractor@example.com / 123456    (OTP)
Consumer:     any registered phone / 123456      (OTP)
```

Backend: `http://localhost:3001` · GraphQL: `http://localhost:3001/graphql`  
Frontend: `http://localhost:3000`
