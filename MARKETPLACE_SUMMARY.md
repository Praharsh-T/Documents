# MARKETPLACE INTEGRATION - QUICK REFERENCE SUMMARY

**Generated:** 17 February 2026  
**Updated:** 18 February 2026  
**Full Plan:** See [MARKETPLACE_IMPLEMENTATION_PLAN.md](./MARKETPLACE_IMPLEMENTATION_PLAN.md)

---

## PROJECT STRUCTURE

| Folder | Purpose |
|--------|---------|
| `userservice/` | Backend (NestJS) - formerly `smartmeter-backend` |
| `web-frontend/` | Frontend (Next.js) - formerly `smartmeter-frontend` |

---

## WHAT IS BEING BUILT?

A **Service Marketplace** for electrical services that allows:
- ✅ Consumers to browse services and select contractors
- ✅ **Payment-first model** - Consumer pays before contractor accepts
- ✅ Contractors can accept/reject jobs (rejection = auto-refund)
- ✅ Location-based pricing using LGD village data
- ✅ SLA-driven job management with deadline tracking
- ✅ OTP-based job start/completion verification
- ✅ **Cashfree** payment gateway integration
- ✅ Dual rating system (consumer ↔ contractor)
- ✅ SMS notifications only (for MVP)

**NOT Included:**
- ❌ Meter Installation (stays in SmartMeter module, separate flow)
- ❌ Add-on services (skipped for MVP)
- ❌ Email/Push notifications (SMS only for MVP)

---

## KEY FLOW: Payment-First Model

```
Consumer → Browse Services → Select Contractor → Pay via Cashfree →
Job Created (REQUESTED) → Contractor Accept/Reject →
If Accept: OTP flow → Job Completed → Rating
If Reject: Auto-Refund to Consumer
```

---

## KEY PRINCIPLES

1. **NO BREAKING CHANGES** - SmartMeter domain remains untouched
2. **Payment Before Acceptance** - Consumer pays first, refund if rejected
3. **Snapshot Everything** - Price and SLA frozen at job creation
4. **Backend Authority** - All validation server-side
5. **SmartMeter Separate** - Meter installation NOT in marketplace

---

## DATABASE CHANGES

### New Tables (12 tables)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `location_master` | LGD villages with admin-classified area type | villageCode, villageName, stateName, districtName, areaType, isClassified |
| `services` | Service catalog | name, category, uom, isPremiumOnly |
| `service_pricing` | Versioned pricing per service+area | serviceId, areaType, basePrice, effectiveFrom |
| `slas` | Versioned SLA per service+area | serviceId, areaType, acceptanceHours, jobStartHours |
| `contractor_location_mapping` | Contractor serviceable areas | contractorId, locationId |
| `contractor_service_mapping` | Contractor offered services | contractorId, serviceId |
| `contractor_subscriptions` | Subscription history | contractorId, subscriptionType, startDate, endDate |
| `jobs` | Main job entity with state machine | jobNumber, status, priceSnapshot, contractorId, paymentId |
| `job_otps` | OTP for job start/completion | jobId, otpType, code, expiresAt |
| `payments` | Cashfree payment tracking | jobId, amount, cfOrderId, cfPaymentId, status |
| `refunds` | Refund tracking (when rejected/cancelled) | paymentId, jobId, cfRefundId, status |
| `ratings` | Dual rating system | jobId, raterUserId, rateeUserId, rating |
| `sla_breaches` | SLA violation audit | jobId, breachType, breachedAt |

### Modified Tables (1 table)

| Table | New Fields |
|-------|------------|
| `contractors` | `subscriptionType`, `subscriptionValidTill`, `ratingAvg`, `totalRatings`, `slaBreachCount`, `qrProfileToken` |

---

## BACKEND CHANGES

### New Modules (12 modules)

**Location:** `/userservice/src/modules/marketplace/`

```
/marketplace/
├── marketplace.module.ts           # Root module
├── locations/                      # LGD import + manual classification
├── services/                       # Service catalog
├── pricing/                        # Versioned pricing
├── slas/                           # Versioned SLA
├── contractor-extension/           # Contractor marketplace features
├── subscriptions/                  # Contractor subscription management
├── jobs/                           # Job lifecycle & state machine
├── otp/                            # OTP generation & validation
├── payments/                       # Cashfree integration
├── refunds/                        # Refund processing
├── ratings/                        # Rating system
└── scheduler/                      # Cron jobs for SLA, refunds
```

**Total New Backend Files:** ~65 files

### Modified Backend Files (7 files)

