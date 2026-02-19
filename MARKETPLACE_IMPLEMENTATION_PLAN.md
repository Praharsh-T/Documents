# MARKETPLACE INTEGRATION - COMPLETE IMPLEMENTATION PLAN
**Generated:** 17 February 2026  
**Last Updated:** 19 February 2026  
**Status:** In Progress (Subscriptions Module ✅ COMPLETED)  
**Target System:** SmartMeter v1.0 + Marketplace Extension

---

## TABLE OF CONTENTS
1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Database Schema Design](#database-schema-design)
4. [Backend Implementation](#backend-implementation)
5. [Frontend Implementation](#frontend-implementation)
6. [Integration Points](#integration-points)
7. [Implementation Phases](#implementation-phases)
8. [Files to Update](#files-to-update)
9. [New Files to Create](#new-files-to-create)
10. [Testing Strategy](#testing-strategy)
11. [Deployment Checklist](#deployment-checklist)

---

## 1. EXECUTIVE SUMMARY

### Goal
Extend the SmartMeter Government Installation System with a **Marketplace** that enables:
- Consumers to book electrical services beyond smart meter installation
- Contractors to offer services with location-based pricing
- SLA-driven job execution with OTP validation
- Payment integration and performance tracking
- Subscription-based contractor monetization

### Core Principle
**NO BREAKING CHANGES** to the existing SmartMeter domain. The Marketplace is a **parallel bounded context** that optionally integrates with SmartMeter when service type is "Energy Meter Installation".

### Key Features
1. **Location-Based Pricing** - Automatic price resolution based on consumer location
2. **Versioned SLA Management** - Service-level agreements with automatic deadline tracking
3. **Job State Machine** - Controlled lifecycle with OTP validation
4. **Payment Integration** - Gateway integration with webhook verification
5. **Rating System** - Dual rating (consumer ↔ contractor)
6. **SmartMeter Integration** - Seamless linking when service is meter installation

---

## 2. ARCHITECTURE OVERVIEW

### Domain Separation

```
┌─────────────────────────────────────────────────────────┐
│                    SmartMeter System                     │
│  (Government Mandate - Meter Installation Lifecycle)    │
│                                                           │
│  Modules: consumers, sites, meters, installations,       │
│           verifications, billing                         │
└─────────────────────────────────────────────────────────┘
                           │
                           │ (Event-based Integration)
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   Marketplace System                     │
│   (Service Marketplace - Beyond Meter Installation)     │
│                                                           │
│  Modules: locations, services, pricing, slas, jobs,      │
│           payments, ratings, subscriptions               │
└─────────────────────────────────────────────────────────┘
```

### Shared Resources
- **Authentication & Authorization** - Reuse existing JWT + RBAC
- **Audit Logs** - Marketplace actions logged to existing audit system
- **File Storage** - Use existing files module
- **Notification** - Use existing notification service
- **Contractors** - Extend existing contractors table

### New Bounded Context
- **Marketplace** - Completely new module group at `/modules/marketplace/`

---

## 3. DATABASE SCHEMA DESIGN

### 3.1 New Tables

#### **location_master**
Manages villages from LGD (Local Government Directory) with admin-assigned area classification.

```typescript
{
  id: uuid (PK),
  
  // LGD Hierarchy (from Excel import)
  stateCode: varchar(10),
  stateVersion: varchar(10),
  stateName: varchar(255),
  districtCode: varchar(10),
  districtName: varchar(255),
  subDistrictCode: varchar(10),      // Taluk/Block
  subDistrictName: varchar(255),
  villageCode: varchar(20) UNIQUE,   // Primary LGD identifier
  villageVersion: varchar(10),
  villageName: varchar(255),
  villageNameLocal: varchar(255),    // Local language name
  villageCategory: varchar(50),      // From LGD
  villageStatus: varchar(50),        // From LGD
  
  // Admin Classification
  areaType: enum('URBAN', 'SEMI_URBAN', 'RURAL') (nullable),
  isClassified: boolean DEFAULT false,
  classifiedBy: uuid (FK -> users.id, nullable),
  classifiedAt: timestamp (nullable),
  
  // System fields
  isActive: boolean DEFAULT true,
  createdAt: timestamp,
  updatedAt: timestamp
}

Indexes:
- villageCode (unique) - Primary LGD identifier
- stateName + districtName + villageName (composite)
- areaType
- stateCode, districtCode, subDistrictCode (composite)
- isClassified
```

**Purpose:** Drive pricing and SLA resolution. Admin uploads LGD Excel, then manually classifies villages as Urban/Semi-Urban/Rural.

**Admin Workflow:**
1. Admin downloads LGD Excel from government portal
2. Admin uploads to system (all villages imported with `isClassified = false`)
3. UI shows unclassified villages in a list (filterable by state/district)
4. Admin selects village(s) and assigns area type (URBAN/SEMI_URBAN/RURAL)
5. System marks `isClassified = true` and records who/when
6. Only classified locations are used for pricing/SLA resolution

**Consumer Site Linking:**
- Consumer selects their village from dropdown (searchable by name)
- System stores `villageCode` reference in consumer's site
- Price/SLA resolved using village's `areaType`

---

#### **services**
Service catalog maintained by admin.

```typescript
{
  id: uuid (PK),
  name: varchar(255),
  description: text,
  category: enum('ELECTRICAL', 'PLUMBING', 'CARPENTRY', 'METER_INSTALLATION'),
  uom: enum('PER_SERVICE', 'PER_HOUR', 'PER_UNIT'),
  isPremiumOnly: boolean, // FREE contractors cannot see this
  isActive: boolean,
  createdAt: timestamp,
  updatedAt: timestamp
}

Indexes:
- category
- isPremiumOnly
```

**Rules:**
- Price is NOT stored here (stored in versioned pricing table)
- SLA is NOT stored here (stored in versioned SLA table)
- `category = METER_INSTALLATION` → triggers SmartMeter integration

---

#### **service_pricing** (Versioned)
Pricing changes over time without losing history.

```typescript
{
  id: uuid (PK),
  serviceId: uuid (FK -> services.id),
  areaType: enum('URBAN', 'SEMI_URBAN', 'RURAL'),
  basePrice: decimal(10,2),
  effectiveFrom: timestamp,
  effectiveTo: timestamp (nullable),
  version: integer,
  createdBy: uuid (FK -> users.id),
  createdAt: timestamp
}

Indexes:
- serviceId + areaType + effectiveFrom (composite)
- version
```

**Resolution Logic:**
```sql
SELECT * FROM service_pricing 
WHERE serviceId = ? 
  AND areaType = ? 
  AND effectiveFrom <= NOW() 
  AND (effectiveTo IS NULL OR effectiveTo > NOW())
ORDER BY effectiveFrom DESC 
LIMIT 1;
```

**Job Creation Rule:**
- Snapshot the resolved `basePrice` into the `jobs.priceSnapshot` field
- NEVER recalculate price after job creation

---

#### **slas** (Versioned)
Service-level agreements per service + area type.

```typescript
{
  id: uuid (PK),
  serviceId: uuid (FK -> services.id),
  areaType: enum('URBAN', 'SEMI_URBAN', 'RURAL'),
  acceptanceHours: integer, // Max hours for contractor to accept
  jobStartHours: integer,   // Max hours to start after acceptance
  completionHours: integer, // Max hours to complete after start
  effectiveFrom: timestamp,
  effectiveTo: timestamp (nullable),
  version: integer,
  createdBy: uuid (FK -> users.id),
  createdAt: timestamp
}

Indexes:
- serviceId + areaType + effectiveFrom (composite)
```

**Job Creation Rule:**
- Calculate and snapshot deadlines:
  - `acceptanceDeadline = createdAt + acceptanceHours`
  - `startDeadline = acceptedAt + jobStartHours`
  - `completionDeadline = startedAt + completionHours`
- Store in `jobs` table as immutable fields

---

#### **contractor_location_mapping**
Which locations can a contractor service?

```typescript
{
  id: uuid (PK),
  contractorId: uuid (FK -> contractors.id),
  locationId: uuid (FK -> location_master.id),
  createdAt: timestamp
}

Indexes:
- contractorId + locationId (unique composite)
- locationId
```

---

#### **contractor_service_mapping**
Which services can a contractor offer?

```typescript
{
  id: uuid (PK),
  contractorId: uuid (FK -> contractors.id),
  serviceId: uuid (FK -> services.id),
  createdAt: timestamp
}

Indexes:
- contractorId + serviceId (unique composite)
- serviceId
```

**Rule:**
- If service is `isPremiumOnly = true` and contractor subscription is `FREE` → BLOCK

---

#### **contractor_subscriptions** ✅ IMPLEMENTED
Tracks contractor subscription history.

**Table:** `mp_contractor_subscriptions`  
**File:** `src/database/schema/marketplace/contractor-extensions.ts`

```typescript
{
  id: uuid (PK),
  contractorId: uuid (FK -> contractors.id),
  subscriptionType: enum('FREE', 'PREMIUM'),
  startDate: timestamp,
  endDate: timestamp (nullable),
  paymentId: uuid (nullable),   // Payment intent ID reference
  isActive: boolean,
  createdAt: timestamp
}

Indexes:
- contractorId + isActive
- subscriptionType
```

**Rules:**
- `FREE` subscription = limited service categories (basic electrical works)
- `PREMIUM` subscription = all marketplace services
- Backend validates subscription before contractor appears in search results
- **Note:** Meter Installation is NOT in marketplace (handled by SmartMeter module)

**Available Plans (Implemented):**
| Plan | Duration | Base Price | Discount | Final Price |
|------|----------|-----------|----------|-------------|
| MONTHLY | 1 month | ₹499 | 0% | ₹499 |
| QUARTERLY | 3 months | ₹1,497 | 10% | ₹1,347 |
| YEARLY | 12 months | ₹5,988 | 25% | ₹4,491 |

---

#### **jobs**
Core job entity with snapshot fields.

**NEW FLOW: Payment First Model**
```
Consumer searches services → Sees contractor list → Selects contractor → Pays → 
Job created with status PAYMENT_PENDING → Payment verified → status REQUESTED →
Contractor accepts/rejects → If rejected, consumer gets refund
```

```typescript
{
  id: uuid (PK),
  jobNumber: varchar(50) UNIQUE, // Auto-generated JOB-YYYYMMDD-XXXX
  consumerId: uuid (FK -> consumers.id),
  siteId: uuid (FK -> consumer_sites.id),
  locationId: uuid (FK -> location_master.id), // Consumer's village
  serviceId: uuid (FK -> services.id),
  contractorId: uuid (FK -> contractors.id), // Consumer selects contractor
  
  // Snapshot fields (immutable after creation)
  priceSnapshot: decimal(10,2),
  areaTypeSnapshot: enum('URBAN', 'SEMI_URBAN', 'RURAL'),
  acceptanceDeadline: timestamp,
  startDeadline: timestamp (nullable), // Set after acceptance
  completionDeadline: timestamp (nullable), // Set after job start
  
  // State machine (UPDATED for payment-first flow)
  status: enum(
    'PAYMENT_PENDING',  // Job created, awaiting payment
    'REQUESTED',        // Payment done, waiting contractor response
    'ACCEPTED',         // Contractor accepted
    'REJECTED',         // Contractor rejected → triggers refund
    'STARTED',          // Contractor started (OTP verified)
    'COMPLETED',        // Contractor completed (OTP verified)
    'CLOSED',           // Rated and finalized
    'CANCELLED',        // Consumer cancelled before acceptance
    'REFUNDED',         // Payment refunded (after rejection/cancellation)
    'SLA_BREACH'        // SLA breached
  ),
  
  // Timestamps
  createdAt: timestamp,
  paidAt: timestamp (nullable),        // When payment succeeded
  acceptedAt: timestamp (nullable),
  rejectedAt: timestamp (nullable),
  startedAt: timestamp (nullable),
  completedAt: timestamp (nullable),
  closedAt: timestamp (nullable),
  
  // Rejection tracking
  rejectionReason: text (nullable),
  
  // Payment
  paymentId: uuid (FK -> payments.id, nullable),
  refundId: uuid (FK -> refunds.id, nullable),
  
  updatedAt: timestamp
}

Indexes:
- jobNumber (unique)
- consumerId
- contractorId
- status
- createdAt
- locationId
```

**State Transition Rules (Payment-First Model):**
```
PAYMENT_PENDING → REQUESTED (payment webhook confirms success)
PAYMENT_PENDING → CANCELLED (payment failed or consumer cancels)

REQUESTED → ACCEPTED (contractor accepts within SLA)
REQUESTED → REJECTED (contractor rejects → auto-refund triggered)
REQUESTED → CANCELLED (consumer cancels → auto-refund triggered)

ACCEPTED → STARTED (OTP verified)
STARTED → COMPLETED (OTP verified)
COMPLETED → CLOSED (consumer rates contractor)

REJECTED → REFUNDED (refund processed)
CANCELLED → REFUNDED (refund processed)
```

---

#### **job_otps**
OTP management for job actions.

```typescript
{
  id: uuid (PK),
  jobId: uuid (FK -> jobs.id),
  otpType: enum('START_JOB', 'COMPLETE_JOB'),
  code: varchar(6),
  generatedBy: uuid (FK -> users.id), // Contractor who generated
  isUsed: boolean,
  usedAt: timestamp (nullable),
  expiresAt: timestamp,
  createdAt: timestamp
}

Indexes:
- jobId + otpType
- code
- expiresAt
```

**Flow:**
1. Contractor clicks "Start Job" → Backend generates OTP → SMS to consumer
2. Consumer gives OTP to contractor
3. Contractor enters OTP → Backend verifies → Job state changes to `STARTED`

---

#### **payments**
Payment tracking with Cashfree webhook integration.

```typescript
{
  id: uuid (PK),
  jobId: uuid (FK -> jobs.id),
  userId: uuid (FK -> users.id), // Consumer who paid
  amount: decimal(10,2),
  currency: varchar(3) DEFAULT 'INR',
  gatewayProvider: varchar(50) DEFAULT 'CASHFREE',
  
  // Cashfree specific fields
  cfOrderId: varchar(255),          // Cashfree order ID
  cfPaymentId: varchar(255) (nullable),
  cfPaymentSessionId: varchar(255) (nullable),
  cfPaymentMethod: varchar(50) (nullable), // UPI, CARD, NB, etc.
  cfSignature: text (nullable),
  
  status: enum('PENDING', 'SUCCESS', 'FAILED', 'REFUNDED'),
  
  createdAt: timestamp,
  completedAt: timestamp (nullable),
  failureReason: text (nullable)
}

Indexes:
- jobId (unique) - one payment per job
- cfOrderId (unique)
- userId
- status
```

---

#### **refunds**
Refund tracking when contractor rejects or consumer cancels.

```typescript
{
  id: uuid (PK),
  paymentId: uuid (FK -> payments.id),
  jobId: uuid (FK -> jobs.id),
  amount: decimal(10,2),
  reason: enum('CONTRACTOR_REJECTED', 'CONSUMER_CANCELLED', 'ADMIN_CANCELLED', 'OTHER'),
  reasonNotes: text (nullable),
  
  // Cashfree refund fields
  cfRefundId: varchar(255) (nullable),
  cfRefundStatus: varchar(50) (nullable),
  cfRefundArn: varchar(255) (nullable), // Bank reference number
  
  status: enum('PENDING', 'PROCESSING', 'SUCCESS', 'FAILED'),
  initiatedBy: uuid (FK -> users.id),
  
  createdAt: timestamp,
  processedAt: timestamp (nullable),
  failureReason: text (nullable)
}

Indexes:
- paymentId
- jobId
- status
- cfRefundId
```

**Refund Flow:**
```
1. Contractor rejects job OR Consumer cancels
2. System creates refund record with status PENDING
3. Backend calls Cashfree Refund API
4. Cashfree webhook confirms refund → status SUCCESS
5. Job status → REFUNDED
6. SMS sent to consumer: "Refund of ₹XXX initiated"
```
- userId
- gatewayOrderId (unique)
- status
```

**Security:**
- NEVER trust frontend payment confirmation
- Always verify via webhook signature
- Mark payment as SUCCESS only after backend verification

---

#### **ratings**
Dual rating system (consumer ↔ contractor).

```typescript
{
  id: uuid (PK),
  jobId: uuid (FK -> jobs.id),
  raterUserId: uuid (FK -> users.id), // Who gave the rating
  rateeUserId: uuid (FK -> users.id), // Who received the rating
  rating: integer CHECK (rating >= 1 AND rating <= 5),
  comment: text (nullable),
  createdAt: timestamp
}

Indexes:
- jobId + raterUserId (unique composite)
- rateeUserId
```

**Rules:**
- Consumer rates contractor after job completion
- Contractor rates consumer after job completion
- Rating triggers recalculation of contractor's `rating_avg`

---

#### **sla_breaches**
Audit trail for SLA violations.

```typescript
{
  id: uuid (PK),
  jobId: uuid (FK -> jobs.id),
  breachType: enum('ACCEPTANCE', 'START', 'COMPLETION'),
  deadline: timestamp,
  breachedAt: timestamp,
  createdAt: timestamp
}

Indexes:
- jobId
- breachType
- breachedAt
```

---

### 3.2 Modifications to Existing Tables

#### **contractors** (Extend)
Add marketplace-specific fields:

```typescript
// NEW FIELDS TO ADD:
subscriptionType: enum('FREE', 'PREMIUM') DEFAULT 'FREE',
subscriptionValidTill: timestamp (nullable),
ratingAvg: decimal(3,2) DEFAULT 0.00,
totalRatings: integer DEFAULT 0,
slaBreachCount: integer DEFAULT 0,
qrProfileToken: varchar(255) UNIQUE (nullable), // For consumer QR scanning
```

**Migration Strategy:**
- Run Drizzle migration to add columns
- Default all existing contractors to `FREE` subscription
- Generate `qrProfileToken` for existing contractors

---

## 4. BACKEND IMPLEMENTATION

### 4.1 Module Structure

Create new module group in userservice: `/userservice/src/modules/marketplace/`

**Note:** Backend folder is now `userservice` (formerly `smartmeter-backend`)
Frontend folder is now `web-frontend` (formerly `smartmeter-frontend`)

```
/marketplace/
├── marketplace.module.ts        # Root marketplace module
├── locations/
│   ├── locations.module.ts
│   ├── locations.service.ts
│   ├── locations.resolver.ts
│   ├── locations.types.ts       # GraphQL types
│   └── index.ts
├── services/
│   ├── services.module.ts
│   ├── services.service.ts
│   ├── services.resolver.ts
│   ├── services.types.ts
│   └── index.ts
├── pricing/
│   ├── pricing.module.ts
│   ├── pricing.service.ts
│   ├── pricing.resolver.ts
│   ├── pricing.types.ts
│   └── index.ts
├── slas/
│   ├── slas.module.ts
│   ├── slas.service.ts
│   ├── slas.resolver.ts
│   ├── slas.types.ts
│   └── index.ts
├── contractor-extension/
│   ├── contractor-extension.module.ts
│   ├── contractor-extension.service.ts
│   ├── contractor-extension.resolver.ts
│   ├── contractor-extension.types.ts
│   └── index.ts
├── subscriptions/
│   ├── subscriptions.module.ts
│   ├── subscriptions.service.ts
│   ├── subscriptions.resolver.ts
│   ├── subscriptions.types.ts
│   └── index.ts
├── jobs/
│   ├── jobs.module.ts
│   ├── jobs.service.ts
│   ├── jobs.resolver.ts
│   ├── jobs.types.ts
│   ├── jobs.state-machine.ts    # State transition logic
│   └── index.ts
├── otp/
│   ├── otp.module.ts
│   ├── otp.service.ts
│   ├── otp.resolver.ts
│   ├── otp.types.ts
│   └── index.ts
├── payments/
│   ├── payments.module.ts
│   ├── payments.service.ts
│   ├── payments.resolver.ts
│   ├── payments.controller.ts   # REST endpoint for webhook
│   ├── payments.types.ts
│   └── index.ts
├── ratings/
│   ├── ratings.module.ts
│   ├── ratings.service.ts
│   ├── ratings.resolver.ts
│   ├── ratings.types.ts
│   └── index.ts
├── scheduler/
│   ├── marketplace-scheduler.module.ts
│   ├── marketplace-scheduler.service.ts
│   └── index.ts
└── index.ts                     # Barrel export
```

---

### 4.2 Key Services

#### **LocationsService**
```typescript
class LocationsService {
  // Admin uploads Excel with village list (name, district, state, pincode)
  async bulkImportLocations(fileKey: string, userId: string): Promise<ImportResult>
  
  // Get unclassified locations (for admin to classify)
  async getUnclassifiedLocations(filters?: LocationFilters): Promise<Location[]>
  
  // Get classified locations
  async getClassifiedLocations(filters?: LocationFilters): Promise<Location[]>
  
  // Admin manually classifies a location
  async classifyLocation(id: string, areaType: AreaType, adminId: string): Promise<Location>
  
  // Bulk classify multiple locations at once
  async bulkClassifyLocations(ids: string[], areaType: AreaType, adminId: string): Promise<Location[]>
  
  // Resolve area type for a consumer's location (by pincode or location id)
  async resolveAreaType(locationId: string): Promise<AreaType>
  
  // Get all active locations
  async findAll(filters?: LocationFilters): Promise<Location[]>
  
  // Search locations by name/pincode (for consumer site linking)
  async searchLocations(query: string): Promise<Location[]>
}
```

---

#### **ServicesService**
```typescript
class ServicesService {
  // Create/update services (admin only)
  async createService(input: CreateServiceInput): Promise<Service>
  async updateService(id: string, input: UpdateServiceInput): Promise<Service>
  
  // Get services available to a consumer (based on location + contractor availability)
  async getAvailableServices(consumerId: string): Promise<ServiceWithPrice[]>
  
  // Get services available to a contractor (based on subscription)
  async getServicesForContractor(contractorId: string): Promise<Service[]>
}
```

---

#### **PricingService**
```typescript
class PricingService {
  // Create new pricing version
  async createPricingVersion(input: CreatePricingInput): Promise<ServicePricing>
  
  // Resolve current price for service + area type
  async resolvePrice(serviceId: string, areaType: AreaType): Promise<Decimal>
  
  // Get pricing history
  async getPricingHistory(serviceId: string): Promise<ServicePricing[]>
  
  // Schedule future pricing (effectiveFrom)
  async schedulePricing(input: SchedulePricingInput): Promise<ServicePricing>
}
```

---

#### **SLAsService**
```typescript
class SLAsService {
  // Create new SLA version
  async createSLAVersion(input: CreateSLAInput): Promise<SLA>
  
  // Resolve current SLA for service + area type
  async resolveSLA(serviceId: string, areaType: AreaType): Promise<SLA>
  
  // Get SLA history
  async getSLAHistory(serviceId: string): Promise<SLA[]>
  
  // Calculate deadlines
  async calculateDeadlines(sla: SLA, baseTime: Date): Promise<Deadlines>
}
```

---

#### **JobsService**
```typescript
class JobsService {
  // Consumer creates job
  async createJob(input: CreateJobInput, userId: string): Promise<Job>
  
  // Contractor accepts job
  async acceptJob(jobId: string, contractorId: string): Promise<Job>
  
  // Contractor rejects job
  async rejectJob(jobId: string, contractorId: string, reason: string): Promise<Job>
  
  // Start job (requires OTP)
  async startJob(jobId: string, otp: string, contractorId: string): Promise<Job>
  
  // Complete job (requires OTP)
  async completeJob(jobId: string, otp: string, contractorId: string): Promise<Job>
  
  // Close job (after payment + rating)
  async closeJob(jobId: string): Promise<Job>
  
  // Get job details
  async findById(jobId: string): Promise<Job>
  
  // Get consumer's jobs
  async getConsumerJobs(consumerId: string, filters?: JobFilters): Promise<Job[]>
  
  // Get contractor's jobs
  async getContractorJobs(contractorId: string, filters?: JobFilters): Promise<Job[]>
  
  // Admin reassign job
  async reassignJob(jobId: string, newContractorId: string, adminId: string): Promise<Job>
}
```

**Critical Logic:**
1. **Price Snapshot:** At job creation, resolve and snapshot price → never recalculate
2. **SLA Snapshot:** At job creation, resolve SLA and calculate all deadlines → store as immutable
3. **State Validation:** Every state transition must validate current state
4. **OTP Validation:** Start/Complete actions must verify OTP
5. **SmartMeter Integration:** If service is meter installation, create linked installation record

---

#### **OTPService**
```typescript
class OTPService {
  // Generate OTP for job action
  async generateOTP(jobId: string, type: OTPType, contractorId: string): Promise<string>
  
  // Verify OTP
  async verifyOTP(jobId: string, type: OTPType, code: string): Promise<boolean>
  
  // Send OTP via SMS
  async sendOTP(phone: string, code: string, jobNumber: string): Promise<void>
}
```

**Rules:**
- OTP valid for 10 minutes (configurable)
- Max 3 OTP generation attempts per job action
- SMS sent to consumer's phone number

---

#### **PaymentsService**
```typescript
class PaymentsService {
  // Initiate payment (before job closure)
  async createPaymentIntent(jobId: string, userId: string): Promise<PaymentIntent>
  
  // Verify payment via webhook
  async verifyPayment(gatewayOrderId: string, signature: string): Promise<Payment>
  
  // Handle payment success
  async handlePaymentSuccess(paymentId: string): Promise<void>
  
  // Handle payment failure
  async handlePaymentFailure(paymentId: string): Promise<void>
  
  // Refund payment
  async refundPayment(paymentId: string, adminId: string): Promise<Payment>
}
```

**Security:**
- Never trust frontend payment confirmation
- Verify webhook signature using gateway's secret key
- Log all payment events to audit

---

#### **RatingsService**
```typescript
class RatingsService {
  // Submit rating
  async submitRating(input: CreateRatingInput, raterUserId: string): Promise<Rating>
  
  // Get ratings for a user
  async getRatingsForUser(userId: string): Promise<Rating[]>
  
  // Recalculate contractor's average rating
  async recalculateContractorRating(contractorId: string): Promise<void>
}
```

---

#### **SubscriptionsService** ✅ IMPLEMENTED
**File:** `src/modules/marketplace/subscriptions/subscriptions.service.ts`

```typescript
class SubscriptionsService {
  // Get available subscription plans (MONTHLY / QUARTERLY / YEARLY)
  getSubscriptionPlans(): SubscriptionPlan[]
  
  // Get current subscription status for a contractor
  async getSubscriptionStatus(contractorId: string): Promise<SubscriptionStatus>
  // Returns: { currentType, validTill, isActive, daysRemaining, canUpgrade, canRenew }
  
  // Get subscription history for a contractor
  async getSubscriptionHistory(contractorId: string): Promise<ContractorSubscription[]>
  
  // Initiate upgrade - creates payment intent (mock Cashfree link)
  async initiateUpgrade(input: InitiateSubscriptionUpgradeInput): Promise<SubscriptionUpgradeResponse>
  // Blocks if already PREMIUM with >30 days remaining
  // Returns paymentLink + orderId + expiresAt (30 min window)
  
  // Confirm payment - upgrades subscription in DB
  async confirmPayment(input: ConfirmSubscriptionPaymentInput): Promise<SubscriptionConfirmResponse>
  // Extends from existing endDate if renewing
  // Deactivates old subscription record, creates new one
  
  // Admin directly set subscription (bypass payment)
  async adminSetSubscription(input: AdminSetSubscriptionInput): Promise<SubscriptionConfirmResponse>
  
  // Called by scheduler - downgrades expired PREMIUM to FREE
  async checkExpiredSubscriptions(): Promise<number>
  
  // Get payment intent status by ID
  getPaymentIntent(id: string): SubscriptionPaymentIntent | null
}
```

**Renewal Logic:**
- If contractor already has PREMIUM with >30 days left → blocked (use renew instead)
- If renewing, new endDate extends from existing `subscriptionValidTill` (not from now)
- On expiry, scheduler auto-downgrades to FREE and creates a new FREE subscription record

**Payment Intent Lifecycle:**
```
PENDING → SUCCESS (on confirmPayment)
PENDING → EXPIRED (after 30 minutes)
```
> ⚠️ **Note:** Payment intents are currently stored in-memory (Map). Replace with DB table or Redis before production.

---

#### **SubscriptionsScheduler** ✅ IMPLEMENTED
**File:** `src/modules/marketplace/subscriptions/subscriptions.scheduler.ts`

```typescript
class SubscriptionsScheduler implements OnModuleInit {
  // Runs daily at midnight - downgrades expired PREMIUM subscriptions
  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  async handleExpiredSubscriptions(): Promise<void>
  
  // Also runs once 10 seconds after server startup
  async onModuleInit(): Promise<void>
}
```

---

#### **MarketplaceSchedulerService**
Runs every 5 minutes via NestJS Scheduler.

```typescript
class MarketplaceSchedulerService {
  @Cron('*/5 * * * *') // Every 5 minutes
  async checkSLABreaches(): Promise<void>
  
  @Cron('0 * * * *') // Every hour
  async checkExpiredSubscriptions(): Promise<void>
}
```

**SLA Breach Logic:**
1. Query jobs with `status IN ('REQUESTED', 'ACCEPTED', 'STARTED')`
2. Check if any deadline is past
3. If breach detected:
   - Create `sla_breaches` record
   - Update job status to `SLA_BREACH`
   - Increment contractor's `slaBreachCount`
   - Send notification
4. If job is `REQUESTED` and past `acceptanceDeadline`:
   - Auto-expire job → `AUTO_EXPIRED` status
   - Consider reassignment to another contractor

---

### 4.3 GraphQL Schema

#### Queries
```graphql
type Query {
  # Consumer
  getAvailableServices(locationId: ID!): [ServiceWithPrice!]!
  getMyJobs(filters: JobFilters): [Job!]!
  getJobDetails(id: ID!): Job!
  
  # Contractor
  getMyContractorJobs(filters: JobFilters): [Job!]!
  getMyProfile: ContractorProfile!
  
  # Subscription (✅ IMPLEMENTED)
  subscriptionPlans: [SubscriptionPlan!]!                          # Public
  mySubscriptionStatus: SubscriptionStatus!                        # Contractor only
  mySubscriptionHistory: [ContractorSubscription!]!                # Contractor only
  contractorSubscriptionStatus(contractorId: ID!): SubscriptionStatus!  # Admin only
  contractorSubscriptionHistory(contractorId: ID!): [ContractorSubscription!]!  # Admin only
  subscriptionPaymentIntent(id: ID!): SubscriptionPaymentIntent    # Check payment status
  
  # Admin
  getAllServices(filters: ServiceFilters): [Service!]!
  getAllJobs(filters: AdminJobFilters): [Job!]!
  getServicePricingHistory(serviceId: ID!): [ServicePricing!]!
  getSLAHistory(serviceId: ID!): [SLA!]!
  getLocations(filters: LocationFilters): [Location!]!
  getUnclassifiedLocations(filters: LocationFilters): [Location!]!
  getClassifiedLocations(filters: LocationFilters): [Location!]!
  getLocationClassificationStats: ClassificationStats!
  getMarketplaceAnalytics(filters: AnalyticsFilters): MarketplaceAnalytics!
}
```

#### Mutations
```graphql
type Mutation {
  # Consumer
  createJob(input: CreateJobInput!): Job!
  cancelJob(id: ID!, reason: String): Job!
  rateContractor(input: RateContractorInput!): Rating!
  
  # Contractor
  acceptJob(id: ID!): Job!
  rejectJob(id: ID!, reason: String!): Job!
  generateStartOTP(jobId: ID!): OTP!
  startJob(jobId: ID!, otp: String!): Job!
  generateCompletionOTP(jobId: ID!): OTP!
  completeJob(jobId: ID!, otp: String!): Job!
  rateConsumer(input: RateConsumerInput!): Rating!
  # Subscription (✅ IMPLEMENTED)
  initiateSubscriptionUpgrade(input: InitiateSubscriptionUpgradeInput!): SubscriptionUpgradeResponse!
  confirmSubscriptionPayment(input: ConfirmSubscriptionPaymentInput!): SubscriptionConfirmResponse!
  
  # Admin
  createService(input: CreateServiceInput!): Service!
  updateService(id: ID!, input: UpdateServiceInput!): Service!
  createPricingVersion(input: CreatePricingInput!): ServicePricing!
  createSLAVersion(input: CreateSLAInput!): SLA!
  bulkImportLocations(fileKey: String!): ImportResult!
  classifyLocation(id: ID!, areaType: AreaType!): Location!
  bulkClassifyLocations(ids: [ID!]!, areaType: AreaType!): [Location!]!
  updateLocation(id: ID!, input: UpdateLocationInput!): Location!
  reassignJob(jobId: ID!, contractorId: ID!): Job!
  adminSetContractorSubscription(input: AdminSetSubscriptionInput!): SubscriptionConfirmResponse!  # ✅ IMPLEMENTED
}
```

---

### 4.4 Guards & Permissions

Reuse existing RBAC system:

```typescript
// Consumer actions
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.CONSUMER)
async createJob(...)

// Contractor actions
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.CONTRACTOR)
async acceptJob(...)

// Admin actions
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN, UserRole.SUPER_ADMIN)
async createService(...)
```

**Additional Guards:**
- `SubscriptionGuard` - Validate contractor's subscription for premium services
- `JobOwnershipGuard` - Validate user owns the job
- `SLAValidationGuard` - Validate action is within SLA deadline

---

## 5. FRONTEND IMPLEMENTATION

### 5.1 Route Structure

#### **Consumer Routes**
```
/consumer/marketplace/
├── services/                    # List available services
├── service/[id]/                # Service detail + booking
├── jobs/                        # My jobs list
├── job/[id]/                    # Job tracking + rating
```

#### **Contractor Routes**
```
/contractor/marketplace/
├── jobs/                        # All jobs (pending, active, completed)
├── job/[id]/                    # Job details + actions
├── profile/                     # Rating, QR code, serviceable areas
├── subscription/                # FREE vs PREMIUM comparison
```

#### **Admin Routes**
```
/admin/marketplace/
├── services/                    # Service management
├── pricing/                     # Pricing versions
├── sla/                         # SLA management
├── locations/                   # Location master
├── subscriptions/               # Contractor subscriptions
├── analytics/                   # Performance dashboard
```

---

### 5.2 Component Structure

```
/src/components/marketplace/
├── shared/
│   ├── ServiceCard.tsx          # Service display card
│   ├── PriceDisplay.tsx         # Formatted price with area badge
│   ├── SLAIndicator.tsx         # SLA countdown visual
│   ├── JobTimeline.tsx          # Job lifecycle timeline
│   ├── OTPModal.tsx             # OTP input modal
│   ├── RatingStars.tsx          # Star rating component
│   ├── SubscriptionBadge.tsx    # FREE/PREMIUM badge
│   └── SmartMeterLinkStatus.tsx # Shows if job is linked to meter installation
├── consumer/
│   ├── ServiceListingPage.tsx
│   ├── ServiceDetailPage.tsx
│   ├── BookingFlow.tsx
│   ├── JobTrackingPage.tsx
│   └── RatingForm.tsx
├── contractor/
│   ├── JobListPage.tsx
│   ├── JobDetailPage.tsx
│   ├── JobActionsPanel.tsx
│   ├── ProfilePage.tsx
│   └── SubscriptionPage.tsx
└── admin/
    ├── ServiceManagement.tsx
    ├── PricingManagement.tsx
    ├── SLAManagement.tsx
    ├── LocationManagement.tsx
    ├── LocationClassificationPanel.tsx  # Two-panel UI for classifying villages
    └── AnalyticsDashboard.tsx
```

---

### 5.3 Custom Hooks

```typescript
// /src/hooks/marketplace/

// Consumer hooks
useServices(category?: string): { services, loading, error }
useContractorsForService(serviceId: string, locationId: string): { contractors, loading }
useVillageSearch(query: string): { villages, loading } // Search LGD villages
useInitiatePayment(serviceId: string, contractorId: string): { createOrder, loading }
useConsumerJobs(filters?: JobFilters): { jobs, loading, refetch }
useJobDetails(jobId: string): { job, loading, refetch }
useCancelJob(jobId: string): { cancel, loading }

// Contractor hooks
useContractorPendingJobs(): { jobs, loading, refetch }
useContractorActiveJobs(): { jobs, loading, refetch }
useContractorCompletedJobs(): { jobs, loading, refetch }
useAcceptJob(): { accept, loading }
useRejectJob(): { reject, loading } // Triggers refund
useGenerateOTP(jobId: string, type: OTPType): { generateOTP, otp, loading }
useVerifyOTPAndStart(jobId: string): { verify, loading }
useVerifyOTPAndComplete(jobId: string): { verify, loading }
useSubscription(): { subscription, upgrade, loading }

// Admin hooks
useLocations(filters?: LocationFilters): { locations, loading, refetch }
useUnclassifiedLocations(filters?: LocationFilters): { locations, loading, refetch }
useClassifyLocation(): { classify, bulkClassify, loading }
useLocationStats(): { total, classified, unclassified, percentage }
useLGDImport(): { importFile, progress, loading }

// Shared hooks
useSLAStatus(job: Job): { status, timeRemaining, isWarning, isBreached }
useRating(jobId: string): { submitRating, loading }
useRefundStatus(jobId: string): { refund, loading }
```

---

### 5.4 GraphQL Integration

```typescript
// /src/graphql/marketplace/

// queries.ts
export const GET_AVAILABLE_SERVICES = gql`
  query GetAvailableServices($locationId: ID!) {
    getAvailableServices(locationId: $locationId) {
      id
      name
      description
      category
      basePrice
      areaType
      isPremiumOnly
    }
  }
`;

// mutations.ts
export const CREATE_JOB = gql`
  mutation CreateJob($input: CreateJobInput!) {
    createJob(input: $input) {
      id
      jobNumber
      status
      priceSnapshot
      acceptanceDeadline
    }
  }
`;

// fragments.ts
export const JOB_FRAGMENT = gql`
  fragment JobDetails on Job {
    id
    jobNumber
    status
    priceSnapshot
    acceptanceDeadline
    startDeadline
    completionDeadline
    service {
      name
      category
    }
    contractor {
      companyName
      ratingAvg
    }
    consumer {
      name
      phone
    }
  }
`;
```

---

### 5.5 Key Frontend Flows

#### **Consumer Booking Flow (Payment-First Model)**
```
1. Consumer logs in → Selects/confirms their village from dropdown
2. Consumer browses services → GET_SERVICES_BY_CATEGORY
3. Consumer selects a service → View service details
4. System shows list of available contractors in consumer's area
   → GET_CONTRACTORS_FOR_SERVICE(serviceId, locationId)
5. Consumer views contractor profiles (rating, reviews, availability)
6. Consumer selects a contractor → Price displayed (based on village area type)
7. Consumer clicks "Book & Pay" → Redirect to Cashfree payment page
8. On payment success → Cashfree webhook triggers:
   a. Payment marked SUCCESS
   b. Job created with status REQUESTED
   c. SMS sent to contractor: "New job request"
   d. SLA countdown for acceptance starts
9. Consumer sees job in "My Bookings" with status "Awaiting Confirmation"
10. Contractor accepts → SMS to consumer: "Contractor confirmed"
11. Contractor rejects → Auto-refund triggered → SMS to consumer: "Refund initiated"
12. Once contractor starts job → Consumer receives OTP via SMS
13. Consumer shares OTP with contractor (verification at site)
14. Once job completed → Consumer receives completion OTP
15. Consumer shares completion OTP with contractor
16. Job closed → Consumer can rate contractor
```

**Consumer Cancellation:**
```
- Before contractor accepts: Full refund
- After contractor accepts: No cancellation (or partial refund per policy)
```

#### **Contractor Job Handling (Updated)**
```
1. Contractor receives SMS: "New job request for [Service] in [Village]"
2. Contractor opens app → GET_MY_PENDING_JOBS
3. Views job details: Service, Consumer location, Price, SLA deadline
4. Contractor can:
   a. ACCEPT → Job status changes, start deadline begins
   b. REJECT → Must provide reason → Auto-refund triggered to consumer
5. After accepting:
   a. Generate Start OTP → SMS sent to consumer
   b. At site, consumer tells OTP to contractor
   c. Contractor enters OTP → START_JOB → Job started, completion SLA begins
6. After completing work:
   a. Generate Completion OTP → SMS sent to consumer
   b. Consumer tells OTP to contractor
   c. Contractor enters OTP → COMPLETE_JOB → Job completed
7. Job auto-closes → Contractor can rate consumer (optional)
```

**Rejection Consequences:**
- High rejection rate affects contractor visibility in search results
- Rejection reason logged for admin review

#### **Admin Pricing Management**
```
1. Select service
2. View current pricing versions
3. Create new version with effective date
4. System auto-switches pricing on effective date
5. All new jobs use new price
6. Existing jobs retain old price (snapshot)
```

#### **Admin Location Classification Flow**
```
UI Layout:
┌──────────────────────────────────┬──────────────────────────────────┐
│  UNCLASSIFIED VILLAGES           │  CLASSIFICATION PANEL            │
│                                  │                                  │
│  ┌─────────────────────────────┐ │  Selected: 5 villages            │
│  │ 🔲 Bengaluru Rural, KA      │ │                                  │
│  │ 🔲 Mysuru Urban, KA         │ │  ┌─────────────────────────────┐ │
│  │ ✅ Tumkur Town, KA          │ │  │  🏙️  URBAN                  │ │
│  │ ✅ Hassan Village, KA       │ │  └─────────────────────────────┘ │
│  │ ✅ Mandya Town, KA          │ │  ┌─────────────────────────────┐ │
│  │ ... (scrollable)            │ │  │  🏘️  SEMI-URBAN             │ │
│  └─────────────────────────────┘ │  └─────────────────────────────┘ │
│                                  │  ┌─────────────────────────────┐ │
│  Filters: [State ▼] [District ▼]│  │  🌾  RURAL                   │ │
│  [ ] Show classified             │  └─────────────────────────────┘ │
│                                  │                                  │
│  [Select All] [Clear Selection]  │  [CLASSIFY SELECTED]             │
└──────────────────────────────────┴──────────────────────────────────┘

Workflow:
1. Admin uploads Excel with villages (name, district, state, pincode)
   → System imports all as `isClassified = false`
   
2. Admin opens Location Classification page
   → Left panel shows unclassified villages (filterable by state/district)
   
3. Admin selects village(s) from left panel
   → Checkbox selection, supports multi-select
   → "Select All" to select all visible (within current filter)
   
4. Admin clicks area type button (URBAN / SEMI-URBAN / RURAL)
   → Calls bulkClassifyLocations mutation
   → Villages move from unclassified to classified
   → Records who classified and when
   
5. Progress bar shows: "245 of 1,200 villages classified (20%)"

6. Only classified villages are used for:
   → Price resolution
   → SLA resolution
   → Service availability check
```

---

## 6. INTEGRATION POINTS

### 6.1 SmartMeter Integration

**Decision: KEEP SEPARATE**

Meter Installation services are **NOT** part of the Marketplace flow. They follow the existing SmartMeter government installation lifecycle.

**Reasoning:**
- SmartMeter is government-mandated with specific compliance requirements
- Different contractor assignment model (RO assigns, not consumer choice)
- Different payment model (government rates, not marketplace pricing)
- Different verification process (RO verification required)

**Implementation:**
```
1. Marketplace service catalogue does NOT include meter installation
2. If admin creates a service with category = METER_INSTALLATION:
   - System shows warning: "Use SmartMeter module for meter installations"
   - Optionally block creation entirely
3. Consumer dashboard shows:
   - "My Installations" → Links to SmartMeter flow
   - "Book Services" → Links to Marketplace flow
4. Contractor dashboard shows:
   - "Assigned Installations" → SmartMeter jobs from RO
   - "Service Requests" → Marketplace jobs from consumers
```

**Future Integration (Phase 2+):**
If needed later, can add optional linking where:
- Marketplace job can reference an existing SmartMeter installation
- But they remain separate lifecycle tracks

---

### 6.2 Audit Integration

Every Marketplace action must be audited:

```typescript
// In JobsService
await this.auditService.log({
  actor: userId,
  role: user.role,
  entity: 'Job',
  entityId: job.id,
  action: 'ACCEPT_JOB',
  oldState: 'REQUESTED',
  newState: 'ACCEPTED',
  metadata: { contractorId, jobNumber: job.jobNumber }
});
```

---

### 6.3 Notification Integration

Reuse existing `NotificationService`:

```typescript
// When job is created
await this.notificationService.sendSms(
  contractor.phone,
  `New job request: ${job.jobNumber}. Service: ${service.name}. View in app.`
);

// When OTP is generated
await this.notificationService.sendSms(
  consumer.phone,
  `Your OTP for job ${job.jobNumber} is: ${otp}. Valid for 10 minutes.`
);
```

---

## 7. IMPLEMENTATION PHASES

### **Phase 1: Foundation (Week 1-2)**
**Goal:** Database + Core Services + Location Classification

**Backend:**
- [ ] Create database schema files (including location_master with isClassified flag)
- [ ] Run migrations for new tables
- [ ] Extend contractors table
- [ ] Create `LocationsService` with:
  - [ ] bulkImportLocations (Excel upload)
  - [ ] getUnclassifiedLocations
  - [ ] classifyLocation / bulkClassifyLocations
  - [ ] resolveAreaType (only works for classified locations)
- [ ] Create `ServicesService`
- [ ] Create `PricingService` + `SLAsService`

**Testing:**
- [ ] Unit tests for pricing resolution
- [ ] Unit tests for SLA calculation
- [ ] Unit tests for location classification

**Deliverable:** Backend can manage locations (with manual classification), services, pricing, SLAs

---

### **Phase 2: Job Lifecycle (Week 3-4)**
**Goal:** Job creation and state machine

**Backend:**
- [ ] Create `JobsService` with state machine
- [ ] Create `OTPService`
- [ ] Create `MarketplaceSchedulerService`
- [ ] Implement job creation with price/SLA snapshot
- [ ] Implement accept/reject/start/complete logic

**Testing:**
- [ ] Unit tests for state transitions
- [ ] Integration tests for job lifecycle

**Deliverable:** Jobs can be created and managed end-to-end

---

### **Phase 3: Payment & Rating (Week 5)**
**Goal:** Payment integration and rating system

**Backend:**
- [ ] Create `PaymentsService`
- [ ] Create payment webhook controller
- [ ] Create `RatingsService`
- [ ] Integrate payment gateway (Razorpay/Stripe)

**Testing:**
- [ ] Payment webhook verification tests
- [ ] Rating calculation tests

**Deliverable:** Jobs can be paid and rated

---

### **Phase 4: Consumer Frontend (Week 6-7)**
**Goal:** Consumer can browse and book services

**Frontend:**
- [ ] Create `/consumer/marketplace/*` routes
- [ ] Create `ServiceListingPage`
- [ ] Create `ServiceDetailPage`
- [ ] Create `BookingFlow` component
- [ ] Create `JobTrackingPage`
- [ ] Create GraphQL queries/mutations

**Testing:**
- [ ] E2E test: Consumer books service
- [ ] E2E test: Consumer tracks job

**Deliverable:** Consumer can book and track services

---

### **Phase 5: Contractor Frontend (Week 8-9)**
**Goal:** Contractor can manage jobs

**Frontend:**
- [ ] Create `/contractor/marketplace/*` routes
- [ ] Create `JobListPage`
- [ ] Create `JobDetailPage`
- [ ] Create `JobActionsPanel` with OTP input
- [ ] Create `ProfilePage`
- [x] Create `SubscriptionPage` (backend complete, frontend page pending)

**Testing:**
- [ ] E2E test: Contractor accepts and completes job
- [ ] E2E test: Contractor upgrades subscription

**Deliverable:** Contractor can execute jobs end-to-end

---

### **Phase 6: Admin Frontend (Week 10-11)**
**Goal:** Admin can manage marketplace

**Frontend:**
- [ ] Create `/admin/marketplace/*` routes
- [ ] Create `ServiceManagement`
- [ ] Create `PricingManagement`
- [ ] Create `SLAManagement`
- [ ] Create `LocationManagement` with two-panel classification UI
- [ ] Create `LocationClassificationPanel` (unclassified list + area type buttons)
- [ ] Create `AnalyticsDashboard` with charts

**Testing:**
- [ ] E2E test: Admin creates service with pricing
- [ ] E2E test: Admin uploads locations Excel
- [ ] E2E test: Admin classifies villages as Urban/Semi-Urban/Rural
- [ ] E2E test: Bulk classification of multiple villages

**Deliverable:** Admin can configure marketplace and classify locations

---

### **Phase 7: SmartMeter Integration (Week 12)**
**Goal:** Link marketplace jobs to SmartMeter installations

**Backend:**
- [ ] Implement event-based sync
- [ ] Create linking logic in `JobsService`
- [ ] Sync job status with installation status

**Frontend:**
- [ ] Display SmartMeter link status in job details
- [ ] Show installation progress in marketplace job page

**Testing:**
- [ ] Integration test: Meter installation job creates installation record
- [ ] E2E test: Job status syncs with SmartMeter verification

**Deliverable:** Marketplace meter installation jobs integrate with SmartMeter domain

---

### **Phase 8: Production Hardening (Week 13-14)**
**Goal:** Security, performance, monitoring

**Backend:**
- [ ] Add rate limiting to OTP generation
- [ ] Add fraud detection for payments
- [ ] Optimize database queries
- [ ] Add caching for pricing/SLA resolution
- [ ] Add monitoring and alerts

**Frontend:**
- [ ] Add error boundaries
- [ ] Add loading skeletons
- [ ] Optimize bundle size
- [ ] Add analytics tracking

**Testing:**
- [ ] Load testing
- [ ] Security audit
- [ ] Accessibility audit

**Deliverable:** Production-ready system

---

## 8. FILES TO UPDATE

### Backend

#### **Existing Files to Modify:**

1. **`/src/database/schema/contractors.ts`**
   - Add marketplace fields: `subscriptionType`, `subscriptionValidTill`, `ratingAvg`, `totalRatings`, `slaBreachCount`, `qrProfileToken`

2. **`/src/database/schema/index.ts`**
   - Export new marketplace tables

3. **`/src/app.module.ts`**
   - Import `MarketplaceModule`

4. **`/src/modules/contractors/contractors.service.ts`**
   - Add method to update rating average
   - Add method to increment SLA breach count

5. **`/src/modules/notification/notification.service.ts`**
   - Add marketplace notification templates

6. **`/src/modules/audit/audit.service.ts`**
   - Ensure marketplace actions are audited

7. **`/src/common/enums/index.ts`**
   - Add marketplace enums: `AreaType`, `ServiceCategory`, `JobStatus`, `OTPType`, `PaymentStatus`, `SubscriptionType`

---

### Frontend

#### **Existing Files to Modify:**

1. **`/src/lib/auth/AuthProvider.tsx`**
   - No changes needed (marketplace uses existing auth)

2. **`/src/app/(consumer)/consumer/layout.tsx`**
   - Add "Marketplace" link to consumer sidebar

3. **`/src/app/(contractor)/contractor/layout.tsx`**
   - Add "Marketplace" link to contractor sidebar

4. **`/src/app/(admin)/admin/layout.tsx`**
   - Add "Marketplace" section to admin sidebar

5. **`/src/graphql/index.ts`**
   - Export marketplace GraphQL operations

---

## 9. NEW FILES TO CREATE

### Backend

#### **Database Schema:**
```
/src/database/schema/
├── marketplace-locations.ts       # location_master table
├── marketplace-services.ts        # services table
├── marketplace-pricing.ts         # service_pricing table
├── marketplace-slas.ts            # slas table
├── marketplace-mappings.ts        # contractor_location_mapping, contractor_service_mapping
├── marketplace-subscriptions.ts   # contractor_subscriptions table
├── marketplace-jobs.ts            # jobs table
├── marketplace-otps.ts            # job_otps table
├── marketplace-payments.ts        # payments table
├── marketplace-ratings.ts         # ratings table
└── marketplace-sla-breaches.ts    # sla_breaches table
```

#### **Modules:**
```
/src/modules/marketplace/
├── marketplace.module.ts
├── locations/                     # 5 files
├── services/                      # 5 files
├── pricing/                       # 5 files
├── slas/                          # 5 files
├── contractor-extension/          # 5 files
├── subscriptions/ ✅ IMPLEMENTED   # 5 files
│   ├── subscriptions.types.ts     ✅ SubscriptionPlan, SubscriptionPaymentIntent, SubscriptionStatus, etc.
│   ├── subscriptions.service.ts   ✅ initiateUpgrade, confirmPayment, adminSet, expiry check
│   ├── subscriptions.resolver.ts  ✅ GraphQL queries + mutations with RBAC
│   ├── subscriptions.scheduler.ts ✅ Daily cron + startup expiry check
│   └── subscriptions.module.ts    ✅
├── jobs/                          # 6 files (includes state-machine.ts)
├── otp/                           # 5 files
├── payments/                      # 6 files (includes controller.ts)
├── ratings/                       # 5 files
├── scheduler/                     # 3 files
└── index.ts
```

**Total Backend New Files:** ~65 files

---

### Frontend

#### **Pages:**
```
/src/app/
├── (consumer)/consumer/marketplace/
│   ├── page.tsx                   # Services listing
│   ├── service/[id]/page.tsx      # Service detail
│   ├── jobs/page.tsx              # My jobs
│   └── job/[id]/page.tsx          # Job tracking
├── (contractor)/contractor/marketplace/
│   ├── page.tsx                   # Job list
│   ├── job/[id]/page.tsx          # Job detail
│   ├── profile/page.tsx           # Profile
│   └── subscription/page.tsx      # Subscription
└── (admin)/admin/marketplace/
    ├── page.tsx                   # Dashboard
    ├── services/page.tsx          # Service management
    ├── pricing/page.tsx           # Pricing management
    ├── sla/page.tsx               # SLA management
    ├── locations/page.tsx         # Location management
    ├── subscriptions/page.tsx     # Subscription management
    └── analytics/page.tsx         # Analytics
```

#### **Components:**
```
/src/components/marketplace/
├── shared/                        # 7 components
├── consumer/                      # 5 components
├── contractor/                    # 5 components
└── admin/                         # 5 components
```

#### **Hooks:**
```
/src/hooks/marketplace/
├── useServices.ts
├── useCreateJob.ts
├── useConsumerJobs.ts
├── useJobDetails.ts
├── useContractorJobs.ts
├── useJobActions.ts
├── useGenerateOTP.ts
├── usePayment.ts
├── useSLAStatus.ts
├── useRating.ts
└── useSubscription.ts
```

#### **GraphQL:**
```
/src/graphql/marketplace/
├── queries.ts
├── mutations.ts
└── fragments.ts
```

**Total Frontend New Files:** ~45 files

---

## 10. TESTING STRATEGY

### Backend Tests

#### Unit Tests
- [ ] Pricing resolution logic
- [ ] SLA calculation logic
- [ ] Job state machine transitions
- [ ] OTP generation and validation
- [ ] Payment webhook verification
- [ ] Rating calculation

#### Integration Tests
- [ ] Job creation with price/SLA snapshot
- [ ] Job lifecycle (REQUESTED → CLOSED)
- [ ] OTP flow for start/complete
- [ ] Payment gateway integration
- [ ] SmartMeter integration events

#### E2E Tests (using Supertest)
- [ ] Consumer books service
- [ ] Contractor accepts and completes job
- [ ] Admin creates service with pricing
- [ ] SLA breach detection by scheduler

---

### Frontend Tests

#### Component Tests (using React Testing Library)
- [ ] ServiceCard renders correctly
- [ ] OTPModal validates input
- [ ] RatingStars allows selection
- [ ] JobTimeline displays correct state

#### Integration Tests
- [ ] Booking flow completes
- [ ] Job tracking updates in real-time
- [ ] Payment flow redirects correctly

#### E2E Tests (using Playwright)
- [ ] Consumer end-to-end journey
- [ ] Contractor end-to-end journey
- [ ] Admin configuration journey

---

## 11. DEPLOYMENT CHECKLIST

### Pre-Deployment

#### Database
- [ ] Run migrations for all new tables
- [ ] Migrate existing contractors with new fields
- [ ] Generate `qrProfileToken` for existing contractors
- [ ] Seed initial location master data
- [ ] Seed initial services (at least meter installation)

#### Backend
- [ ] Set environment variables for payment gateway
- [ ] Configure OTP service credentials
- [ ] Enable scheduler for SLA checks
- [ ] Set up webhook endpoints with gateway
- [ ] Test webhook signature verification

#### Frontend
- [ ] Build production bundle
- [ ] Test on staging environment
- [ ] Verify all GraphQL operations
- [ ] Test payment flow on test gateway

---

### Post-Deployment

#### Monitoring
- [ ] Set up alerts for SLA breaches
- [ ] Monitor payment webhook failures
- [ ] Track OTP delivery success rate
- [ ] Monitor database query performance

#### Documentation
- [ ] Update API documentation
- [ ] Create user guides for consumers
- [ ] Create user guides for contractors
- [ ] Create admin manual

#### Training
- [ ] Train admin staff on marketplace management
- [ ] Train contractors on job handling
- [ ] Provide consumer onboarding materials

---

## 12. RISK MITIGATION

### Technical Risks

1. **Payment Gateway Integration**
   - **Risk:** Webhook failures or signature mismatches
   - **Mitigation:** Implement retry logic, log all webhook events, manual verification UI

2. **SLA Breach Detection**
   - **Risk:** Scheduler delays or misses breaches
   - **Mitigation:** Run scheduler every 5 minutes, add alerts, allow manual breach recording

3. **SmartMeter Integration**
   - **Risk:** State sync issues between domains
   - **Mitigation:** Use event-based architecture, add reconciliation job

4. **Performance**
   - **Risk:** Slow queries on large job datasets
   - **Mitigation:** Add database indexes, implement caching, paginate results

---

### Business Risks

1. **Pricing Conflicts**
   - **Risk:** Multiple active pricing versions
   - **Mitigation:** Enforce single active version per service+area, admin validation

2. **Contractor Disputes**
   - **Risk:** SLA breach disputes
   - **Mitigation:** Immutable deadlines, audit trail, admin override capability

3. **Payment Fraud**
   - **Risk:** Fake payment confirmations
   - **Mitigation:** Never trust frontend, always verify via webhook, log all attempts

---

## 13. SUCCESS METRICS

### Phase 1 Metrics (Foundation)
- [ ] All database tables created
- [ ] Location master can be imported via Excel
- [ ] Admin can classify villages as Urban/Semi-Urban/Rural
- [ ] Bulk classification works
- [ ] Only classified locations are used for price/SLA resolution
- [ ] Services can be created with pricing
- [ ] SLA resolution works correctly (for classified locations only)

### Phase 2 Metrics (Job Lifecycle)
- [ ] Jobs can be created
- [ ] State transitions work correctly
- [ ] OTP flow is functional
- [ ] Scheduler detects SLA breaches

### Phase 3 Metrics (Payment & Rating)
- [ ] Payment gateway integration works
- [ ] Webhooks are verified correctly
- [ ] Ratings update contractor average

### Phase 4 Metrics (Consumer Frontend)
- [ ] Consumers can browse services
- [ ] Consumers can book services
- [ ] Consumers can track jobs

### Phase 5 Metrics (Contractor Frontend)
- [ ] Contractors can view jobs
- [ ] Contractors can accept/complete jobs
- [x] Contractors can upgrade subscription (backend API complete ✅)

### Phase 6 Metrics (Admin Frontend)
- [ ] Admin can manage services
- [ ] Admin can configure pricing/SLA
- [ ] Admin can view analytics

### Phase 7 Metrics (SmartMeter Integration)
- [ ] Meter installation jobs create SmartMeter records
- [ ] Job status syncs with SmartMeter lifecycle

### Phase 8 Metrics (Production)
- [ ] System handles 1000+ concurrent users
- [ ] API response time <200ms (p95)
- [ ] Payment success rate >99%
- [ ] OTP delivery rate >95%

---

## 14. NEXT STEPS

1. **Review and Approve Plan**
   - Stakeholder review
   - Technical team review
   - Adjust timeline if needed

2. **Set Up Development Environment**
   - Create feature branch: `feature/marketplace`
   - Set up test payment gateway account
   - Configure test SMS gateway

3. **Start Phase 1 Implementation**
   - Create database schema files
   - Write migrations
   - Begin LocationsService implementation

---

## 15. APPENDIX

### A. Environment Variables to Add

```env
# ===========================================
# CASHFREE PAYMENT GATEWAY
# ===========================================
CASHFREE_APP_ID=your_app_id
CASHFREE_SECRET_KEY=your_secret_key
CASHFREE_API_VERSION=2023-08-01
CASHFREE_ENV=TEST  # TEST or PRODUCTION
CASHFREE_WEBHOOK_SECRET=your_webhook_secret

# Payment URLs (for frontend redirect)
CASHFREE_RETURN_URL=https://yourapp.com/payment/return
CASHFREE_NOTIFY_URL=https://yourapi.com/api/payments/webhook

# ===========================================
# MARKETPLACE OTP (via existing SMS service)
# ===========================================
MARKETPLACE_OTP_EXPIRY_MINUTES=10
MARKETPLACE_OTP_MAX_ATTEMPTS=3

# ===========================================
# CRON JOB SCHEDULES
# ===========================================
# SLA breach check - every 5 minutes
CRON_SLA_CHECK=*/5 * * * *

# Subscription expiry check - every hour
CRON_SUBSCRIPTION_CHECK=0 * * * *

# Refund status check - every 15 minutes
CRON_REFUND_CHECK=*/15 * * * *

# ===========================================
# NOTIFICATION (SMS Only for MVP)
# ===========================================
SMS_ENABLED=true
EMAIL_ENABLED=false
PUSH_ENABLED=false
```

---

### B. Database Migration Commands

```bash
# Generate migration
cd smartmeter-backend
pnpm drizzle-kit generate --name add_marketplace_tables

# Apply migration
pnpm drizzle-kit push

# Migrate existing contractors
pnpm run migrate:contractors-marketplace
```

---

### C. GraphQL Schema Preview

```graphql
type Location {
  id: ID!
  name: String!
  state: String!
  district: String!
  taluk: String
  pincode: String
  lgdCode: String               # Backup reference (optional)
  areaType: AreaType            # Nullable until classified
  isClassified: Boolean!
  isActive: Boolean!
  classifiedBy: User
  classifiedAt: DateTime
  createdAt: DateTime!
}

type ClassificationStats {
  total: Int!
  classified: Int!
  unclassified: Int!
  urbanCount: Int!
  semiUrbanCount: Int!
  ruralCount: Int!
  percentageComplete: Float!
}

type Service {
  id: ID!
  name: String!
  description: String
  category: ServiceCategory!
  uom: UnitOfMeasure!
  isPremiumOnly: Boolean!
  isActive: Boolean!
}

type ServiceWithPrice {
  id: ID!
  name: String!
  description: String
  category: ServiceCategory!
  basePrice: Decimal!
  areaType: AreaType!
  isPremiumOnly: Boolean!
}

type Job {
  id: ID!
  jobNumber: String!
  service: Service!
  consumer: Consumer!
  contractor: Contractor
  status: JobStatus!
  priceSnapshot: Decimal!
  areaTypeSnapshot: AreaType!
  acceptanceDeadline: DateTime!
  startDeadline: DateTime
  completionDeadline: DateTime
  isSmartMeterLinked: Boolean!
  installation: Installation
  createdAt: DateTime!
  acceptedAt: DateTime
  startedAt: DateTime
  completedAt: DateTime
  closedAt: DateTime
}

enum JobStatus {
  REQUESTED
  ACCEPTED
  STARTED
  COMPLETED
  CLOSED
  CANCELLED
  AUTO_EXPIRED
  REASSIGNED
  SLA_BREACH
}

enum AreaType {
  URBAN
  SEMI_URBAN
  RURAL
}

enum ServiceCategory {
  ELECTRICAL
  PLUMBING
  CARPENTRY
  METER_INSTALLATION
}
```

### D. Subscription Upgrade Flow ✅ IMPLEMENTED

**Files:**
- `src/modules/marketplace/subscriptions/subscriptions.types.ts`
- `src/modules/marketplace/subscriptions/subscriptions.service.ts`
- `src/modules/marketplace/subscriptions/subscriptions.resolver.ts`
- `src/modules/marketplace/subscriptions/subscriptions.scheduler.ts`
- `src/modules/marketplace/subscriptions/subscriptions.module.ts`

**Contractor Upgrade Flow:**
```
1. GET subscriptionPlans          → View available plans + pricing
2. GET mySubscriptionStatus       → Check current status (daysRemaining, canUpgrade)
3. MUTATION initiateSubscriptionUpgrade(contractorId, duration: MONTHLY/QUARTERLY/YEARLY)
   → Returns: paymentIntent { id, amount, paymentLink, orderId, expiresAt }
4. Contractor pays via paymentLink (Cashfree - currently mock URL)
5. MUTATION confirmSubscriptionPayment(paymentIntentId)
   → Updates mp_contractor_profile.subscriptionType = PREMIUM
   → Updates mp_contractor_profile.subscriptionValidTill = now + duration
   → Deactivates old subscription record in mp_contractor_subscriptions
   → Creates new PREMIUM record in mp_contractor_subscriptions
   → Returns: { success, newSubscriptionType, validTill }
```

**Renewal Logic:**
- Renewal allowed only when ≤30 days remain on current PREMIUM
- New `validTill` extends from existing expiry (not from now) — no gap

**Expiry Flow (Scheduler):**
```
Daily at midnight:
→ Find all mp_contractor_profile where subscriptionType = PREMIUM AND subscriptionValidTill <= NOW
→ For each: set subscriptionType = FREE, create FREE record in mp_contractor_subscriptions
```

**Admin Override:**
```
MUTATION adminSetContractorSubscription(contractorId, subscriptionType, durationMonths?)
→ Directly sets subscription without payment (for manual grants/corrections)
```

**GraphQL Types Added:**
```graphql
enum SubscriptionDuration { MONTHLY QUARTERLY YEARLY }
enum SubscriptionPaymentStatus { PENDING SUCCESS FAILED EXPIRED }

type SubscriptionPlan { duration durationMonths price discountPercent finalPrice description }
type SubscriptionPaymentIntent { id contractorId duration amount status paymentLink orderId expiresAt createdAt }
type ContractorSubscription { id contractorId subscriptionType startDate endDate paymentId isActive createdAt }
type SubscriptionStatus { currentType validTill isActive daysRemaining canUpgrade canRenew }
type SubscriptionUpgradeResponse { success message paymentIntent }
type SubscriptionConfirmResponse { success message newSubscriptionType validTill }

input InitiateSubscriptionUpgradeInput { contractorId duration }
input ConfirmSubscriptionPaymentInput { paymentIntentId paymentReference? }
input AdminSetSubscriptionInput { contractorId subscriptionType durationMonths? }
```

**⚠️ Production TODOs:**
- Replace in-memory `paymentIntents` Map with a DB table or Redis
- Wire `paymentLink` to real Cashfree payment order creation
- Add webhook endpoint to auto-call `confirmPayment` on Cashfree success event

---

**Document Version:** 1.0  
**Last Updated:** 17 February 2026  
**Next Review:** After Phase 1 completion

---

**CRITICAL REMINDERS:**

1. ✅ NEVER modify existing SmartMeter domain logic
2. ✅ ALWAYS snapshot price and SLA at job creation
3. ✅ ALWAYS verify payment via webhook
4. ✅ ALWAYS audit state transitions
5. ✅ ALWAYS validate OTP server-side
6. ✅ NEVER trust frontend for critical logic
7. ✅ ALWAYS maintain backward compatibility
8. ✅ ALWAYS test before merging

---
