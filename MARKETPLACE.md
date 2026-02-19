SMARTMETER + MARKETPLACE FULL FRONTEND IMPLEMENTATION BLUEPRINT
=====================================================================
Generated On: 2026-02-17 06:13:39 UTC
=====================================================================

OVERVIEW

This document defines the complete frontend implementation strategy to
integrate the Contractor–Consumer Marketplace into the existing
SmartMeter Next.js 16 (App Router) system.

The goal is to MERGE the marketplace cleanly without breaking: •
Existing SmartMeter lifecycle UI • Role-based routing • Authentication
flows • Apollo Client setup • Tailwind design system

The Marketplace must behave as a separate domain inside the same app.

===================================================================== 1.
ARCHITECTURAL PRINCIPLES
=====================================================================

1.  Do NOT modify existing SmartMeter pages.
2.  Add Marketplace as a parallel feature set.
3.  Reuse layout, sidebar, auth guards.
4.  Never calculate price or SLA on frontend.
5.  All actions go through GraphQL mutations.
6.  After every mutation → refetch job state.
7.  Snapshot values always come from backend.

===================================================================== 2.
FOLDER STRUCTURE ADDITIONS
=====================================================================

/src/app/ (admin)/admin/marketplace/
(contractor)/contractor/marketplace/ (consumer)/consumer/marketplace/

/src/components/marketplace/ ServiceCard.tsx PriceDisplay.tsx
SLAIndicator.tsx JobTimeline.tsx OTPModal.tsx RatingStars.tsx
SubscriptionBadge.tsx SmartMeterInstallationProgress.tsx

/src/hooks/marketplace/ useServices.ts useCreateJob.ts
useContractorJobs.ts useJobDetails.ts usePayment.ts useSLAStatus.ts

/src/graphql/marketplace/ queries.ts mutations.ts fragments.ts

===================================================================== 3.
CONSUMER FRONTEND IMPLEMENTATION
=====================================================================

ROUTES:

/consumer/marketplace/services /consumer/marketplace/service/[id]
/consumer/marketplace/jobs /consumer/marketplace/job/[id]

A) Service Listing Page

• Fetch services via GraphQL • Pass consumer location_id • Backend
resolves area-based pricing • Display: - Service name - Price (from
backend) - SLA summary - Premium badge • Clicking → Service detail page

B) Service Detail Page

• Show description • Show resolved price • Show SLA timelines • Show
add-ons • “Book Service” button

C) Booking Flow

Step 1: Confirm address Step 2: Confirm price summary Step 3: Initiate
payment Step 4: Redirect to gateway Step 5: On success → create job
mutation Step 6: Redirect to job tracking page

Never trust frontend payment success.

D) Job Tracking Page

• Fetch job by id • Render: - Status badge - Timeline - SLA countdown
(visual only) - Contractor details - OTP modal trigger (if required) •
Poll every 10 seconds for active jobs

===================================================================== 4.
CONTRACTOR FRONTEND IMPLEMENTATION
=====================================================================

ROUTES:

/contractor/marketplace/jobs /contractor/marketplace/job/[id]
/contractor/marketplace/profile /contractor/marketplace/subscription

A) Job List Page

Filters: • Pending • Accepted • In Progress • Completed • SLA Breached

Each card shows: • Service • Location • Deadline • Countdown indicator

B) Job Detail Page

Actions based on state:

REQUESTED: • Accept • Reject

ACCEPTED: • Generate Start OTP • Start Job

STARTED: • Generate Completion OTP • Complete Job

All actions via mutation → refetch job.

C) Profile Page

• Rating average • SLA compliance • Subscription status • Serviceable
areas • QR code display

D) Subscription Page

• FREE vs PREMIUM comparison • Upgrade button • Payment flow

===================================================================== 5.
ADMIN FRONTEND IMPLEMENTATION
=====================================================================

ROUTES:

/admin/marketplace/services /admin/marketplace/pricing
/admin/marketplace/sla /admin/marketplace/locations
/admin/marketplace/subscriptions /admin/marketplace/analytics

A) Location Management

• Upload LGD Excel • Edit classification • Activate/deactivate entries

B) Service Management

• Create/edit services • Toggle premium • Activate/deactivate

C) Pricing Management

• View version history • Create new version • Schedule effective date

