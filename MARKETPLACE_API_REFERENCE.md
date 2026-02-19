# Marketplace API Reference

> Complete API documentation for Consumer and Contractor marketplace integration.

---

## Table of Contents

- [Consumer APIs](#consumer-apis)
  - [Queries](#consumer-queries)
  - [Mutations](#consumer-mutations)
- [Contractor APIs](#contractor-apis)
  - [Queries](#contractor-queries)
  - [Mutations](#contractor-mutations)
- [Shared APIs](#shared-apis)
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
      contractorName
      totalAmount
      createdAt
    }
    pagination {
      total
      page
      limit
      totalPages
    }
  }
}
```

**Roles:** `CONSUMER`

---

#### `searchMarketplaceContractors`
Search for contractors by location and service.

```graphql
query SearchContractors($input: ContractorSearchInput!) {
  searchMarketplaceContractors(input: $input) {
    items {
      id
      contractorId
      contractorName
      phone
      averageRating
      totalCompletedJobs
      subscriptionType
      price
      currency
    }
    pagination {
      total
      page
      limit
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
    categoryName
    uomName
    price
    currency
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`

---

#### `marketplaceServicesByCategory`
Get services grouped by category.

```graphql
query ServicesByCategory($areaType: MarketplaceAreaType) {
  marketplaceServicesByCategory(areaType: $areaType) {
    categoryId
    categoryName
    categoryCode
    services {
      id
      name
      description
      price
      uomName
    }
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`

---

#### `marketplaceContractorProfile`
Get a contractor's marketplace profile.

```graphql
query ContractorProfile($contractorId: ID!) {
  marketplaceContractorProfile(contractorId: $contractorId) {
    id
    contractorId
    contractorName
    phone
    email
    subscriptionType
    averageRating
    totalCompletedJobs
    totalReviews
    isAvailable
    createdAt
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceContractorLocations`
Get locations a contractor serves.

```graphql
query ContractorLocations($contractorId: ID!) {
  marketplaceContractorLocations(contractorId: $contractorId) {
    id
    locationId
    stateName
    districtName
    subDistrictName
    villageName
    areaType
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceContractorServices`
Get services a contractor offers.

```graphql
query ContractorServices($contractorId: ID!) {
  marketplaceContractorServices(contractorId: $contractorId) {
    id
    serviceId
    serviceName
    categoryName
    uomName
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceContractorReviews`
Get reviews for a contractor.

```graphql
query ContractorReviews($contractorId: ID!, $page: Int, $limit: Int) {
  marketplaceContractorReviews(contractorId: $contractorId, page: $page, limit: $limit) {
    items {
      id
      jobNumber
      rating
      comment
      raterName
      createdAt
    }
    pagination {
      total
      page
      limit
    }
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

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

# Get villages for a sub-district (only classified ones)
query Villages($subDistrictCode: String!) {
  marketplaceVillages(subDistrictCode: $subDistrictCode) {
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
query Categories {
  serviceCategories {
    id
    name
    code
    description
    icon
    isFreeAvailable
    displayOrder
  }
}

# Get categories available to FREE contractors
query FreeCategories {
  serviceCategoriesFreeAvailable {
    id
    name
    code
  }
}

# Get all UOMs
query UOMs {
  serviceUoms {
    id
    name
    code
    description
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
    success
    message
    job {
      id
      jobNumber
      status
      totalAmount
      currency
    }
    paymentSession {
      paymentId
      sessionId
      paymentLink
      amount
    }
  }
}
```

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
- Returns a payment session for redirect to payment gateway
- Job status starts as `PAYMENT_PENDING`
- After payment success, status changes to `REQUESTED`

---

#### `cancelMarketplaceJob`
Cancel a job (before contractor starts).

```graphql
mutation CancelJob($input: CancelJobInput!) {
  cancelMarketplaceJob(input: $input) {
    success
    message
    job {
      id
      jobNumber
      status
    }
    refund {
      id
      amount
      status
    }
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
- Can only cancel jobs in `REQUESTED` or `ACCEPTED` status
- Cannot cancel once job is `STARTED`
- Triggers automatic refund

---

#### `initiateMarketplacePayment`
Retry/initiate payment for a pending job.

```graphql
mutation InitiatePayment($jobId: ID!) {
  initiateMarketplacePayment(jobId: $jobId) {
    paymentId
    sessionId
    paymentLink
    amount
    currency
    expiresAt
  }
}
```

**Roles:** `CONSUMER`

---

#### `createMarketplaceRating`
Rate a contractor after job completion.

```graphql
mutation RateJob($input: CreateRatingInput!) {
  createMarketplaceRating(input: $input) {
    id
    jobId
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
      consumerName
      consumerPhone
      consumerAddress
      totalAmount
      locationName
      acceptanceDeadline
      completionDeadline
      createdAt
    }
    pagination {
      total
      page
      limit
      totalPages
    }
  }
}
```

**Roles:** `CONTRACTOR`

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
    jobStartDeadline
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
- Sets SLA deadlines after acceptance

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
Generate OTP for job start or completion.

```graphql
mutation GenerateOTP($input: GenerateOtpInput!) {
  generateMarketplaceOtp(input: $input) {
    jobId
    otpType
    expiresAt
    message
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
- `START_JOB` - Generate when arriving at location
- `COMPLETE_JOB` - Generate when work is done

**Notes:**
- OTP is sent to consumer's phone via SMS
- OTP expires in 10 minutes
- Consumer must share OTP with contractor

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
- For START_JOB: Job must be in `ACCEPTED` status → becomes `STARTED`
- For COMPLETE_JOB: Job must be in `STARTED` status → becomes `COMPLETED`

---

## Shared APIs

These APIs are available to both Consumer and Contractor roles.

#### `marketplaceJob`
Get a single job by ID.

```graphql
query GetJob($id: ID!) {
  marketplaceJob(id: $id) {
    id
    jobNumber
    status
    serviceName
    serviceDescription
    quantity
    unitPrice
    totalAmount
    currency
    
    consumerId
    consumerName
    consumerPhone
    consumerAddress
    
    contractorId
    contractorName
    contractorPhone
    
    locationId
    locationName
    areaType
    
    acceptanceDeadline
    jobStartDeadline
    completionDeadline
    
    createdAt
    acceptedAt
    startedAt
    completedAt
    cancelledAt
    
    rejectionReason
    cancellationReason
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

#### `marketplaceRating`
Get a single rating by ID.

```graphql
query GetRating($id: ID!) {
  marketplaceRating(id: $id) {
    id
    jobId
    jobNumber
    raterId
    raterName
    raterRole
    rateeId
    rateeName
    rateeRole
    rating
    comment
    createdAt
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
    userId
    averageRating
    totalRatings
    ratingBreakdown {
      stars
      count
      percentage
    }
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
    jobNumber
    amount
    currency
    status
    gatewayProvider
    gatewayOrderId
    gatewayPaymentId
    createdAt
    paidAt
  }
}
```

**Roles:** `CONSUMER`, `CONTRACTOR`, `ADMIN`

---

## Input Types

### CreateJobInput
```graphql
input CreateJobInput {
  contractorId: ID!        # Selected contractor's ID
  serviceId: ID!           # Selected service ID
  locationId: ID!          # Consumer's village/location ID
  quantity: Int = 1        # Quantity (default: 1)
  consumerAddress: String  # Service address (optional)
  consumerPhone: String    # Contact phone (optional)
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
  reason: String!          # Rejection reason (required)
}
```

### GenerateOtpInput
```graphql
input GenerateOtpInput {
  jobId: ID!
  otpType: MarketplaceOtpType!  # START_JOB or COMPLETE_JOB
}
```

### VerifyMarketplaceOtpInput
```graphql
input VerifyMarketplaceOtpInput {
  jobId: ID!
  code: String!            # 6-digit OTP code
}
```

### CancelJobInput
```graphql
input CancelJobInput {
  jobId: ID!
  reason: String           # Cancellation reason (optional)
}
```

### CreateRatingInput
```graphql
input CreateRatingInput {
  jobId: ID!
  rating: Int!             # 1-5 stars (required)
  comment: String          # Review comment (max 500 chars)
}
```

### ContractorSearchInput
```graphql
input ContractorSearchInput {
  locationId: ID!          # Village ID (required)
  serviceId: ID!           # Service ID (required)
  minRating: Int           # Minimum rating filter (1-5)
  sortBy: String           # rating | price | completed_jobs
  sortOrder: String        # asc | desc
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
  search: String           # Search in job number, names
  dateFrom: DateTime
  dateTo: DateTime
  page: Int = 1
  limit: Int = 20
}
```

---

## Enums

### MarketplaceJobStatus
| Value | Description |
|-------|-------------|
| `PAYMENT_PENDING` | Job created, awaiting payment |
| `REQUESTED` | Payment complete, waiting for contractor |
| `ACCEPTED` | Contractor accepted the job |
| `REJECTED` | Contractor rejected the job |
| `STARTED` | Job in progress (start OTP verified) |
| `COMPLETED` | Job completed (completion OTP verified) |
| `CLOSED` | Rating given, job fully closed |
| `CANCELLED` | Consumer cancelled the job |
| `REFUNDED` | Refund processed |
| `SLA_BREACH` | SLA deadline breached |

### MarketplaceOtpType
| Value | Description |
|-------|-------------|
| `START_JOB` | OTP to start the job |
| `COMPLETE_JOB` | OTP to mark job complete |

### MarketplaceAreaType
| Value | Description |
|-------|-------------|
| `URBAN` | Urban area |
| `SEMI_URBAN` | Semi-urban area |
| `RURAL` | Rural area |

### SubscriptionType
| Value | Description |
|-------|-------------|
| `FREE` | Free tier (limited categories) |
| `PREMIUM` | Premium tier (all categories) |

### PaymentStatus
| Value | Description |
|-------|-------------|
| `PENDING` | Payment initiated |
| `SUCCESS` | Payment successful |
| `FAILED` | Payment failed |
| `REFUNDED` | Payment refunded |

### RefundStatus
| Value | Description |
|-------|-------------|
| `PENDING` | Refund requested |
| `PROCESSING` | Refund being processed |
| `SUCCESS` | Refund completed |
| `FAILED` | Refund failed |

---

## Flow Diagrams

### Consumer Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONSUMER FLOW                            │
└─────────────────────────────────────────────────────────────────┘

1. BROWSE SERVICES
   ├── marketplaceServicesByCategory
   └── marketplaceServicesWithPrices(areaType)

2. SELECT LOCATION
   ├── marketplaceStates
   ├── marketplaceDistricts(stateCode)
   ├── marketplaceSubDistricts(districtCode)
   └── marketplaceVillages(subDistrictCode)

3. SEARCH CONTRACTORS
   └── searchMarketplaceContractors(locationId, serviceId)
       ├── View Profile: marketplaceContractorProfile(contractorId)
       ├── View Services: marketplaceContractorServices(contractorId)
       └── View Reviews: marketplaceContractorReviews(contractorId)

4. CREATE JOB
   └── createMarketplaceJob(input)
       └── Returns paymentSession → Redirect to payment

5. PAYMENT
   └── Complete payment on gateway
       └── Webhook updates job status → REQUESTED

6. TRACK JOB
   └── myMarketplaceJobs / marketplaceJob(id)

7. CANCEL (Optional - before STARTED)
   └── cancelMarketplaceJob(jobId, reason)
       └── Auto-triggers refund

8. RECEIVE SERVICE
   └── Share OTPs when contractor requests
       ├── START_JOB OTP → Job becomes STARTED
       └── COMPLETE_JOB OTP → Job becomes COMPLETED

9. RATE CONTRACTOR
   └── createMarketplaceRating(jobId, rating, comment)
       └── Job becomes CLOSED
```

### Contractor Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                       CONTRACTOR FLOW                           │
└─────────────────────────────────────────────────────────────────┘

1. VIEW INCOMING JOBS
   └── myContractorMarketplaceJobs(status: REQUESTED)

2. ACCEPT OR REJECT
   ├── acceptMarketplaceJob(jobId)
   │   └── Job becomes ACCEPTED, SLA deadlines set
   └── rejectMarketplaceJob(jobId, reason)
       └── Job becomes REJECTED, consumer gets refund

3. START JOB (At consumer location)
   ├── generateMarketplaceOtp(jobId, START_JOB)
   │   └── OTP sent to consumer's phone
   └── verifyMarketplaceOtp(jobId, code)
       └── Job becomes STARTED

4. COMPLETE JOB (After work done)
   ├── generateMarketplaceOtp(jobId, COMPLETE_JOB)
   │   └── OTP sent to consumer's phone
   └── verifyMarketplaceOtp(jobId, code)
       └── Job becomes COMPLETED

5. RATE CONSUMER (Optional)
   └── createMarketplaceRating(jobId, rating, comment)
```

### Job Status Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          JOB STATUS FLOW                                 │
└──────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │ PAYMENT_PENDING │ ← createMarketplaceJob
                    └────────┬────────┘
                             │ Payment Success
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
        │           │ STARTED │───cancelJob───┤
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

| Code | Message | Cause |
|------|---------|-------|
| `JOB_NOT_FOUND` | Job not found | Invalid job ID |
| `INVALID_STATUS` | Invalid job status for this action | Trying to perform action on wrong status |
| `NOT_AUTHORIZED` | You are not authorized | Job doesn't belong to user |
| `OTP_EXPIRED` | OTP has expired | OTP older than 10 minutes |
| `OTP_INVALID` | Invalid OTP code | Wrong OTP entered |
| `ALREADY_RATED` | You have already rated this job | Duplicate rating attempt |
| `CONTRACTOR_NOT_AVAILABLE` | Contractor not available | Contractor deactivated/unavailable |
| `SERVICE_NOT_AVAILABLE` | Service not available | Service deactivated |
| `LOCATION_NOT_SERVED` | Contractor doesn't serve this location | Location not in contractor's coverage |

---

## Rate Limits

| Endpoint Type | Limit |
|---------------|-------|
| Queries | 100 requests/minute |
| Mutations | 30 requests/minute |
| OTP Generation | 5 requests/job/hour |

---

*Last Updated: 19 February 2026*