| File | Changes |
|------|---------|
| `/src/database/schema/contractors.ts` | Add marketplace fields |
| `/src/database/schema/index.ts` | Export marketplace tables |
| `/src/app.module.ts` | Import MarketplaceModule |
| `/src/modules/contractors/contractors.service.ts` | Add rating/SLA breach methods |
| `/src/modules/notification/notification.service.ts` | Add marketplace notification templates |
| `/src/modules/audit/audit.service.ts` | Ensure marketplace auditing |
| `/src/common/enums/index.ts` | Add marketplace enums |

---

## FRONTEND CHANGES

### New Routes

#### Consumer Routes (4 routes)
```
/consumer/marketplace/services            # Browse services
/consumer/marketplace/service/[id]        # Service detail + booking
/consumer/marketplace/jobs                # My jobs list
/consumer/marketplace/job/[id]            # Job tracking + rating
```

#### Contractor Routes (4 routes)
```
/contractor/marketplace/jobs              # Job list (pending, active, completed)
/contractor/marketplace/job/[id]          # Job details + actions (accept/start/complete)
/contractor/marketplace/profile           # Profile, rating, QR code
/contractor/marketplace/subscription      # FREE vs PREMIUM upgrade
```

#### Admin Routes (7 routes)
```
/admin/marketplace/services               # Service management
/admin/marketplace/pricing                # Pricing version management
/admin/marketplace/sla                    # SLA version management
/admin/marketplace/locations              # Location master upload
/admin/marketplace/subscriptions          # Contractor subscriptions
/admin/marketplace/analytics              # Performance dashboard
```

**Total New Frontend Files:** ~45 files

### New Components (22 components)

```
/src/components/marketplace/
├── shared/
│   ├── ServiceCard.tsx                   # Service display card
│   ├── PriceDisplay.tsx                  # Formatted price with area badge
│   ├── SLAIndicator.tsx                  # SLA countdown visual
│   ├── JobTimeline.tsx                   # Job lifecycle timeline
│   ├── OTPModal.tsx                      # OTP input modal
│   ├── RatingStars.tsx                   # Star rating component
│   └── SubscriptionBadge.tsx             # FREE/PREMIUM badge
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
    ├── LocationClassificationPanel.tsx   # Two-panel UI for classifying villages
    └── AnalyticsDashboard.tsx
```

### New Hooks (15 hooks)

```
/src/hooks/marketplace/
├── useServices.ts                        # Fetch available services
├── useCreateJob.ts                       # Create job
├── useConsumerJobs.ts                    # Get consumer's jobs
├── useJobDetails.ts                      # Get job details
├── useContractorJobs.ts                  # Get contractor's jobs
├── useJobActions.ts                      # Accept/reject/start/complete
├── useGenerateOTP.ts                     # Generate OTP
├── usePayment.ts                         # Payment flow
├── useSLAStatus.ts                       # SLA countdown
├── useRating.ts                          # Submit rating
├── useSubscription.ts                    # Subscription management
├── useLocations.ts                       # Fetch all locations (admin)
├── useUnclassifiedLocations.ts           # Fetch unclassified villages (admin)
├── useClassifyLocation.ts                # Classify villages as Urban/Rural/Semi-Urban
└── useLocationStats.ts                   # Classification progress stats
```

### New GraphQL Files (3 files)

```
/src/graphql/marketplace/
├── queries.ts                            # All queries
├── mutations.ts                          # All mutations
└── fragments.ts                          # Reusable fragments
```

### Modified Frontend Files (4 files)

**Location:** `/web-frontend/src/`

| File | Changes |
|------|---------|
| `app/(consumer)/consumer/layout.tsx` | Add "Marketplace" link to sidebar |
| `app/(contractor)/contractor/layout.tsx` | Add "Marketplace" link to sidebar |
| `app/(admin)/admin/layout.tsx` | Add "Marketplace" section to sidebar |
| `graphql/index.ts` | Export marketplace operations |

---

## JOB STATE MACHINE (Payment-First Model)

```
Consumer Pays via Cashfree
         │
         ▼
    PAYMENT_PENDING ──────────────────► CANCELLED (payment failed)
         │
         │ (Cashfree webhook: success)
         ▼
     REQUESTED ───────────────────────► REJECTED ──► REFUNDED
         │                                  │
         │ (Contractor accepts)             └──► (Auto-refund)
         ▼
     ACCEPTED
         │
         │ (OTP verified - Start)
         ▼
     STARTED
         │
         │ (OTP verified - Complete)
         ▼
    COMPLETED
         │
         │ (Consumer rates)
         ▼
      CLOSED

Cancellation paths:
- REQUESTED → CANCELLED → REFUNDED (consumer cancels)
- ANY active state → SLA_BREACH (scheduler detection)
```

---

## CRITICAL BUSINESS RULES