D) SLA Management

• Configure SLA per service + area • Version management • Active toggle

E) Analytics Dashboard

Charts: • SLA compliance % • Breach count • Revenue trends • Contractor
ranking

Use Recharts. Lazy load heavy charts.

===================================================================== 6.
SMARTMETER SERVICE INTEGRATION
=====================================================================

If service type = Energy Meter Installation:

• Detect job.isSmartMeterLinked • Render SmartMeterInstallationProgress
component • Sync with existing SmartMeter UI states

Mapping:

SmartMeter INSTALLED → Marketplace COMPLETED SmartMeter VERIFIED →
Marketplace CLOSED

Do not duplicate lifecycle UI logic.

===================================================================== 7.
GRAPHQL STRATEGY
=====================================================================

Queries: • getServices(locationId) • getJob(id) • getContractorJobs() •
getSLA(serviceId, areaType) • getPricing(serviceId, areaType)

Mutations: • createJob • acceptJob • rejectJob • startJob • completeJob
• generateOTP • verifyOTP • createPaymentIntent • submitRating •
upgradeSubscription

Always refetch after mutation.

===================================================================== 8.
UI & UX RULES
=====================================================================

• Disable buttons during loading. • Prevent double submission. • Use
consistent status colors. • Show SLA warning at 80% threshold. • Mask
phone numbers. • Use signed URLs for files. • Never expose internal IDs.

===================================================================== 9.
PERFORMANCE OPTIMIZATION
=====================================================================

• Use pagination for job lists. • Lazy load analytics. • Debounce search
inputs. • Avoid unnecessary refetches. • Use Apollo cache normalization.

=====================================================================
10. IMPLEMENTATION ORDER
=====================================================================

Phase 1: • Consumer service listing • Booking flow • Job tracking

Phase 2: • Contractor job handling • OTP flows

Phase 3: • Admin pricing & SLA management

Phase 4: • Analytics dashboard

Phase 5: • SmartMeter integration UI

=====================================================================
FINAL RESULT
=====================================================================

The frontend will now support:

• Government SmartMeter installation lifecycle • Electrical services
marketplace • SLA-driven job tracking • Subscription monetization •
Performance analytics

All within the same Next.js system, cleanly modularized without breaking
the existing SmartMeter domain.

=====================================================================
END OF FRONTEND IMPLEMENTATION BLUEPRINT
=====================================================================







SMARTMETER SYSTEM + MARKETPLACE INTEGRATION MASTER IMPLEMENTATION PROMPT
============================================================

Generated On: 2026-02-17 06:03:38 UTC

OBJECTIVE

Extend the existing SmartMeter Government Installation System by
introducing a new bounded domain called MARKETPLACE.

The Marketplace must support: • Consumer service booking • Contractor
selection • Location-based pricing • SLA-driven job execution •
OTP-based validation • Payment integration • Ratings & performance
tracking • Subscription-based contractor access

The implementation must NOT break the existing SmartMeter domain and
must maintain strict domain separation.

ARCHITECTURAL RULES (MANDATORY)

1.  SmartMeter domain remains untouched.
2.  Create a new /marketplace module group.
3.  Do NOT mix payment, SLA, or rating logic into SmartMeter modules.
4.  Snapshot SLA and pricing at booking time.
5.  All state transitions backend-enforced.
6.  SLA timers handled via background scheduler.
7.  SmartMeter integration must be event-based.

MODULE STRUCTURE

/src/modules/marketplace/ locations/ services/ pricing/ slas/
contractor-extension/ subscriptions/ jobs/ otp/ payments/ ratings/
scheduler/

Each submodule must contain: • Entity/schema • Service layer • GraphQL
resolver • Guard • Audit integration

DATABASE TABLES TO CREATE

location_master services service_pricing slas
contractor_location_mapping contractor_service_mapping
contractor_subscriptions jobs job_otps payments ratings sla_breaches

Modify existing: contractors (extend with marketplace fields)

Do NOT modify SmartMeter lifecycle tables.

LOCATION MASTER

Admin manages: • State • District • Taluk • LGD code • Area
classification (Urban/Semi-Urban/Rural)

Location drives: • Pricing resolution • SLA resolution • Contractor
eligibility

SERVICE CATALOGUE

