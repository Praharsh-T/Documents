# Marketplace Mobile App — API Flows

**For:** Consumer App & Contractor App  
**Backend:** `user-service` (NestJS GraphQL)  
**Last Updated:** 27 February 2026  
**Base GraphQL URL:** `https://connectshakti.klonec.cloud/graphql`  
**Auth:** All requests need `Authorization: Bearer <jwt_token>` header

---

## Table of Contents

1. [Authentication (Login)](#1-authentication)
2. [Consumer Flow](#2-consumer-flow)
3. [Contractor Flow](#3-contractor-flow)
4. [Shared APIs (Both Roles)](#4-shared-apis)
5. [Enums Reference](#5-enums-reference)
6. [Error Codes](#6-error-codes)

---

## 1. Authentication

Both consumer and contractor log in via **OTP on phone number**.

### Step 1 — Request OTP

```graphql
mutation RequestOtp($input: RequestLoginOtpInput!) {
  requestLoginOtp(input: $input) {
    success
    message
  }
}
```

**Input:**

```json
{
  "input": {
    "phone": "9876543210",
    "loginType": "CONSUMER" // or "CONTRACTOR"
  }
}
```

> `loginType` must match the user's actual role. Wrong type → error.

---

### Step 2 — Verify OTP → Get Token

```graphql
mutation VerifyOtp($input: VerifyLoginOtpInput!) {
  verifyLoginOtp(input: $input) {
    accessToken
    refreshToken
    user {
      id
      name
      phone
      role
    }
  }
}
```

**Input:**

```json
{
  "input": {
    "phone": "9876543210",
    "otp": "123456",
    "loginType": "CONSUMER"
  }
}
```

**Save:** Store `accessToken` — attach as `Authorization: Bearer <accessToken>` on all subsequent requests. Store `refreshToken` separately for obtaining new access tokens when the current one expires.

---

## 2. Consumer Flow

```
Select Village → Browse Services → Search Contractors → Create Job
  → Simulate Payment → Track Job → View OTP → Rate Contractor
```

---

### Step 1 — Select Location (Cascading Dropdowns)

Consumer must pick their village to get area-based pricing.

#### 1a. Get all states

```graphql
query GetStates {
  marketplaceStates {
    code
    name
  }
}
```

**Returns:** `[{ code: "29", name: "Karnataka" }]`

---

#### 1b. Get districts for selected state

```graphql
query GetDistricts($stateCode: String!) {
  marketplaceDistricts(stateCode: $stateCode) {
    code
    name
    stateCode
  }
}
```

**Input:** `{ "stateCode": "29" }` (from Step 1a)

---

#### 1c. Get sub-districts for selected district

```graphql
query GetSubDistricts($districtCode: String!) {
  marketplaceSubDistricts(districtCode: $districtCode) {
    code
    name
    districtCode
  }
}
```

**Input:** `{ "districtCode": "572" }` (from Step 1b)

---

#### 1d. Get villages for selected sub-district

```graphql
query GetVillages(
  $districtCode: String!
  $subDistrictCode: String
  $classifiedOnly: Boolean
) {
  marketplaceVillages(
    districtCode: $districtCode
    subDistrictCode: $subDistrictCode
    classifiedOnly: $classifiedOnly
  ) {
    id
    code
    name
    nameLocal
    category
    areaType
    isClassified
  }
}
```

**Input:**

```json
{
  "districtCode": "572",
  "subDistrictCode": "5720001",
  "classifiedOnly": true
}
```

> **IMPORTANT:** Only villages where `isClassified: true` and `areaType` is set will appear when `classifiedOnly: true`. These are the only villages eligible for booking. Admin must classify villages first.

**Save:** The selected village's `id` = **`locationId`** (used in every booking step). Also save `areaType` (URBAN / SEMI_URBAN / RURAL) — used to fetch prices.

---

### Step 2 — Browse Services with Prices

#### Option A — Grouped by category (recommended for UI)

```graphql
query ServicesByCategory($areaType: AreaType!) {
  marketplaceServicesByCategory(areaType: $areaType) {
    category {
      id
      name
      code
      isFreeAvailable
    }
    services {
      id
      name
      description
      currentPrice
      isFreeAvailable
      category {
        id
        name
        code
        isFreeAvailable
      }
      uom {
        id
        name
        code
        requiresQuantity
      }
    }
  }
}
```

**Input:** `{ "areaType": "URBAN" }` (from selected village's `areaType`)

---

#### Option B — Flat list with prices

```graphql
query ServicesWithPrices($areaType: AreaType!) {
  marketplaceServicesWithPrices(areaType: $areaType) {
    id
    name
    description
    currentPrice
    isFreeAvailable
    category {
      id
      name
      code
    }
    uom {
      id
      name
      code
      requiresQuantity
    }
  }
}
```

**Save:** Selected service's `id` = **`serviceId`**. Note `uom.requiresQuantity` — if `true`, ask the user for a quantity input, else quantity defaults to `1`.

---

### Step 3 — Search Contractors

```graphql
query SearchContractors($input: ContractorSearchInput!) {
  searchMarketplaceContractors(input: $input) {
    items {
      contractorId
      contractorName
      companyName
      phone
      ratingAvg
      totalRatings
      totalJobsCompleted
      isPremiumContractor
      priceForService
    }
    pagination {
      total
      page
      limit
      totalPages
      hasNext
      hasPrev
    }
  }
}
```

**Input:**

```json
{
  "input": {
    "locationId": "<village-uuid>",
    "serviceId": "<service-uuid>",
    "minRating": null,
    "sortBy": "rating",
    "sortOrder": "desc",
    "page": 1,
    "limit": 20
  }
}
```

> `locationId` = village `id` from Step 1d  
> `serviceId` = service `id` from Step 2

**What it does internally:** Finds contractors who:

1. Are **PREMIUM** subscribers (FREE contractors do not appear in search)
2. Have an active marketplace profile
3. Have selected a coverage area that **includes the consumer's location** — hierarchical match: contractor who selected the consumer's district covers all villages in it; one who selected the sub-district covers all villages in it; one who selected the exact village covers that village only
4. Offer the selected service
5. Have pricing configured for the location's area type

**Save:** Selected contractor's `contractorId` = **`contractorId`** (for booking).

---

#### Optional — View Contractor Profile & Reviews

```graphql
query ContractorProfile($contractorId: ID!) {
  marketplaceContractorProfile(contractorId: $contractorId) {
    id
    contractorId
    contractorName
    companyName
    phone
    subscriptionType
    subscriptionValidTill
    ratingAvg
    totalRatings
    totalJobsCompleted
    slaBreachCount
    isMarketplaceActive
  }
}

query ContractorReviews($contractorId: ID!, $page: Int, $limit: Int) {
  marketplaceContractorReviews(
    contractorId: $contractorId
    page: $page
    limit: $limit
  ) {
    items {
      id
      consumerName
      rating
      comment
      serviceName
      createdAt
    }
    summary {
      averageRating
      totalRatings
      fiveStarCount
      fourStarCount
      threeStarCount
      twoStarCount
      oneStarCount
    }
  }
}
```

---

### Step 4 — Create Job

```graphql
mutation CreateJob($input: CreateJobInput!) {
  createMarketplaceJob(input: $input) {
    jobId
    jobNumber
    amount
    paymentSessionId
    paymentUrl
  }
}
```

**Input:**

```json
{
  "input": {
    "contractorId": "<contractor-uuid>",
    "serviceId": "<service-uuid>",
    "locationId": "<village-uuid>",
    "quantity": 1,
    "consumerAddress": "123 Main Street, Yelahanka",
    "consumerPhone": "9876543210"
  }
}
```

> `quantity` — only required if `uom.requiresQuantity = true`, otherwise send `1`  
> `consumerAddress` and `consumerPhone` are optional but recommended

**What happens internally:**

1. Verifies contractor is active
2. Verifies service is active
3. Verifies location is classified and has `areaType`
4. Looks up current price for `serviceId + areaType` → **snapshotted forever**
5. Looks up current SLA for `serviceId + areaType` → **snapshotted forever**
6. Calculates `acceptanceDeadline` = now + SLA.acceptanceHours
7. Creates job with status `PAYMENT_PENDING`

**Returns:** `jobId`, `jobNumber`, `amount` (total to pay), `paymentSessionId` (placeholder), `paymentUrl` (null until Cashfree integrated)

**Save:** `jobId` is needed for all subsequent steps.

---

### Step 5 — Simulate Payment _(DEV ONLY — will be replaced by Cashfree)_

```graphql
mutation SimulatePayment($jobId: ID!) {
  simulatePaymentSuccess(jobId: $jobId) {
    id
    jobNumber
    status
    paidAt
  }
}
```

**Input:** `{ "jobId": "<job-uuid>" }`

**What it does:** Moves job from `PAYMENT_PENDING` → `REQUESTED` and sets `paidAt`. In production this will be triggered by Cashfree webhook.

> ⚠️ This is available to all authenticated users for development. Remove/guard before production.

---

### Step 6 — Track Job

#### Poll single job (refresh every 10s for active jobs)

```graphql
query GetJob($id: ID!) {
  marketplaceJob(id: $id) {
    id
    jobNumber
    status
    consumerId
    consumerName
    contractorId
    contractorName
    contractorPhone
    serviceId
    serviceName
    locationId
    locationName
    consumerAddress
    priceSnapshot
    areaTypeSnapshot
    acceptanceDeadline
    startDeadline
    completionDeadline
    createdAt
    paidAt
    acceptedAt
    rejectedAt
    startedAt
    completedAt
    closedAt
    rejectionReason
    updatedAt
  }
}
```

#### List all my jobs

```graphql
query MyJobs($filters: JobsFilterInput) {
  myMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      status
      serviceName
      contractorName
      contractorPhone
      locationName
      priceSnapshot
      acceptanceDeadline
      startDeadline
      completionDeadline
      createdAt
      paidAt
      acceptedAt
      startedAt
      completedAt
      rejectedAt
      rejectionReason
    }
    pagination {
      total
      page
      limit
      totalPages
      hasNext
      hasPrev
    }
  }
}
```

**Filter by status:**

```json
{
  "filters": {
    "status": "REQUESTED",
    "page": 1,
    "limit": 20
  }
}
```

> Omit `status` to get all jobs.

---

### Step 7 — Show OTP to Contractor

OTPs are **auto-generated by the backend** — consumer does not request them:

- **START OTP** → generated automatically when contractor accepts the job. Consumer sees it immediately in their app and receives it via SMS.
- **COMPLETE OTP** → generated automatically when the job moves to `STARTED`. Consumer sees it in their app and receives it via SMS.

Poll this query to display the current active OTP:

```graphql
query ActiveOtp($jobId: ID!) {
  activeJobOtp(jobId: $jobId) {
    id
    jobId
    otpType
    code
    expiresAt
    isUsed
    createdAt
  }
}
```

**Input:** `{ "jobId": "<job-uuid>" }`

> **When job is `ACCEPTED`:** shows the START OTP — display it prominently so consumer can tell contractor when they arrive.  
> **When job is `STARTED`:** shows the COMPLETE OTP — display it so consumer can confirm work is done.  
> Returns `null` only if both OTPs have been used or expired.  
> Poll every 10s while job is active. Show `expiresAt` countdown so consumer knows the code is still valid.

---

### Step 8 — Cancel Job _(before STARTED)_

```graphql
mutation CancelJob($input: CancelJobInput!) {
  cancelMarketplaceJob(input: $input) {
    id
    jobNumber
    status
    updatedAt
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "reason": "Changed my mind"
  }
}
```

> Allowed only when job is in `PAYMENT_PENDING`, `REQUESTED`, or `ACCEPTED` status.  
> Cannot cancel once job is `STARTED`.

---

### Step 9 — Rate Contractor _(after COMPLETED)_

```graphql
mutation RateContractor($input: CreateRatingInput!) {
  createMarketplaceRating(input: $input) {
    id
    jobId
    rating
    comment
    createdAt
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "rating": 5,
    "comment": "Excellent work, very professional!"
  }
}
```

> Only allowed when job is `COMPLETED`.  
> After consumer rates → job moves to `CLOSED`.  
> Rating is 1–5. Comment optional (max 500 chars).

---

## 3. Contractor Flow

```
[Admin creates marketplace profile once]
  → Contractor upgrades to PREMIUM
  → Contractor sets Coverage Areas (district / sub-district / village)
  → Contractor selects Services they offer
  → Contractor appears in consumer search
  → View Incoming Jobs → Accept / Reject
  → [Backend auto-generates START OTP → Consumer sees it]
  → Enter Start OTP (consumer tells you) → job STARTED
  → [Backend auto-generates COMPLETE OTP → Consumer sees it]
  → Enter Completion OTP (consumer tells you) → job COMPLETED
  → Rate Consumer (optional)
  → Manage Subscription
```

> **Pre-requisite:** Admin must have created a contractor marketplace profile (one-time setup). After that, the contractor themselves selects their coverage areas and services. **PREMIUM subscription is required to appear in consumer searches.**

---

### Admin Setup for a Contractor _(one-time — Admin creates the profile only)_

> These mutations require **ADMIN** or **SUPER_ADMIN** role.

#### Create Marketplace Profile

```graphql
mutation {
  createContractorMarketplaceProfile(
    input: { contractorId: "<contractor-uuid>" }
  ) {
    id
    contractorId
    subscriptionType
    isMarketplaceActive
  }
}
```

> That's it from Admin's side. Location assignment is no longer Admin-managed — the contractor sets their own coverage areas after upgrading to PREMIUM (see below).

---

### Contractor Self-Service: Coverage Areas _(PREMIUM only)_

After upgrading to PREMIUM, the contractor selects the geographic areas they want to serve. This is what makes them appear in consumer searches.

#### Rules

- **State-level selection is not allowed** — must select at district level or below
- **DISTRICT** → covers all consumers in that district
- **SUB_DISTRICT** → covers all consumers in that taluk/block
- **VILLAGE** → covers only consumers from that specific village
- Multiple selections of any type are allowed (e.g. 2 districts + 3 specific villages)
- Each save **replaces** the entire selection

#### Get current coverage areas

```graphql
query {
  myContractorCoverageAreas {
    id
    selectionType
    referenceCode
    displayName
    isActive
    createdAt
  }
}
```

> **Role:** `CONTRACTOR`  
> Returns the contractor's current selections. Empty array = not set yet.

---

#### Set / update coverage areas

```graphql
mutation UpdateCoverageAreas($input: UpdateCoverageAreasInput!) {
  updateMyContractorCoverageAreas(input: $input) {
    success
    message
  }
}
```

**Input:**

```json
{
  "input": {
    "selections": [
      {
        "selectionType": "DISTRICT",
        "referenceCode": "572",
        "displayName": "Bengaluru Urban, Karnataka"
      },
      {
        "selectionType": "SUB_DISTRICT",
        "referenceCode": "5720001",
        "displayName": "Yelahanka, Bengaluru Urban"
      },
      {
        "selectionType": "VILLAGE",
        "referenceCode": "<village-uuid>",
        "displayName": "Vidyaranyapura, Yelahanka, Bengaluru Urban"
      }
    ]
  }
}
```

> **Role:** `CONTRACTOR` — uses JWT, contractor can only update their own  
> **PREMIUM required** — throws `FORBIDDEN` if subscription is FREE  
> `referenceCode` for DISTRICT = `districtCode` from `marketplaceDistricts`; for SUB_DISTRICT = `subDistrictCode` from `marketplaceSubDistricts`; for VILLAGE = village `id` UUID from `marketplaceVillages`  
> `displayName` is a human-readable label stored for display — build it as `"<name>, <parent name>"`  
> Send **all** desired selections in one call — **replaces** entirely

> ⚠️ Until coverage areas are set, the contractor will not appear in any `searchMarketplaceContractors` result.

---

### Contractor Self-Service: Select Services

After Admin creates the profile, the **contractor selects which services they offer** from the Admin-created service catalogue. Works for both FREE and PREMIUM.

#### Step 1 — Fetch all available services (browse what Admin has created)

```graphql
query {
  marketplaceServicesByCategory(areaType: URBAN) {
    category {
      id
      name
      code
    }
    services {
      id
      name
      description
      isFreeAvailable
      uom {
        name
        requiresQuantity
      }
    }
  }
}
```

> Use any `areaType` here — just to browse the catalogue. The `id` is what you need.

#### Step 2 — Save selected services

```graphql
mutation {
  updateMyContractorServices(
    serviceIds: ["<service-uuid-1>", "<service-uuid-2>"]
  ) {
    success
    message
  }
}
```

> **Role:** `CONTRACTOR` (uses JWT — contractor can only update their own).  
> **Replaces** the full selection — send all desired service IDs each time.  
> Validates that every ID is an active service. Invalid IDs → `BAD_REQUEST`.  
> Contractor will only appear in consumer search for services they have selected **AND** that have pricing configured for their area **AND** they are PREMIUM with coverage areas set.

---

### Step 1 — View Incoming Jobs

```graphql
query MyContractorJobs($filters: JobsFilterInput) {
  myContractorMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      status
      consumerId
      consumerName
      consumerPhone
      consumerAddress
      serviceId
      serviceName
      locationId
      locationName
      priceSnapshot
      areaTypeSnapshot
      acceptanceDeadline
      startDeadline
      completionDeadline
      createdAt
      paidAt
      acceptedAt
      startedAt
    }
    pagination {
      total
      page
      limit
      totalPages
      hasNext
      hasPrev
    }
  }
}
```

**Filter only pending (action required):**

```json
{ "filters": { "status": "REQUESTED", "page": 1, "limit": 20 } }
```

> `acceptanceDeadline` — if this expires while job is still `REQUESTED`, it becomes `SLA_BREACH`. Show a countdown timer.

---

### Step 2a — Accept Job

```graphql
mutation AcceptJob($input: AcceptJobInput!) {
  acceptMarketplaceJob(input: $input) {
    id
    jobNumber
    status
    acceptedAt
    startDeadline
    completionDeadline
  }
}
```

**Input:**

```json
{ "input": { "jobId": "<job-uuid>" } }
```

**Validations (backend):**

- Job must be in `REQUESTED` status
- Job must be assigned to this contractor
- `acceptanceDeadline` must not have passed

**What happens:** Job → `ACCEPTED`. `startDeadline` = now + SLA.jobStartHours.

---

### Step 2b — Reject Job

```graphql
mutation RejectJob($input: RejectJobInput!) {
  rejectMarketplaceJob(input: $input) {
    id
    jobNumber
    status
    rejectedAt
    rejectionReason
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "reason": "Not available on the requested date"
  }
}
```

> `reason` is required.  
> Job → `REJECTED`. Auto-refund to consumer will be triggered (TODO: when Cashfree integrated).

---

### Step 3 — Enter Start OTP _(ask consumer for the code when you arrive)_

When the contractor **accepts** the job, the backend **automatically generates a START OTP** and:

- Sends it to the consumer via SMS
- Makes it visible in the consumer's app

The consumer will have this code ready. Contractor arrives at site, asks the consumer for the code, and enters it:

```graphql
mutation VerifyStartOtp($input: VerifyMarketplaceOtpInput!) {
  verifyMarketplaceOtp(input: $input) {
    id
    jobNumber
    status
    startedAt
    completionDeadline
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "code": "123456"
  }
}
```

> Job must be in `ACCEPTED` status.  
> Wrong code → `BadRequestException: Invalid OTP`.  
> After 3 failed attempts → `BadRequestException: Max OTP attempts reached` → use Resend OTP below.  
> On success: Job → `STARTED`. `completionDeadline` calculated. Backend **auto-generates COMPLETE OTP** and sends it to consumer.

---

### Step 4 — Enter Completion OTP _(ask consumer for the code when work is done)_

When the job transitions to `STARTED`, the backend **automatically generates a COMPLETE OTP** and:

- Sends it to the consumer via SMS
- Makes it visible in the consumer's app

Contractor finishes work, asks consumer for the code, and enters it:

```graphql
mutation VerifyCompleteOtp($input: VerifyMarketplaceOtpInput!) {
  verifyMarketplaceOtp(input: $input) {
    id
    jobNumber
    status
    completedAt
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "code": "654321"
  }
}
```

> Job must be in `STARTED` status.  
> On success: Job → `COMPLETED`. Contractor's `totalJobsCompleted` count incremented.

---

### Resend OTP _(fallback — only if consumer didn't receive SMS or OTP expired)_

Under normal flow this is **never needed**. Use only if the consumer lost the OTP.

```graphql
mutation ResendOtp($input: GenerateOtpInput!) {
  generateMarketplaceOtp(input: $input) {
    id
    jobId
    otpType
    code
    expiresAt
    createdAt
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "otpType": "START_JOB" // or "COMPLETE_JOB"
  }
}
```

> Generates a fresh OTP valid for **15 minutes**, sends new SMS to consumer.  
> Previous OTP is superseded (the latest active OTP is always used for verification).

---

### Step 5 — Rate Consumer _(optional, after COMPLETED)_

```graphql
mutation RateConsumer($input: CreateRatingInput!) {
  createMarketplaceRating(input: $input) {
    id
    jobId
    rating
    comment
    createdAt
  }
}
```

**Input:**

```json
{
  "input": {
    "jobId": "<job-uuid>",
    "rating": 4,
    "comment": "Cooperative consumer, easy to work with."
  }
}
```

---

### Subscription Management

#### View Plans

```graphql
query SubscriptionPlans {
  subscriptionPlans {
    duration
    durationMonths
    price
    discountPercent
    finalPrice
    description
  }
}
```

| Plan      | Months | Price  | Discount | Final  |
| --------- | ------ | ------ | -------- | ------ |
| MONTHLY   | 1      | ₹499   | 0%       | ₹499   |
| QUARTERLY | 3      | ₹1,497 | 10%      | ₹1,347 |
| YEARLY    | 12     | ₹5,988 | 25%      | ₹4,491 |

---

#### Check Current Status

```graphql
query MySubscription {
  mySubscriptionStatus {
    currentType
    validTill
    isActive
    daysRemaining
    canUpgrade
    canRenew
  }
}
```

---

#### Initiate Upgrade

```graphql
mutation InitiateUpgrade($input: InitiateSubscriptionUpgradeInput!) {
  initiateSubscriptionUpgrade(input: $input) {
    success
    message
    paymentIntent {
      id
      amount
      duration
      status
      paymentLink
      orderId
      expiresAt
    }
  }
}
```

}

````
**Input:**
```json
{
  "input": {
    "duration": "MONTHLY",
    "bankAccount": {
      "accountNumber": "123456789",
      "ifscCode": "SBIN0001234",
      "accountHolderName": "John Doe"
    }
  }
}
````

**Save:** `paymentIntent.id` — needed for the confirm step.

> `paymentLink` is currently a mock URL (Cashfree not yet integrated).  
> Intent expires in **30 minutes** — if expired, initiate again.  
> Blocked if already PREMIUM with >30 days remaining.

---

#### Confirm Payment

```graphql
mutation ConfirmSubscription($input: ConfirmSubscriptionPaymentInput!) {
  confirmSubscriptionPayment(input: $input) {
    success
    message
    newSubscriptionType
    validTill
  }
}
```

**Input:**

```json
{
  "input": {
    "paymentIntentId": "<paymentIntent.id from above>",
    "paymentReference": "<cashfree-order-id>"
  }
}
```

> `paymentReference` is optional — will be the Cashfree order ID when integrated.

---

#### ⚡ DEV Flow (Cashfree not yet integrated)

Since the payment gateway is not live yet, use this 2-step flow to upgrade without real payment:

**Step 1 — Initiate:**

```graphql
mutation {
  initiateSubscriptionUpgrade(
    input: { contractorId: "<contractor-uuid>", duration: MONTHLY }
  ) {
    paymentIntent {
      id # ← save this
      amount
      expiresAt
    }
  }
}
```

**Step 2 — Confirm immediately** (skip the payment URL entirely):

```graphql
mutation {
  confirmSubscriptionPayment(input: { paymentIntentId: "<id from step 1>" }) {
    success
    newSubscriptionType
    validTill
  }
}
```

> ⚠️ In production this confirm step will be called **only by the Cashfree webhook** after verifying payment signature — never by the app directly.

---

#### View Subscription History

```graphql
query SubHistory {
  mySubscriptionHistory {
    id
    subscriptionType
    startDate
    endDate
    paymentId
    isActive
    createdAt
  }
}
```

---

#### ⚠️ Production TODOs (Subscription)

| #   | Issue                                                     | Where                                                              | What needs to be done                                                                                                                   |
| --- | --------------------------------------------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Payment intents stored **in-memory**                      | `subscriptions.service.ts` — `paymentIntents Map`                  | Move to a DB table or Redis so intents survive server restarts                                                                          |
| 2   | `paymentLink` is a **mock URL**                           | `initiateUpgrade()` — hardcoded `https://payments.example.com/...` | Replace with real Cashfree Order API call to get a live checkout URL                                                                    |
| 3   | `confirmSubscriptionPayment` callable by **app directly** | `subscriptions.resolver.ts`                                        | In production, confirmation must only happen via Cashfree **webhook** (after verifying signature) — never trust the app to self-confirm |

---

## 4. Shared APIs

### Get Job by ID (works for both consumer and contractor)

```graphql
query GetJob($id: ID!) {
  marketplaceJob(id: $id) {
    id
    jobNumber
    status
    consumerName
    consumerPhone
    contractorName
    contractorPhone
    serviceName
    locationName
    consumerAddress
    priceSnapshot
    areaTypeSnapshot
    acceptanceDeadline
    startDeadline
    completionDeadline
    hasRated
    createdAt
    paidAt
    acceptedAt
    rejectedAt
    startedAt
    completedAt
    closedAt
    rejectionReason
    updatedAt
  }
}
```

---

### Rating Summary for any user

```graphql
query RatingSummary($userId: ID!) {
  marketplaceRatingSummary(userId: $userId) {
    averageRating
    totalRatings
    fiveStarCount
    fourStarCount
    threeStarCount
    twoStarCount
    oneStarCount
  }
}
```

---

## 5. Enums Reference

### MarketplaceJobStatus

| Value             | Meaning                                        |
| ----------------- | ---------------------------------------------- |
| `PAYMENT_PENDING` | Job created, payment not done yet              |
| `REQUESTED`       | Payment done, waiting for contractor to accept |
| `ACCEPTED`        | Contractor accepted, waiting for job to start  |
| `REJECTED`        | Contractor rejected → refund triggered         |
| `STARTED`         | Start OTP verified, work in progress           |
| `COMPLETED`       | Completion OTP verified, work done             |
| `CLOSED`          | Consumer rated → fully closed                  |
| `CANCELLED`       | Consumer cancelled                             |
| `SLA_BREACH`      | SLA deadline missed (set by scheduler)         |

### MarketplaceOtpType

| Value          | When to use                                   |
| -------------- | --------------------------------------------- |
| `START_JOB`    | Contractor arrives at site, job is `ACCEPTED` |
| `COMPLETE_JOB` | Work is done, job is `STARTED`                |

### MarketplaceAreaType

| Value        | Meaning                             |
| ------------ | ----------------------------------- |
| `URBAN`      | Urban area — typically higher price |
| `SEMI_URBAN` | Semi-urban area — mid price         |
| `RURAL`      | Rural area — lower price            |

### CoverageSelectionType

| Value          | `referenceCode` field                | Meaning                                                |
| -------------- | ------------------------------------ | ------------------------------------------------------ |
| `DISTRICT`     | `districtCode` (e.g. `"572"`)        | Contractor covers all consumers in the entire district |
| `SUB_DISTRICT` | `subDistrictCode` (e.g. `"5720001"`) | Contractor covers all consumers in that taluk/block    |
| `VILLAGE`      | village `id` UUID                    | Contractor covers only that specific village           |

> **State-level is intentionally unsupported.** Contractors must select at district level or below.

---

### SubscriptionType

| Value     | Meaning                                                           |
| --------- | ----------------------------------------------------------------- |
| `FREE`    | Cannot set coverage areas; does NOT appear in consumer searches   |
| `PREMIUM` | Can set coverage areas and services; appears in consumer searches |

### SubscriptionDuration

| Value       | Months |
| ----------- | ------ |
| `MONTHLY`   | 1      |
| `QUARTERLY` | 3      |
| `YEARLY`    | 12     |

### LoginType

| Value           | Who uses it          |
| --------------- | -------------------- |
| `CONSUMER`      | Consumer app login   |
| `CONTRACTOR`    | Contractor app login |
| `ADMIN`         | Admin web portal     |
| `RETAIL_OUTLET` | RO web portal        |

---

## 6. Error Codes

| HTTP/GraphQL Error                                                    | Message                          | What to do                                        |
| --------------------------------------------------------------------- | -------------------------------- | ------------------------------------------------- |
| `401 Unauthorized`                                                    | Token missing/expired            | Re-login                                          |
| `403 Forbidden`                                                       | Wrong role for this operation    | Check loginType                                   |
| `NOT_FOUND`                                                           | Job/Service/Contractor not found | Invalid ID                                        |
| `BAD_REQUEST: Job cannot be accepted in current status`               | Wrong job state                  | Refresh job and check status                      |
| `BAD_REQUEST: Location not found or not classified`                   | Village not classified yet       | Admin must classify the village                   |
| `BAD_REQUEST: No pricing configured`                                  | No price for service+area        | Admin must set pricing                            |
| `BAD_REQUEST: No SLA configured`                                      | No SLA for service+area          | Admin must set SLA                                |
| `BAD_REQUEST: No valid OTP found`                                     | OTP expired or already used      | Generate a new OTP                                |
| `BAD_REQUEST: Invalid OTP`                                            | Wrong code entered               | Try again (max 3 attempts)                        |
| `BAD_REQUEST: Max OTP attempts reached`                               | 3 wrong attempts                 | Generate a new OTP                                |
| `BAD_REQUEST: Acceptance deadline has passed`                         | Too late to accept               | Job will be marked SLA_BREACH                     |
| `FORBIDDEN: You are not assigned to this job`                         | Wrong contractor                 | Check job assignment                              |
| `FORBIDDEN: Coverage area management requires a PREMIUM subscription` | Contractor is FREE               | Upgrade to PREMIUM first, then set coverage areas |

---

## 7. Where Each ID Comes From

| ID                               | What it is                  | Where you get it                                                          |
| -------------------------------- | --------------------------- | ------------------------------------------------------------------------- |
| `locationId`                     | Village UUID                | `marketplaceVillages` → `id`                                              |
| `serviceId`                      | Service UUID                | `marketplaceServicesByCategory` or `marketplaceServicesWithPrices` → `id` |
| `contractorId`                   | Contractor UUID             | `searchMarketplaceContractors` → `contractorId`                           |
| `jobId`                          | Job UUID                    | `createMarketplaceJob` → `jobId`, or `myMarketplaceJobs` → `id`           |
| `areaType`                       | URBAN/SEMI_URBAN/RURAL      | `marketplaceVillages` → `areaType` (from selected village)                |
| `districtCode` (for coverage)    | District reference code     | `marketplaceDistricts` → `code`                                           |
| `subDistrictCode` (for coverage) | Sub-district reference code | `marketplaceSubDistricts` → `code`                                        |
| `villageId` (for coverage)       | Village UUID                | `marketplaceVillages` → `id`                                              |

---

## 8. Complete Job State Machine

```
createMarketplaceJob
        │
        ▼
  PAYMENT_PENDING
        │
        │ simulatePaymentSuccess (DEV) / Cashfree webhook (PROD)
        ▼
    REQUESTED ──────────────────────────────► CANCELLED (cancelMarketplaceJob)
        │                                           │
        │ acceptMarketplaceJob                      │
        │  └─ [auto: START OTP generated,           │
        │       SMS sent to consumer]               │
        ▼                                           ▼
    ACCEPTED ───────────────────────────────► CANCELLED (cancelMarketplaceJob)
        │                 │
        │                 └── rejectMarketplaceJob ──► REJECTED ──► (refund)
        │ verifyMarketplaceOtp(START_JOB code)
        │  └─ [auto: COMPLETE OTP generated,
        │       SMS sent to consumer]
        ▼
    STARTED
        │
        │ verifyMarketplaceOtp(COMPLETE_JOB code)
        ▼
   COMPLETED
        │
        │ createMarketplaceRating (consumer rates)
        ▼
     CLOSED

Any active state ──► SLA_BREACH (set by backend scheduler if deadlines missed)
```

---

## 9. SLA Deadlines Explained

When a job is created, three deadlines are calculated and **locked forever**:

| Deadline             | Set at             | Formula                           |
| -------------------- | ------------------ | --------------------------------- |
| `acceptanceDeadline` | Job creation       | now + `SLA.acceptanceHours`       |
| `startDeadline`      | Contractor accepts | acceptedAt + `SLA.jobStartHours`  |
| `completionDeadline` | Start OTP verified | startedAt + `SLA.completionHours` |

Show countdown timers in the app for active deadlines. When a deadline is within 20% of total time remaining, show a warning.

---

_This document reflects the actual implemented backend resolvers and services as of 26 February 2026._