### Payment-First Flow
- ✅ Consumer pays BEFORE contractor accepts
- ✅ Payment via Cashfree (webhook verification)
- ✅ If contractor REJECTS → Auto-refund triggered
- ✅ If consumer CANCELS before acceptance → Auto-refund

### Pricing
- ✅ Price resolved based on consumer's village area type
- ✅ Consumer selects village from LGD dropdown
- ✅ Price snapshot stored in `jobs.priceSnapshot` (immutable)
- ✅ NEVER recalculate price after payment

### SLA
- ✅ SLA resolved at job creation based on service + area type
- ✅ All deadlines calculated and snapshotted (immutable)
- ✅ Cron job checks every 5 minutes for breaches
- ✅ SMS notification at 80% threshold

### OTP
- ✅ OTP required for job start and completion
- ✅ Valid for 10 minutes (configurable)
- ✅ Max 3 generation attempts per action
- ✅ SMS sent to consumer's phone

### Payment
- ✅ Payment via Cashfree (India)
- ✅ NEVER trust frontend payment confirmation
- ✅ Always verify via Cashfree webhook signature
- ✅ Job created only after payment success

### Refunds
- ✅ Auto-refund when contractor rejects
- ✅ Auto-refund when consumer cancels before acceptance
- ✅ Refund via Cashfree Refund API
- ✅ Cron job checks refund status every 15 minutes

### Subscription
- ✅ FREE contractors: Limited service categories
- ✅ PREMIUM contractors: All services
- ✅ Backend validates subscription before contractor appears in search
- ✅ **Note:** Meter Installation is NOT in marketplace

### SmartMeter Integration
- ❌ **KEPT SEPARATE** - Meter Installation uses existing SmartMeter flow
- ❌ No marketplace job for meter installation
- ✅ Consumer dashboard shows both separately:
  - "My Installations" → SmartMeter
  - "Book Services" → Marketplace

---

## IMPLEMENTATION PHASES

| Phase | Timeline | Deliverable |
|-------|----------|-------------|
| **Phase 1** | Week 1-2 | Database + LGD Import + Classification UI |
| **Phase 2** | Week 3-4 | Services + Pricing + SLA management |
| **Phase 3** | Week 5-6 | Job lifecycle + Cashfree + Refunds |
| **Phase 4** | Week 7-8 | Consumer frontend (browse, pay, track) |
| **Phase 5** | Week 9-10 | Contractor frontend (accept/reject, OTP) |
| **Phase 6** | Week 11-12 | Admin frontend (manage marketplace) |
| **Phase 7** | Week 13-14 | Production hardening + testing |

**Total Timeline:** ~14 weeks

---

## GRAPHQL API PREVIEW

### Sample Queries

```graphql
# Consumer: Search villages (LGD data)
query SearchVillages($query: String!, $stateCode: String) {
  searchVillages(query: $query, stateCode: $stateCode) {
    id
    villageCode
    villageName
    districtName
    stateName
    areaType
  }
}

# Consumer: Get contractors for a service in their area
query GetContractorsForService($serviceId: ID!, $locationId: ID!) {
  getContractorsForService(serviceId: $serviceId, locationId: $locationId) {
    id
    companyName
    ratingAvg
    totalRatings
    subscriptionType
    priceForService  # Calculated based on area type
  }
}

# Contractor: Get my pending jobs
query GetMyPendingJobs {
  getMyPendingJobs {
    id
    jobNumber
    status
    service { name }
    consumer { name }
    location { villageName, districtName }
    priceSnapshot
    acceptanceDeadline
  }
}
```

### Sample Mutations

```graphql
# Consumer: Initiate payment (creates Cashfree order)
mutation InitiatePayment($input: InitiatePaymentInput!) {
  initiatePayment(input: $input) {
    cfOrderId
    paymentSessionId
    paymentLink
  }
}

# Contractor: Accept job
mutation AcceptJob($jobId: ID!) {
  acceptJob(jobId: $jobId) {
    id
    status
    acceptedAt
    startDeadline
  }
}

# Contractor: Reject job (triggers refund)
mutation RejectJob($jobId: ID!, $reason: String!) {
  rejectJob(jobId: $jobId, reason: $reason) {
    id
    status
    refund { id, status }
  }
}

# Contractor: Generate OTP for job start
mutation GenerateStartOTP($jobId: ID!) {
  generateStartOTP(jobId: $jobId) {
    otpSentTo  # Masked phone number
    expiresAt
  }
}

# Admin: Classify villages
mutation BulkClassifyLocations($villageIds: [ID!]!, $areaType: AreaType!) {
  bulkClassifyLocations(villageIds: $villageIds, areaType: $areaType) {
    classifiedCount
    failedCount
  }
}
```