Admin manages services. Each service: • Has UOM • May be premium-gated •
Has versioned pricing • Has versioned SLA mapping

Never store price directly in service table.

SERVICE PRICING (VERSIONED)

Fields: • service_id • area_type • base_price • effective_from •
effective_to • version

At job creation: • Resolve correct price • Store price_snapshot in job •
Never recalculate later

SLA ENGINE

Versioned SLA per Service + Area Type.

Fields: • service_id • area_type • acceptance_hours • job_start_hours •
completion_hours • version

At job creation: • Compute acceptance_deadline • Compute start_deadline
• Compute completion_deadline • Store inside job

Scheduler (every 5 mins): • Detect SLA nearing breach (80%) • Detect SLA
breach • Auto-expire or reassign • Record breach • Send notifications

CONTRACTOR EXTENSION

Extend contractors table with: • subscription_type •
subscription_valid_till • rating_avg • total_ratings • sla_breach_count
• qr_profile_token

Add mapping tables: • contractor_location_mapping •
contractor_service_mapping • contractor_subscriptions

Rules: FREE → only energy meter installation PREMIUM → all services

JOB STATE MACHINE

REQUESTED → ACCEPTED → STARTED (OTP required) → COMPLETED (OTP required)
→ CLOSED

Failure states: AUTO_EXPIRED REASSIGNED CANCELLED SLA_BREACH

Each transition must: • Validate role • Validate SLA • Validate OTP •
Write audit log

OTP ENFORCEMENT

OTP required for: • Job Start • Job Completion

Configurable: • Expiry duration • Retry count

Admin override must be audited.

PAYMENT INTEGRATION

Flow: 1. Consumer initiates payment 2. Redirect to payment gateway 3.
Gateway webhook confirms payment 4. Backend verifies signature 5. Mark
payment as SUCCESS 6. Allow job closure

Never trust frontend confirmation.

RATINGS

After job closure: • Consumer rates contractor • Contractor rates
consumer

Store: • rating • comment • timestamp • job_id

Update contractor rating metrics.

SMARTMETER INTEGRATION

If service = Energy Meter Installation:

1.  Create marketplace job
2.  Trigger SmartMeter installation flow
3.  Sync job status with SmartMeter lifecycle

Mapping: SmartMeter INSTALLED → Marketplace COMPLETED SmartMeter
VERIFIED → Marketplace CLOSED

Do NOT duplicate SmartMeter logic.

ROLE PERMISSIONS

Consumer: • Create job • View own jobs • Rate

Contractor: • Accept/reject • Start job • Complete job • Cannot modify
price

Admin: • Manage SLA • Manage pricing • Manage subscriptions • Override
job (audited)

All enforced in backend guards.

NON-NEGOTIABLE RULES

• Snapshot SLA & price at booking • Version everything • Never overwrite
history • Audit every state change • Backend authority only • Maintain
strict domain separation

=====================================================================
IMPLEMENTATION STATUS & COMPLETE FLOW DOCUMENTATION
=====================================================================

## Current Status (Updated: 2026-02-18)

### Backend (100% Complete)
- ✅ Database schema (migrations applied)
- ✅ All services implemented
- ✅ All resolvers implemented  
- ✅ SLA scheduler service
- ✅ Location bulk import mutation

### Frontend (100% Complete)
- ✅ Admin marketplace pages (services, pricing, SLA, locations)
- ✅ Consumer marketplace pages (browse, book, jobs)
- ✅ Contractor marketplace pages (jobs, profile)
- ✅ Location bulk upload UI
- ✅ Sidebar navigation added

### Pending
- ⏳ Cashfree payment integration
- ⏳ Refund handling
- ⏳ SMS notifications

=====================================================================
COMPLETE USER FLOWS
=====================================================================

## 1. ADMIN FLOW

### 1.1 Import Locations (LGD Data)
```
1. Navigate to /admin/upload
2. Select "Import Locations" tab
3. Upload Excel with columns:
   - stateCode, stateName (required)
   - districtCode, districtName (required)
   - subDistrictCode, subDistrictName (optional)
   - villageCode, villageName (required)
   - villageNameLocal, villageCategory (optional)
4. System imports/updates locations
5. Navigate to /admin/marketplace/locations to classify
```

