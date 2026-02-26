# Marketplace API Reference

> Complete API documentation matching the implemented backend schema.
> **Last Updated:** 26 February 2026

---

## Table of Contents

- [Consumer APIs](#consumer-apis)
  - [Queries](#consumer-queries)
  - [Mutations](#consumer-mutations)
- [Contractor APIs](#contractor-apis)
  - [Queries](#contractor-queries)
  - [Mutations](#contractor-mutations)
- [Admin APIs](#admin-apis)
  - [Queries](#admin-queries)
  - [Mutations](#admin-mutations)
- [Shared APIs](#shared-apis)
- [Types](#types)
- [Input Types](#input-types)
- [Enums](#enums)
- [Flow Diagrams](#flow-diagrams)

---

## Consumer APIs

### Consumer Queries

#### `myMarketplaceJobs`

Get all jobs for the current logged-in consumer.

```graphql
query MyMarketplaceJobs($filters: JobsFilterInput) {
  myMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      status
      serviceName
      serviceId
      contractorId
      contractorName
      contractorPhone
      consumerId
      consumerName
      consumerPhone
      consumerAddress
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
      completedAt
      closedAt
      rejectedAt
      rejectionReason
      updatedAt
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

**Roles:** `CONSUMER`

---

#### `activeJobOtp`

Get the active (unexpired, unused) OTP for a job. Used by consumers to see the verification code in their UI instead of relying on SMS.

```graphql
query ActiveJobOtp($jobId: ID!) {
  activeJobOtp(jobId: $jobId) {
    id
    jobId
    otpType
    code
    expiresAt
    isUsed
    usedAt
    createdAt
  }
}
```

**Roles:** `CONSUMER`

**Returns:** `JobOtp` or `null` if no active OTP exists.

**Notes:**

- Consumer must own the job (ownership validated server-side)
- Returns the latest unexpired, unused OTP
- Frontend should poll this query when job is in `ACCEPTED` or `STARTED` status
- Code is a 6-digit string
- OTP expires after 15 minutes

---

#### `searchMarketplaceContractors`

Search for contractors by location and service.

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

**Variables:**

```json
{
  "input": {
    "locationId": "village-uuid",
    "serviceId": "service-uuid",
    "minRating": 4,
    "sortBy": "rating",
    "sortOrder": "desc",
    "page": 1,
    "limit": 20
  }
}
```

**Roles:** `CONSUMER`

---

#### `marketplaceServicesWithPrices`

Get all services with current prices for a specific area type.

```graphql
query ServicesWithPrices($areaType: MarketplaceAreaType!) {
  marketplaceServicesWithPrices(areaType: $areaType) {
    id
    name
    description
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
    currentPrice
    isFreeAvailable
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`

---

#### `marketplaceServicesByCategory`

Get services grouped by category with prices for an area type.

```graphql
query ServicesByCategory($areaType: MarketplaceAreaType!) {
  marketplaceServicesByCategory(areaType: $areaType) {
    category {
      id
      name
      code
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

**Roles:** `CONSUMER`, `CONTRACTOR`

---

#### Location Dropdowns

```graphql
# Get all states
query States {
  marketplaceStates {
    stateCode
    stateName
  }
}

# Get districts for a state
query Districts($stateCode: String!) {
  marketplaceDistricts(stateCode: $stateCode) {
    districtCode
    districtName
  }
}

# Get sub-districts for a district
query SubDistricts($districtCode: String!) {
  marketplaceSubDistricts(districtCode: $districtCode) {
    subDistrictCode
    subDistrictName
  }
}

# Get villages (only classified ones with area type)
query Villages($districtCode: String!, $subDistrictCode: String) {
  marketplaceVillages(
    districtCode: $districtCode
    subDistrictCode: $subDistrictCode
  ) {
    id
    villageCode
    villageName
    areaType
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### Service Categories & UOMs

```graphql
# Get all categories
query Categories($activeOnly: Boolean) {
  serviceCategories(activeOnly: $activeOnly) {
    id
    name
    code
    description
    icon
    isFreeAvailable
    displayOrder
    isActive
    createdAt
    updatedAt
  }
}

# Get categories available to FREE contractors
query FreeCategories {
  freeAvailableCategories {
    id
    name
    code
    isFreeAvailable
    isActive
  }
}

# Get all UOMs
query UOMs($activeOnly: Boolean) {
  serviceUoms(activeOnly: $activeOnly) {
    id
    name
    code
    description
    requiresQuantity
    displayOrder
    isActive
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

### Consumer Mutations

#### `createMarketplaceJob`

Create a new marketplace job (initiates payment flow).

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

**Returns:** `CreateJobResult`

**Variables:**

```json
{
  "input": {
    "contractorId": "contractor-uuid",
    "serviceId": "service-uuid",
    "locationId": "village-uuid",
    "quantity": 1,
    "consumerAddress": "123 Main St, Village",
    "consumerPhone": "9876543210"
  }
}
```

**Roles:** `CONSUMER`

**Notes:**

- Job starts as `PAYMENT_PENDING`
- `paymentSessionId` and `paymentUrl` are placeholders until Cashfree is integrated
- `quantity` defaults to 1; use for UOMs with `requiresQuantity: true`
- `amount` = `unitPrice × quantity` (calculated server-side)

---

#### `simulatePaymentSuccess` (DEV ONLY)

Simulate payment gateway success. Will be replaced by Cashfree webhook.

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

**Roles:** Any authenticated user (DEV ONLY)

---

#### `cancelMarketplaceJob`

Cancel a job (before contractor starts).

```graphql
mutation CancelJob($input: CancelJobInput!) {
  cancelMarketplaceJob(input: $input) {
    id
    jobNumber
    status
    rejectionReason
  }
}
```

**Variables:**

```json
{
  "input": {
    "jobId": "job-uuid",
    "reason": "Changed my mind"
  }
}
```

**Roles:** `CONSUMER`

**Rules:**

- Can only cancel jobs in `PAYMENT_PENDING` or `REQUESTED` status
- Cannot cancel once job is `STARTED`

---

#### `createMarketplaceRating`

Rate a contractor after job completion.

```graphql
mutation RateJob($input: CreateRatingInput!) {
  createMarketplaceRating(input: $input) {
    id
    jobId
    raterUserId
    rateeUserId
    rating
    comment
    createdAt
  }
}
```

**Variables:**

```json
{
  "input": {
    "jobId": "job-uuid",
    "rating": 5,
    "comment": "Excellent work, very professional!"
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`

**Rules:**

- Job must be in `COMPLETED` status
- Each party can rate once per job
- Rating is 1-5 stars
- Comment is optional (max 500 chars)
- After consumer rates, job moves to `CLOSED`

---

## Contractor APIs

### Contractor Queries

#### `myContractorMarketplaceJobs`

Get all jobs assigned to the current contractor.

```graphql
query MyContractorJobs($filters: JobsFilterInput) {
  myContractorMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      status
      serviceName
      serviceId
      consumerId
      consumerName
      consumerPhone
      consumerAddress
      contractorId
      contractorName
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
      completedAt
      closedAt
      updatedAt
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

**Roles:** `CONTRACTOR`

---

#### `mySubscriptionStatus`

Get the current subscription status for the logged-in contractor. Returns a graceful FREE default if no marketplace profile row exists yet — never throws 404.

```graphql
query MySubscriptionStatus {
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

**Roles:** `CONTRACTOR`

**Notes:**
- `canUpgrade` → `true` when currently FREE
- `canRenew` → `true` when PREMIUM with ≤ 30 days remaining
- `daysRemaining` → `0` if FREE or expired
- Always returns a valid object (never 404)

---

#### `mySubscriptionHistory`

Get the subscription history for the logged-in contractor.

```graphql
query MySubscriptionHistory {
  mySubscriptionHistory {
    id
    contractorId
    subscriptionType
    startDate
    endDate
    paymentId
    createdAt
  }
}
```

**Roles:** `CONTRACTOR`

---

#### `subscriptionPlans`

Get available subscription plans.

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

**Roles:** Any authenticated user

---

### Contractor Mutations

#### `acceptMarketplaceJob`

Accept an incoming job request.

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

**Variables:**

```json
{
  "input": {
    "jobId": "job-uuid"
  }
}
```

**Roles:** `CONTRACTOR`

**Rules:**

- Job must be in `REQUESTED` status
- Job must be assigned to this contractor
- Sets SLA start deadline after acceptance

---

#### `rejectMarketplaceJob`

Reject an incoming job request.

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

**Variables:**

```json
{
  "input": {
    "jobId": "job-uuid",
    "reason": "Not available on requested date"
  }
}
```

**Roles:** `CONTRACTOR`

**Rules:**

- Job must be in `REQUESTED` status
- Reason is required
- Triggers automatic refund to consumer

---

#### `generateMarketplaceOtp`

Generate OTP for job start or completion. The 6-digit code is returned in the response and also sent to the consumer via SMS. Consumer can also see the code via the `activeJobOtp` query.

```graphql
mutation GenerateOTP($input: GenerateOtpInput!) {
  generateMarketplaceOtp(input: $input) {
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

**Variables:**

```json
{
  "input": {
    "jobId": "job-uuid",
    "otpType": "START_JOB"
  }
}
```

**Roles:** `CONTRACTOR`

**OTP Types:**

- `START_JOB` — Generate when arriving at consumer's location (job must be `ACCEPTED`)
- `COMPLETE_JOB` — Generate when work is done (job must be `STARTED`)

**Notes:**

- OTP is a 6-digit code, valid for 15 minutes
- Consumer sees OTP in their UI via `activeJobOtp` query
- OTP is also sent via SMS to consumer's phone
- Max 3 verification attempts per OTP

---

#### `verifyMarketplaceOtp`

Verify OTP to start or complete job.

```graphql
mutation VerifyOTP($input: VerifyMarketplaceOtpInput!) {
  verifyMarketplaceOtp(input: $input) {
    id
    jobNumber
    status
    startedAt
    completedAt
  }
}
```

**Variables:**

```json
{
  "input": {
    "jobId": "job-uuid",
    "code": "123456"
  }
}
```

**Roles:** `CONTRACTOR`

**Rules:**

- For `START_JOB`: Job must be in `ACCEPTED` status → becomes `STARTED`
- For `COMPLETE_JOB`: Job must be in `STARTED` status → becomes `COMPLETED`
- Max 3 attempts per OTP; after that, generate a new one

---

#### `initiateSubscriptionUpgrade`

Initiate subscription upgrade to PREMIUM. `contractorId` is **never** passed by the client — it is derived automatically from the JWT token.

This mutation also optionally saves bank account details for payout purposes. If the contractor's marketplace profile does not yet exist, it is auto-created as FREE (inactive).

```graphql
mutation InitiateUpgrade($input: InitiateSubscriptionUpgradeInput!) {
  initiateSubscriptionUpgrade(input: $input) {
    success
    message
    paymentIntent {
      id
      contractorId
      duration
      amount
      status
      paymentLink
      orderId
      expiresAt
      createdAt
    }
  }
}
```

**Variables — minimal:**

```json
{
  "input": {
    "duration": "MONTHLY"
  }
}
```

**Variables — with bank account:**

```json
{
  "input": {
    "duration": "QUARTERLY",
    "bankAccount": {
      "accountNumber": "1234567890",
      "ifscCode": "SBIN0001234",
      "accountHolderName": "John Doe",
      "bankName": "State Bank of India"
    }
  }
}
```

**Roles:** `CONTRACTOR`

> ⚠️ **Breaking change from previous version:** `contractorId` has been removed from the input. It was causing a `"property contractorId should not exist"` 400 error. The server now reads it from the JWT session automatically.

**Business Rules:**
- If no marketplace profile exists → auto-creates FREE profile before initiating payment
- If `bankAccount` is provided → saves/updates bank details on the contractor profile
- If already PREMIUM with > 30 days remaining → throws `CONFLICT` (renewal available at ≤ 30 days)
- Payment intent expires in **30 minutes**

---

#### `confirmSubscriptionPayment`

Confirm subscription payment after payment gateway callback. On success, sets `isMarketplaceActive = true` on the contractor's profile, enabling them to appear in marketplace search results.

```graphql
mutation ConfirmPayment($input: ConfirmSubscriptionPaymentInput!) {
  confirmSubscriptionPayment(input: $input) {
    success
    message
    newSubscriptionType
    validTill
  }
}
```

**Variables:**

```json
{
  "input": {
    "paymentIntentId": "payment-intent-uuid",
    "paymentReference": "cashfree-ref-123"
  }
}
```

**Roles:** `CONTRACTOR`

**Side effects:**
- Sets `isMarketplaceActive = true` → contractor becomes visible in consumer search
- Adds row to subscription history
- Payment intent status → `SUCCESS`

---

## Admin APIs

### Admin Queries

#### `marketplaceJobs`

Get all jobs with filters.

```graphql
query AllJobs($filters: JobsFilterInput) {
  marketplaceJobs(filters: $filters) {
    items {
      ...MarketplaceJobFields
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

**Roles:** `ADMIN`, `SUPER_ADMIN`

---

#### `marketplaceSlaBreaches`

Get SLA breaches with filters.

```graphql
query SlaBreaches($filters: SlaBreachFilterInput) {
  marketplaceSlaBreaches(filters: $filters) {
    items {
      id
      jobId
      jobNumber
      breachType
      deadline
      breachedAt
      createdAt
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

**Roles:** `ADMIN`, `SUPER_ADMIN`

---

#### `contractorSubscriptionStatus` / `contractorSubscriptionHistory`

Admin queries for checking any contractor's subscription.

```graphql
query ContractorSubStatus($contractorId: ID!) {
  contractorSubscriptionStatus(contractorId: $contractorId) {
    currentType
    validTill
    isActive
    daysRemaining
    canUpgrade
    canRenew
  }
}

query ContractorSubHistory($contractorId: ID!) {
  contractorSubscriptionHistory(contractorId: $contractorId) {
    id
    contractorId
    subscriptionType
    startDate
    endDate
    paymentId
    isActive
    createdAt
  }
}
```

**Roles:** `ADMIN`, `SUPER_ADMIN`

---

### Admin Mutations

#### `adminSetContractorSubscription`

Set a contractor's subscription directly (bypass payment).

```graphql
mutation AdminSetSub($input: AdminSetSubscriptionInput!) {
  adminSetContractorSubscription(input: $input) {
    success
    message
    subscription {
      id
      contractorId
      subscriptionType
      startDate
      endDate
    }
  }
}
```

**Roles:** `ADMIN`, `SUPER_ADMIN`

---

## Shared APIs

These APIs are available to multiple roles.

#### `marketplaceJob`

Get a single job by ID.

```graphql
query GetJob($id: ID!) {
  marketplaceJob(id: $id) {
    id
    jobNumber
    status
    serviceName
    serviceId
    consumerId
    consumerName
    consumerPhone
    consumerAddress
    contractorId
    contractorName
    contractorPhone
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
    rejectedAt
    startedAt
    completedAt
    closedAt
    rejectionReason
    updatedAt
  }
}
```

**Roles:** Any authenticated user

---

#### `marketplaceContractorProfile`

Get a contractor's marketplace profile.

```graphql
query ContractorProfile($contractorId: ID!) {
  marketplaceContractorProfile(contractorId: $contractorId) {
    id
    contractorId
    contractorName
    isMarketplaceActive
    subscriptionType
    subscriptionValidTill
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceContractorLocations` / `marketplaceContractorServices`

```graphql
query ContractorLocations($contractorId: ID!) {
  marketplaceContractorLocations(contractorId: $contractorId) {
    id
    contractorId
    locationId
    assignedAt
  }
}

query ContractorServices($contractorId: ID!) {
  marketplaceContractorServices(contractorId: $contractorId) {
    id
    contractorId
    serviceId
    assignedAt
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceContractorReviews`

Get reviews for a contractor.

```graphql
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
    pagination {
      total
      page
      limit
      totalPages
      hasNext
      hasPrev
    }
    summary {
      averageRating
      totalRatings
      oneStarCount
      twoStarCount
      threeStarCount
      fourStarCount
      fiveStarCount
    }
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceRatingSummary`

Get rating summary for a user.

```graphql
query RatingSummary($userId: ID!) {
  marketplaceRatingSummary(userId: $userId) {
    averageRating
    totalRatings
    oneStarCount
    twoStarCount
    threeStarCount
    fourStarCount
    fiveStarCount
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplacePaymentByJob`

Get payment details for a job.

```graphql
query PaymentByJob($jobId: ID!) {
  marketplacePaymentByJob(jobId: $jobId) {
    id
    jobId
    amount
    status
    createdAt
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

## Types

### MarketplaceJob

```graphql
type MarketplaceJob {
  id: ID!
  jobNumber: String!
  status: MarketplaceJobStatus!
  consumerId: ID!
  consumerName: String!
  consumerPhone: String
  consumerAddress: String
  contractorId: ID!
  contractorName: String!
  contractorPhone: String
  serviceId: ID!
  serviceName: String!
  locationId: ID!
  locationName: String!
  priceSnapshot: Float!
  areaTypeSnapshot: MarketplaceAreaType!
  acceptanceDeadline: DateTime!
  startDeadline: DateTime
  completionDeadline: DateTime
  createdAt: DateTime!
  paidAt: DateTime
  acceptedAt: DateTime
  rejectedAt: DateTime
  startedAt: DateTime
  completedAt: DateTime
  closedAt: DateTime
  rejectionReason: String
  updatedAt: DateTime!
}
```

### JobOtp

```graphql
type JobOtp {
  id: ID!
  jobId: ID!
  otpType: MarketplaceOtpType!
  code: String!
  expiresAt: DateTime!
  isUsed: Boolean!
  usedAt: DateTime
  createdAt: DateTime!
}
```

### SubscriptionStatus

```graphql
type SubscriptionStatus {
  currentType: SubscriptionType!   # FREE | PREMIUM
  validTill: DateTime              # null if FREE
  isActive: Boolean!               # true if PREMIUM and not expired
  daysRemaining: Int!              # 0 if FREE or expired
  canUpgrade: Boolean!             # true if currently FREE
  canRenew: Boolean!               # true if PREMIUM with ≤30 days left
}
```

### SubscriptionUpgradeResponse

```graphql
type SubscriptionUpgradeResponse {
  success: Boolean!
  message: String!
  paymentIntent: SubscriptionPaymentIntent
}
```

### SubscriptionPaymentIntent

```graphql
type SubscriptionPaymentIntent {
  id: ID!
  contractorId: ID!
  duration: SubscriptionDuration!
  amount: Float!
  status: SubscriptionPaymentStatus!
  paymentLink: String
  orderId: String
  expiresAt: DateTime!
  createdAt: DateTime!
}
```

### SubscriptionConfirmResponse

```graphql
type SubscriptionConfirmResponse {
  success: Boolean!
  message: String!
  newSubscriptionType: SubscriptionType
  validTill: DateTime
}
```

### CreateJobResult

```graphql
type CreateJobResult {
  jobId: ID!
  jobNumber: String!
  amount: Float!
  paymentSessionId: String! # Cashfree placeholder
  paymentUrl: String # Cashfree placeholder
}
```

### ContractorReview

```graphql
type ContractorReview {
  id: ID!
  consumerName: String!
  rating: Int!
  comment: String
  serviceName: String!
  createdAt: DateTime!
}
```

### ContractorSearchResult

```graphql
type ContractorSearchResult {
  contractorId: ID!
  contractorName: String!
  companyName: String
  phone: String!
  ratingAvg: Float!
  totalRatings: Int!
  totalJobsCompleted: Int!
  isPremiumContractor: Boolean!
  priceForService: Float!
}
```

---

## Input Types

### CreateJobInput

```graphql
input CreateJobInput {
  contractorId: ID! # Selected contractor
  serviceId: ID! # Selected service
  locationId: ID! # Consumer's village/location
  quantity: Int = 1 # Quantity (for UOMs with requiresQuantity)
  consumerAddress: String # Service address (optional)
  consumerPhone: String # Contact phone (optional)
}
```

### AcceptJobInput

```graphql
input AcceptJobInput {
  jobId: ID!
}
```

### RejectJobInput

```graphql
input RejectJobInput {
  jobId: ID!
  reason: String!
}
```

### GenerateOtpInput

```graphql
input GenerateOtpInput {
  jobId: ID!
  otpType: MarketplaceOtpType!
}
```

### VerifyMarketplaceOtpInput

```graphql
input VerifyMarketplaceOtpInput {
  jobId: ID!
  code: String! # 6-digit OTP code
}
```

### CancelJobInput

```graphql
input CancelJobInput {
  jobId: ID!
  reason: String # Optional
}
```

### CreateRatingInput

```graphql
input CreateRatingInput {
  jobId: ID!
  rating: Int! # 1-5 stars
  comment: String # Max 500 chars
}
```

### BankAccountInput

```graphql
input BankAccountInput {
  accountNumber: String!       # Max 30 chars
  ifscCode: String!            # Format: SBIN0001234 (regex validated)
  accountHolderName: String!   # Max 255 chars
  bankName: String             # Optional, max 255 chars
}
```

### InitiateSubscriptionUpgradeInput

```graphql
input InitiateSubscriptionUpgradeInput {
  duration: SubscriptionDuration!  # MONTHLY | QUARTERLY | YEARLY
  bankAccount: BankAccountInput    # Optional — saves/updates payout account
  # contractorId is NOT a client field — injected from JWT by the server
}
```

### ConfirmSubscriptionPaymentInput

```graphql
input ConfirmSubscriptionPaymentInput {
  paymentIntentId: ID!     # UUID from initiateSubscriptionUpgrade response
  paymentReference: String # Optional payment gateway reference
}
```

### AdminSetSubscriptionInput

```graphql
input AdminSetSubscriptionInput {
  contractorId: ID!              # Target contractor
  subscriptionType: SubscriptionType!  # FREE | PREMIUM
  durationMonths: Int            # Required for PREMIUM
}
```

### ContractorSearchInput

```graphql
input ContractorSearchInput {
  locationId: ID! # Village ID (required)
  serviceId: ID! # Service ID (required)
  minRating: Int # Minimum rating filter
  sortBy: String # rating | price | completed_jobs
  sortOrder: String # asc | desc
  page: Int = 1
  limit: Int = 20
}
```

### JobsFilterInput

```graphql
input JobsFilterInput {
  status: MarketplaceJobStatus
  consumerId: ID
  contractorId: ID
  serviceId: ID
  locationId: ID
  search: String
  dateFrom: DateTime
  dateTo: DateTime
  page: Int = 1
  limit: Int = 20
}
```

---

## Enums

### MarketplaceJobStatus

| Value             | Description                              |
| ----------------- | ---------------------------------------- |
| `PAYMENT_PENDING` | Job created, awaiting payment            |
| `REQUESTED`       | Payment complete, waiting for contractor |
| `ACCEPTED`        | Contractor accepted the job              |
| `REJECTED`        | Contractor rejected the job              |
| `STARTED`         | Job in progress (start OTP verified)     |
| `COMPLETED`       | Job completed (completion OTP verified)  |
| `CLOSED`          | Rating given, job fully closed           |
| `CANCELLED`       | Consumer cancelled the job               |
| `REFUNDED`        | Refund processed                         |
| `SLA_BREACH`      | SLA deadline breached                    |

### MarketplaceOtpType

| Value          | Description              |
| -------------- | ------------------------ |
| `START_JOB`    | OTP to start the job     |
| `COMPLETE_JOB` | OTP to mark job complete |

### MarketplaceAreaType

| Value        | Description     |
| ------------ | --------------- |
| `URBAN`      | Urban area      |
| `SEMI_URBAN` | Semi-urban area |
| `RURAL`      | Rural area      |

### SubscriptionType

| Value     | Description                    |
| --------- | ------------------------------ |
| `FREE`    | Free tier (limited categories) |
| `PREMIUM` | Premium tier (all categories)  |

### SubscriptionDuration

| Value       | Description                   |
| ----------- | ----------------------------- |
| `MONTHLY`   | 1 month (₹499)                |
| `QUARTERLY` | 3 months (₹1,347 — 10% off)  |
| `YEARLY`    | 12 months (₹4,491 — 25% off) |

### SubscriptionPaymentStatus

| Value     | Description                          |
| --------- | ------------------------------------ |
| `PENDING` | Payment intent created, awaiting pay |
| `SUCCESS` | Payment confirmed                    |
| `FAILED`  | Payment failed                       |
| `EXPIRED` | Payment intent expired (30 min TTL)  |

### PaymentStatus (Job Payments)

| Value      | Description        |
| ---------- | ------------------ |
| `PENDING`  | Payment initiated  |
| `SUCCESS`  | Payment successful |
| `FAILED`   | Payment failed     |
| `REFUNDED` | Payment refunded   |

### RefundStatus / RefundReason

| RefundStatus | Description            |
| ------------ | ---------------------- |
| `PENDING`    | Refund requested       |
| `PROCESSING` | Refund being processed |
| `SUCCESS`    | Refund completed       |
| `FAILED`     | Refund failed          |

| RefundReason          | Description                 |
| --------------------- | --------------------------- |
| `CONTRACTOR_REJECTED` | Contractor rejected the job |
| `CONSUMER_CANCELLED`  | Consumer cancelled          |
| `SLA_BREACH`          | SLA was breached            |
| `ADMIN_CANCELLED`     | Admin cancelled             |
| `OTHER`               | Other reason                |

---

## Flow Diagrams

### Consumer Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONSUMER FLOW                            │
└─────────────────────────────────────────────────────────────────┘

1. SELECT LOCATION (Village)
   ├── marketplaceStates
   ├── marketplaceDistricts(stateCode)
   ├── marketplaceSubDistricts(districtCode)
   └── marketplaceVillages(districtCode, subDistrictCode?)
       └── Returns areaType (URBAN/SEMI_URBAN/RURAL)

2. BROWSE SERVICES (priced by areaType)
   ├── marketplaceServicesByCategory(areaType)
   └── marketplaceServicesWithPrices(areaType)

3. SEARCH CONTRACTORS
   └── searchMarketplaceContractors({ serviceId, locationId })
       ├── View Profile: marketplaceContractorProfile(contractorId)
       ├── View Areas: marketplaceContractorLocations(contractorId)
       └── View Reviews: marketplaceContractorReviews(contractorId)

4. CREATE JOB
   └── createMarketplaceJob({ contractorId, serviceId, locationId, quantity })
       └── Returns jobId + paymentSessionId

5. PAYMENT
   └── simulatePaymentSuccess(jobId)  [DEV ONLY]
       └── Job status → REQUESTED

6. TRACK JOB
   └── myMarketplaceJobs / marketplaceJob(id)

7. VIEW OTP (when contractor generates it)
   └── activeJobOtp(jobId)
       └── Returns 6-digit code to show on screen

8. CANCEL (Optional — before STARTED)
   └── cancelMarketplaceJob({ jobId, reason })

9. RATE CONTRACTOR
   └── createMarketplaceRating({ jobId, rating, comment })
       └── Job status → CLOSED
```

### Contractor Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                       CONTRACTOR FLOW                           │
└─────────────────────────────────────────────────────────────────┘

1. VIEW INCOMING JOBS
   └── myContractorMarketplaceJobs(status: REQUESTED)

2. ACCEPT OR REJECT
   ├── acceptMarketplaceJob({ jobId })
   │   └── Job → ACCEPTED, SLA deadlines set
   └── rejectMarketplaceJob({ jobId, reason })
       └── Job → REJECTED, auto-refund triggered

3. START JOB (At consumer location)
   ├── generateMarketplaceOtp({ jobId, otpType: START_JOB })
   │   └── 6-digit code returned + sent to consumer
   │   └── Consumer sees code via activeJobOtp(jobId)
   └── verifyMarketplaceOtp({ jobId, code })
       └── Job → STARTED

4. COMPLETE JOB (After work done)
   ├── generateMarketplaceOtp({ jobId, otpType: COMPLETE_JOB })
   │   └── 6-digit code returned + sent to consumer
   └── verifyMarketplaceOtp({ jobId, code })
       └── Job → COMPLETED

5. SUBSCRIPTION UPGRADE
   ├── subscriptionPlans → View plans
   ├── mySubscriptionStatus → Check current plan (no 404 — returns FREE default if no profile)
   ├── initiateSubscriptionUpgrade({ duration, bankAccount? })
   │   └── contractorId is auto-injected from JWT — DO NOT pass it
   │   └── Auto-creates FREE marketplace profile if missing
   └── confirmSubscriptionPayment({ paymentIntentId, paymentReference? })
       └── Sets isMarketplaceActive=true → contractor appears in search

6. RATE CONSUMER (Optional)
   └── createMarketplaceRating({ jobId, rating, comment })
```

### Job Status Flow

```
                    ┌─────────────────┐
                    │ PAYMENT_PENDING │ ← createMarketplaceJob
                    └────────┬────────┘
                             │ Payment Success (webhook/simulate)
                             ▼
                    ┌─────────────────┐
         ┌─────────│    REQUESTED    │─────────┐
         │         └────────┬────────┘         │
         │                  │                  │
    rejectJob          acceptJob          cancelJob
         │                  │                  │
         ▼                  ▼                  ▼
   ┌──────────┐      ┌──────────┐       ┌───────────┐
   │ REJECTED │      │ ACCEPTED │       │ CANCELLED │
   └────┬─────┘      └────┬─────┘       └─────┬─────┘
        │                 │                   │
        │          verifyOtp(START)           │
        │                 │                   │
        │                 ▼                   │
        │           ┌─────────┐               │
        │           │ STARTED │               │
        │           └────┬────┘               │
        │                │                    │
        │         verifyOtp(COMPLETE)         │
        │                │                    │
        │                ▼                    │
        │          ┌───────────┐              │
        │          │ COMPLETED │              │
        │          └─────┬─────┘              │
        │                │                    │
        │           createRating              │
        │                │                    │
        │                ▼                    │
        │           ┌────────┐                │
        │           │ CLOSED │                │
        │           └────────┘                │
        │                                     │
        └──────────────┬──────────────────────┘
                       │ Auto-refund
                       ▼
                  ┌──────────┐
                  │ REFUNDED │
                  └──────────┘
```

---

## Error Codes

| Code          | Message                          | Cause                            |
| ------------- | -------------------------------- | -------------------------------- |
| `NOT_FOUND`   | Job/Service/Contractor not found | Invalid ID                       |
| `BAD_REQUEST` | Invalid status for this action   | Wrong job state for operation    |
| `FORBIDDEN`   | Not authorized for this job      | Job doesn't belong to user       |
| `BAD_REQUEST` | No valid OTP found               | OTP expired or already used      |
| `BAD_REQUEST` | Invalid OTP                      | Wrong code entered               |
| `BAD_REQUEST` | Max OTP attempts reached         | 3+ failed attempts               |
| `BAD_REQUEST` | No pricing configured            | Missing pricing for service+area |
| `BAD_REQUEST` | Location not classified          | Admin hasn't set area type       |

---

## Rate Limits

| Endpoint Type  | Limit               |
| -------------- | ------------------- |
| Queries        | 100 requests/minute |
| Mutations      | 30 requests/minute  |
| OTP Generation | 5 requests/job/hour |

---

_Last Updated: 26 February 2026_