---

## TESTING CHECKLIST

### Backend
- [ ] Unit tests for pricing resolution
- [ ] Unit tests for SLA calculation
- [ ] Unit tests for job state transitions
- [ ] Integration tests for OTP flow
- [ ] Integration tests for payment webhook
- [ ] E2E test: Complete job lifecycle

### Frontend
- [ ] Component tests for all marketplace components
- [ ] Integration test: Consumer booking flow
- [ ] Integration test: Contractor job handling
- [ ] E2E test: Full consumer journey
- [ ] E2E test: Full contractor journey
- [ ] E2E test: Admin configuration

---

## ENVIRONMENT VARIABLES TO ADD

```env
# Payment Gateway
PAYMENT_GATEWAY_PROVIDER=razorpay
PAYMENT_GATEWAY_KEY=rzp_test_xxx
PAYMENT_GATEWAY_SECRET=xxx
PAYMENT_WEBHOOK_SECRET=whsec_xxx

# Marketplace OTP
MARKETPLACE_OTP_EXPIRY_MINUTES=10
MARKETPLACE_OTP_MAX_ATTEMPTS=3

# Marketplace Scheduler
MARKETPLACE_SLA_CHECK_INTERVAL=5
MARKETPLACE_SUBSCRIPTION_CHECK_INTERVAL=60
```

---

## DEPLOYMENT STEPS

### Pre-Deployment
1. Run database migrations: `pnpm drizzle-kit generate && pnpm drizzle-kit push`
2. Migrate existing contractors with new fields
3. Generate QR profile tokens for existing contractors
4. Seed location master data (Excel import)
5. Seed initial services (at least meter installation)
6. Configure payment gateway webhooks
7. Test payment gateway on test environment

### Post-Deployment
1. Monitor SLA breach detection
2. Monitor payment webhook success rate
3. Monitor OTP delivery success rate
4. Train admin staff
5. Provide user guides to contractors
6. Provide consumer onboarding materials

---

## FILE COUNT SUMMARY

| Category | Existing Files Modified | New Files Created | Total |
|----------|------------------------|-------------------|-------|
| **Backend** | 7 | 65 | 72 |
| **Frontend** | 4 | 50 | 54 |
| **Database Schema** | 2 | 11 | 13 |
| **Total** | **13** | **126** | **139** |

---

## RISK MITIGATION

### Top 5 Risks

1. **Payment Gateway Integration**
   - Risk: Webhook failures
   - Mitigation: Retry logic, manual verification UI

2. **SLA Breach Detection**
   - Risk: Scheduler delays
   - Mitigation: Run every 5 minutes, manual override

3. **SmartMeter Integration**
   - Risk: State sync issues
   - Mitigation: Event-based architecture, reconciliation job

4. **Performance**
   - Risk: Slow queries on large datasets
   - Mitigation: Database indexes, caching, pagination

5. **Pricing Conflicts**
   - Risk: Multiple active versions
   - Mitigation: Enforce single active version, admin validation

---

## SUCCESS CRITERIA

### Functional
- ✅ Consumers can browse and book services
- ✅ Contractors can accept and complete jobs
- ✅ Admin can manage services, pricing, SLA
- ✅ Payments are processed securely
- ✅ SLA breaches are detected automatically
- ✅ SmartMeter integration works seamlessly

### Non-Functional
- ✅ API response time <200ms (p95)
- ✅ Payment success rate >99%
- ✅ OTP delivery rate >95%
- ✅ SLA breach detection within 5 minutes
- ✅ System handles 1000+ concurrent users
- ✅ Zero downtime deployment

---

## NEXT IMMEDIATE STEPS

1. ✅ **Review this plan** with stakeholders
2. ⏳ **Create feature branch:** `feature/marketplace`
3. ⏳ **Set up test environment** (payment gateway, SMS gateway)
4. ⏳ **Start Phase 1:** Database schema + migrations
5. ⏳ **Begin LocationsService** implementation

---

## QUICK LINKS

- [Full Implementation Plan](./MARKETPLACE_IMPLEMENTATION_PLAN.md)
- [Original Requirement](./MARKETPLACE.md)
- [Project Brain (System Architecture)](./PROJECT_BRAIN.md)
- [Session Changes Log](./SESSION_CHANGES.md)

---

**Document Version:** 1.0  
**Generated:** 17 February 2026  
**Maintained By:** Development Team

---

**REMEMBER:**
- ⚠️ NO breaking changes to SmartMeter domain
- ⚠️ ALWAYS snapshot price and SLA at job creation
- ⚠️ NEVER trust frontend for critical logic
- ⚠️ ALWAYS verify payment via webhook
- ⚠️ ALWAYS audit state transitions