### 1.2 Classify Locations
```
1. Navigate to /admin/marketplace/locations
2. Filter by state/district
3. Select unclassified locations
4. Assign area type: URBAN, SEMI_URBAN, or RURAL
5. Bulk classify or individual classify
```

### 1.3 Create Services
```
1. Navigate to /admin/marketplace/services
2. Click "Add Service"
3. Enter:
   - Name (e.g., "Meter Installation")
   - Description
   - Category: ENERGY_METER, WIRING_CABLING, DISTRIBUTION_BOARD, 
               EARTHING_SAFETY, INSPECTION_TESTING, SPECIAL_SERVICES
   - UOM: PER_JOB, PER_POINT, PER_METER, PER_UNIT, PER_VISIT
   - Is Premium Only (checkbox)
4. Save service
```

### 1.4 Configure Pricing
```
1. Navigate to /admin/marketplace/pricing
2. Select a service
3. Set prices for each area type:
   - Urban: ₹XXX
   - Semi-Urban: ₹XXX
   - Rural: ₹XXX
4. Prices are versioned (old prices preserved for audit)
```

### 1.5 Configure SLA
```
1. Navigate to /admin/marketplace/sla
2. Select a service
3. Set SLA hours for each area type:
   - Acceptance deadline (hours)
   - Job start deadline (hours)
   - Completion deadline (hours)
4. SLAs are versioned
```

## 2. CONSUMER FLOW

### 2.1 Browse Services
```
1. Navigate to /consumer/marketplace
2. View services grouped by category
3. See current prices based on your location's area type
```

### 2.2 Search Contractors
```
1. Select a service
2. System shows contractors:
   - Who offer the service
   - Are active in marketplace
   - Serve your area
3. View contractor profiles:
   - Rating, reviews
   - Jobs completed
   - Response time
```

### 2.3 Create Job (Book Service)
```
1. Select contractor and service
2. Confirm address and contact
3. System shows:
   - Price (based on your area type)
   - SLA terms
4. Click "Book Now"
5. System creates job with PAYMENT_PENDING status
6. [TODO] Redirect to Cashfree payment
7. On payment success → Job status = REQUESTED
8. Contractor is notified (SMS)
```

### 2.4 Track Job
```
1. Navigate to /consumer/marketplace/jobs
2. View all your jobs with status
3. Click job for details
4. Timeline shows:
   - Created → Paid → Requested → Accepted → Started → Completed → Rated
```

### 2.5 Cancel Job
```
1. From job details, click "Cancel"
2. Only allowed before job is STARTED
3. [TODO] Refund process initiated
```

### 2.6 OTP Verification (At Site)
```
1. Contractor arrives and generates OTP
2. Consumer receives OTP (will be via SMS)
3. Consumer shares OTP with contractor
4. Contractor enters OTP in their app
5. For START: Job status → STARTED
6. For COMPLETE: Job status → COMPLETED
```

### 2.7 Rate Contractor
```
1. After job completion
2. Rate 1-5 stars
3. Add optional comment
4. Submit rating
5. Job status → CLOSED
```

## 3. CONTRACTOR FLOW

### 3.1 Setup Marketplace Profile
```
1. Navigate to /contractor/marketplace/profile
2. Enter:
   - Service areas (select locations)
   - Services offered (select from list)
   - Bio
   - Certifications
3. Toggle marketplace active/inactive
```

### 3.2 View Job Requests
```
1. Navigate to /contractor/marketplace/jobs
2. Filter by status (Requested, Accepted, etc.)
3. New jobs show acceptance deadline
```

### 3.3 Accept/Reject Job
```
Accept:
1. Review job details (consumer, location, service, price)
2. Click "Accept"
3. Job status → ACCEPTED
4. Start deadline timer begins
5. Consumer notified

Reject:
1. Click "Reject" with reason
2. Job status → REJECTED
3. Contractor rejection count incremented
4. [TODO] Consumer refunded
```

### 3.4 Start Job (OTP Flow)
```
1. Arrive at consumer's location
2. From job details, click "Generate Start OTP"
3. OTP is generated (6 digits, 15 min validity)
4. Consumer receives OTP [via SMS]
5. Consumer tells contractor the OTP
6. Contractor enters OTP and clicks "Verify"
7. If valid: Job status → STARTED
8. Completion deadline timer begins
```

### 3.5 Complete Job (OTP Flow)
```
1. After work is done
2. Click "Generate Completion OTP"
3. Same OTP verification flow
4. If valid: Job status → COMPLETED
5. Contractor's completed jobs count incremented
```

### 3.6 View Ratings & Stats
```
1. From profile, view:
   - Average rating
   - Total jobs completed
   - SLA breach count
   - Rejection count
2. View individual reviews from consumers
```

## 4. SLA BREACH HANDLING

```
Scheduler runs every 5 minutes checking:

1. Acceptance Breach:
   - Job in REQUESTED status
   - Acceptance deadline passed
   - Records breach
   - [TODO] Auto-reject and refund

2. Start Breach:
   - Job in ACCEPTED status  
   - Start deadline passed
   - Records breach
   - Notifies admin

3. Completion Breach:
   - Job in STARTED status
   - Completion deadline passed
   - Records breach
   - [TODO] Partial refund consideration
```

=====================================================================
SERVICE CATEGORIES
=====================================================================

| Category | Description |
|----------|-------------|
| ENERGY_METER | Meter installation, replacement, testing |
| WIRING_CABLING | Electrical wiring, cable laying |
| DISTRIBUTION_BOARD | DB installation, repair |
| EARTHING_SAFETY | Earthing, safety devices |
| INSPECTION_TESTING | Electrical audits, testing |
| SPECIAL_SERVICES | Premium/custom services |

=====================================================================
UNIT OF MEASUREMENT (UOM)
=====================================================================

| UOM | Description |
|-----|-------------|
| PER_JOB | Fixed price per job |
| PER_POINT | Price per electrical point |
| PER_METER | Price per meter of cable |
| PER_UNIT | Price per unit installed |
| PER_VISIT | Price per visit/consultation |

=====================================================================
JOB STATUS STATE MACHINE
=====================================================================

```
PAYMENT_PENDING → (payment success) → REQUESTED
                                          ↓
                   ← (reject) ← REJECTED   
                                          ↓
                              (accept) → ACCEPTED
                                          ↓
                  ← (cancel by consumer) ← 
                                          ↓
                           (start OTP) → STARTED
                                          ↓
                        (complete OTP) → COMPLETED
                                          ↓
                             (rating) → CLOSED

From any state:
- CANCELLED (by consumer before STARTED)
- SLA_BREACH (by scheduler)
- REFUNDED (after refund processed)
```

=====================================================================
GRAPHQL OPERATIONS REFERENCE
=====================================================================

### Queries
- `marketplaceServices` - List services with filters
- `marketplaceService` - Get single service
- `marketplaceServicesWithPrices` - Services with prices for area
- `marketplaceLocations` - List locations with filters
- `marketplaceStates` - Dropdown states
- `marketplaceDistricts` - Dropdown districts
- `marketplaceSubDistricts` - Dropdown sub-districts
- `marketplaceVillages` - Dropdown villages (classified only)
- `marketplacePricing` - Get pricing for service
- `marketplaceSla` - Get SLA for service
- `searchMarketplaceContractors` - Search contractors
- `marketplaceContractorProfile` - Get contractor profile
- `marketplaceJobs` - List jobs with filters
- `marketplaceJob` - Get job details

### Mutations
- `createMarketplaceService` - Create service (Admin)
- `updateMarketplaceService` - Update service (Admin)
- `classifyLocation` - Classify single location (Admin)
- `bulkClassifyLocations` - Bulk classify (Admin)
- `importLocations` - Import from Excel (Admin)
- `setMarketplacePricing` - Set/update pricing (Admin)
- `setMarketplaceSla` - Set/update SLA (Admin)
- `updateMarketplaceProfile` - Update contractor profile
- `createMarketplaceJob` - Create job (Consumer)
- `simulateMarketplacePaymentSuccess` - [TEMP] Simulate payment
- `acceptMarketplaceJob` - Accept job (Contractor)
- `rejectMarketplaceJob` - Reject job (Contractor)
- `cancelMarketplaceJob` - Cancel job (Consumer)
- `generateMarketplaceOtp` - Generate OTP (Contractor)
- `verifyMarketplaceOtp` - Verify OTP (Contractor)
- `rateMarketplaceJob` - Rate job (Consumer/Contractor)

END OF MASTER IMPLEMENTATION PROMPT
