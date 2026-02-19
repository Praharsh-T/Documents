# Marketplace API Documentation

**Last Updated:** 19 February 2026

This document covers the Consumer and Contractor APIs for the SmartMeter Marketplace module.

---

## Table of Contents
1. [Authentication](#authentication)
2. [Consumer APIs](#consumer-apis)
3. [Contractor APIs](#contractor-apis)
4. [Common Types](#common-types)
5. [Error Handling](#error-handling)
6. [Flow Diagrams](#flow-diagrams)

---

## Authentication

All marketplace APIs require a JWT token in the Authorization header:
```
Authorization: Bearer <token>
```

### Consumer Login (OTP-based)

**Step 1: Request OTP**
```graphql
mutation RequestLoginOtp($input: RequestLoginOtpInput!) {
  requestLoginOtp(input: $input) {
    success
    message
  }
}

# Variables
{
  "input": {
    "phone": "9876543210",
    "loginType": "CONSUMER"
  }
}
```

**Step 2: Verify OTP**
```graphql
mutation VerifyLoginOtp($input: VerifyLoginOtpInput!) {
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

# Variables
{
  "input": {
    "phone": "9876543210",
    "otp": "123456",
    "loginType": "CONSUMER"
  }
}
```

### Contractor Login (OTP-based)

Same as consumer, but with `loginType: "CONTRACTOR"`

---

## Consumer APIs

### 1. Browse Services

#### Get Services by Category
Returns services grouped by category with current prices for the consumer's area type.

```graphql
query GetMarketplaceServicesByCategory($areaType: MarketplaceAreaType!) {
  marketplaceServicesByCategory(areaType: $areaType) {
    category
    services {
      id
      name
      description
      category
      uom
      isPremiumOnly
      currentPrice
    }
  }
}

# Variables
{
  "areaType": "URBAN"  # URBAN | SEMI_URBAN | RURAL
}
```

#### Get Services with Prices
```graphql
query GetMarketplaceServicesWithPrices($areaType: MarketplaceAreaType!) {
  marketplaceServicesWithPrices(areaType: $areaType) {
    id
    name
    description
    category
    uom
    isPremiumOnly
    currentPrice
  }
}
```

### 2. Location Selection

#### Get States
```graphql
query GetMarketplaceStates {
  marketplaceStates {
    code
    name
  }
}
```

#### Get Districts
```graphql
query GetMarketplaceDistricts($stateCode: String!) {
  marketplaceDistricts(stateCode: $stateCode) {
    code
    name
    stateCode
  }
}
```

#### Get Sub-Districts
```graphql
query GetMarketplaceSubDistricts($districtCode: String!) {
  marketplaceSubDistricts(districtCode: $districtCode) {
    code
    name
    districtCode
  }
}
```

#### Get Villages (Consumer's location selection)
```graphql
query GetMarketplaceVillages($districtCode: String!, $subDistrictCode: String) {
  marketplaceVillages(districtCode: $districtCode, subDistrictCode: $subDistrictCode) {
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

### 3. Search Contractors

Search for contractors who service a specific location and offer a specific service.

```graphql
query SearchMarketplaceContractors($input: ContractorSearchInput!) {
  searchMarketplaceContractors(input: $input) {
    items {
      contractorId
      contractorName
      companyName
      phone
      ratingAvg
      totalJobsCompleted
      totalRatings
      isPremiumContractor
      priceForService
    }
    pagination {
      page
      limit
      total
      totalPages
      hasNext
      hasPrev
    }
  }
}

# Variables
{
  "input": {
    "serviceId": "uuid-of-service",
    "locationId": "uuid-of-location",
    "page": 1,
    "limit": 10
  }
}
```

### 4. View Contractor Profile & Reviews

#### Get Contractor Profile
```graphql
query GetMarketplaceContractorProfile($contractorId: ID!) {
  marketplaceContractorProfile(contractorId: $contractorId) {
    id
    contractorId
    contractorName
    companyName
    phone
    email
    subscriptionType
    ratingAvg
    totalRatings
    totalJobsCompleted
    isMarketplaceActive
  }
}
```

#### Get Contractor Reviews
```graphql
query GetMarketplaceContractorReviews($contractorId: ID!, $page: Int, $limit: Int) {
  marketplaceContractorReviews(contractorId: $contractorId, page: $page, limit: $limit) {
    items {
      id
      jobId
      jobNumber
      rating
      comment
      raterName
      createdAt
    }
    pagination {
      page
      limit
      total
      totalPages
      hasNext
      hasPrev
    }
  }
}
```

### 5. Create Job (Book Service)

Creates a job with `PAYMENT_PENDING` status. Consumer must complete payment to proceed.

```graphql
mutation CreateMarketplaceJob($input: CreateJobInput!) {
  createMarketplaceJob(input: $input) {
    jobId
    jobNumber
    amount
    paymentSessionId
    paymentUrl
  }
}

# Variables
{
  "input": {
    "serviceId": "uuid-of-service",
    "contractorId": "uuid-of-contractor",
    "locationId": "uuid-of-location",
    "consumerAddress": "123 Main Street, City",
    "consumerPhone": "9876543210",
    "quantity": 1
  }
}
```

**Response:**
```json
{
  "data": {
    "createMarketplaceJob": {
      "jobId": "uuid",
      "jobNumber": "JOB-20260219-0001",
      "amount": 500,
      "paymentSessionId": "cf_session_xxx",
      "paymentUrl": "https://payments.cashfree.com/..."
    }
  }
}
```

### 6. Simulate Payment Success (DEV ONLY)

For development/testing without Cashfree integration:

```graphql
mutation SimulatePaymentSuccess($jobId: ID!) {
  simulatePaymentSuccess(jobId: $jobId) {
    id
    jobNumber
    status
    paidAt
  }
}
```

### 7. View My Jobs

```graphql
query GetMyMarketplaceJobs($filters: JobsFilterInput) {
  myMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      consumerId
      consumerName
      consumerPhone
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
      status
      createdAt
      paidAt
      acceptedAt
      rejectedAt
      startedAt
      completedAt
      closedAt
      rejectionReason
    }
    pagination {
      page
      limit
      total
      totalPages
      hasNext
      hasPrev
    }
  }
}

# Variables (all optional)
{
  "filters": {
    "status": "REQUESTED",  # Optional: filter by status
    "page": 1,
    "limit": 20
  }
}
```

### 8. Get Single Job Details

```graphql
query GetMarketplaceJob($id: ID!) {
  marketplaceJob(id: $id) {
    id
    jobNumber
    serviceName
    contractorName
    contractorPhone
    locationName
    consumerAddress
    priceSnapshot
    status
    acceptanceDeadline
    startDeadline
    completionDeadline
    createdAt
    paidAt
    acceptedAt
    startedAt
    completedAt
    rejectionReason
  }
}
```

### 9. Cancel Job

Consumer can cancel a job if it's in `PAYMENT_PENDING` or `REQUESTED` status.

```graphql
mutation CancelMarketplaceJob($input: CancelJobInput!) {
  cancelMarketplaceJob(input: $input) {
    id
    jobNumber
    status
  }
}

# Variables
{
  "input": {
    "jobId": "uuid-of-job",
    "reason": "Changed my mind"  # Optional
  }
}
```

### 10. Rate Contractor

After job completion, consumer can rate the contractor.

```graphql
mutation CreateMarketplaceRating($input: CreateRatingInput!) {
  createMarketplaceRating(input: $input) {
    id
    jobId
    rating
    comment
    createdAt
  }
}

# Variables
{
  "input": {
    "jobId": "uuid-of-job",
    "rating": 5,         # 1-5 stars
    "comment": "Excellent service!"  # Optional
  }
}
```

### 11. Payment & Refund Status

#### Get Payment Details
```graphql
query GetMarketplacePaymentByJob($jobId: ID!) {
  marketplacePaymentByJob(jobId: $jobId) {
    id
    jobId
    jobNumber
    amount
    currency
    status
    gatewayProvider
    completedAt
    failureReason
  }
}
```

#### Get Refund Status
```graphql
query GetMarketplaceRefundByJob($jobId: ID!) {
  marketplaceRefundByJob(jobId: $jobId) {
    id
    jobId
    jobNumber
    amount
    reason
    status
    processedAt
    failureReason
  }
}
```

---

## Contractor APIs

### 1. View My Jobs

```graphql
query GetMyContractorMarketplaceJobs($filters: JobsFilterInput) {
  myContractorMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      consumerName
      consumerPhone
      consumerAddress
      serviceId
      serviceName
      locationId
      locationName
      priceSnapshot
      areaTypeSnapshot
      status
      acceptanceDeadline
      startDeadline
      completionDeadline
      createdAt
      paidAt
      acceptedAt
      startedAt
      completedAt
    }
    pagination {
      page
      limit
      total
      totalPages
      hasNext
      hasPrev
    }
  }
}

# Variables - Filter by status
{
  "filters": {
    "status": "REQUESTED",  # REQUESTED | ACCEPTED | STARTED | COMPLETED
    "page": 1,
    "limit": 20
  }
}
```

### 2. Accept Job

Contractor accepts a job request. SLA countdown for job start begins.

```graphql
mutation AcceptMarketplaceJob($input: AcceptJobInput!) {
  acceptMarketplaceJob(input: $input) {
    id
    jobNumber
    status
    acceptedAt
    startDeadline
  }
}

# Variables
{
  "input": {
    "jobId": "uuid-of-job"
  }
}
```

### 3. Reject Job

Contractor rejects a job request. Auto-refund is triggered for the consumer.

```graphql
mutation RejectMarketplaceJob($input: RejectJobInput!) {
  rejectMarketplaceJob(input: $input) {
    id
    jobNumber
    status
    rejectedAt
    rejectionReason
  }
}

# Variables
{
  "input": {
    "jobId": "uuid-of-job",
    "reason": "Unable to service this area currently"
  }
}
```

### 4. Generate OTP

Generates OTP for job start or completion. OTP is sent to consumer via SMS.

```graphql
mutation GenerateMarketplaceOtp($input: GenerateOtpInput!) {
  generateMarketplaceOtp(input: $input) {
    id
    jobId
    otpType
    code
    expiresAt
    isUsed
  }
}

# Variables for START OTP
{
  "input": {
    "jobId": "uuid-of-job",
    "otpType": "START"
  }
}

# Variables for COMPLETE OTP
{
  "input": {
    "jobId": "uuid-of-job",
    "otpType": "COMPLETE"
  }
}
```

**Note:** In development mode (OTP_MODE=DUMMY), OTP code is always `123456`.

### 5. Verify OTP and Update Job Status

Verifies the OTP provided by consumer and updates job status.

```graphql
mutation VerifyMarketplaceOtp($input: VerifyMarketplaceOtpInput!) {
  verifyMarketplaceOtp(input: $input) {
    id
    jobNumber
    status
    startedAt
    completedAt
    completionDeadline
  }
}

# Variables for START
{
  "input": {
    "jobId": "uuid-of-job",
    "otpType": "START",
    "code": "123456"
  }
}

# Variables for COMPLETE
{
  "input": {
    "jobId": "uuid-of-job",
    "otpType": "COMPLETE",
    "code": "654321"
  }
}
```

### 6. View My Profile

```graphql
query MyContractor {
  myContractor {
    id
    companyName
    phoneNumber
    email
    isActive
  }
}

query GetMyMarketplaceProfile($contractorId: ID!) {
  marketplaceContractorProfile(contractorId: $contractorId) {
    id
    contractorId
    contractorName
    companyName
    subscriptionType
    subscriptionValidTill
    ratingAvg
    totalRatings
    slaBreachCount
    totalJobsCompleted
    isMarketplaceActive
  }
}
```

### 7. View My Assigned Locations

```graphql
query GetMarketplaceContractorLocations($contractorId: ID!) {
  marketplaceContractorLocations(contractorId: $contractorId) {
    id
    locationId
    villageName
    districtName
    stateName
    areaType
    isActive
  }
}
```

### 8. View My Assigned Services

```graphql
query GetMarketplaceContractorServices($contractorId: ID!) {
  marketplaceContractorServices(contractorId: $contractorId) {
    id
    serviceId
    serviceName
    serviceCategory
    isActive
  }
}
```

---

## Common Types

### Job Status Enum
```typescript
enum JobStatus {
  PAYMENT_PENDING   // Job created, awaiting payment
  REQUESTED         // Payment complete, awaiting contractor response
  ACCEPTED          // Contractor accepted, awaiting job start
  REJECTED          // Contractor rejected (refund triggered)
  CANCELLED         // Consumer cancelled (refund triggered if paid)
  STARTED           // Job in progress
  COMPLETED         // Job completed, awaiting rating
  CLOSED            // Job closed (rated)
  REFUNDED          // Refund processed
  AUTO_EXPIRED      // SLA breach - auto-expired
}
```

### Area Type Enum
```typescript
enum MarketplaceAreaType {
  URBAN
  SEMI_URBAN
  RURAL
}
```

### OTP Type Enum
```typescript
enum OtpType {
  START      // To start the job
  COMPLETE   // To complete the job
}
```

### Pagination Input
```typescript
interface PaginationInput {
  page: number;   // Default: 1
  limit: number;  // Default: 20, Max: 100
}
```

### Pagination Response
```typescript
interface Pagination {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}
```

---

## Error Handling

All errors are returned in the standard GraphQL format:

```json
{
  "errors": [
    {
      "message": "Error description",
      "extensions": {
        "code": "ERROR_CODE",
        "statusCode": 400
      }
    }
  ],
  "data": null
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `UNAUTHENTICATED` | Missing or invalid JWT token |
| `FORBIDDEN` | User doesn't have required role |
| `NOT_FOUND` | Resource not found |
| `BAD_REQUEST` | Invalid input data |
| `CONFLICT` | Action conflicts with current state |

### Job-Specific Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "Job is not in PAYMENT_PENDING status" | Trying to pay for already-paid job | Check job status first |
| "Job is not in REQUESTED status" | Trying to accept/reject non-pending job | Refresh job list |
| "Job is not in ACCEPTED status" | Trying to start non-accepted job | Accept job first |
| "Location not found or not classified" | Location doesn't exist or isn't ready | Contact admin |
| "No pricing configured" | Service doesn't have pricing for area | Contact admin |
| "Invalid OTP" | Wrong OTP code | Get correct OTP from consumer |
| "OTP expired" | OTP validity (10 min) expired | Generate new OTP |

---

## Flow Diagrams

### Consumer Booking Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                     CONSUMER BOOKING FLOW                         │
└──────────────────────────────────────────────────────────────────┘

1. Consumer Login (OTP)
   │
   ▼
2. Browse Services
   marketplaceServicesByCategory(areaType)
   │
   ▼
3. Select Service & Search Contractors
   searchMarketplaceContractors(serviceId, locationId)
   │
   ▼
4. View Contractor Profile & Reviews
   marketplaceContractorProfile(contractorId)
   marketplaceContractorReviews(contractorId)
   │
   ▼
5. Create Job (Book Service)
   createMarketplaceJob(input)
   │
   ├──► Job Status: PAYMENT_PENDING
   │
   ▼
6. Complete Payment
   (Cashfree redirect / simulatePaymentSuccess for dev)
   │
   ├──► Job Status: REQUESTED
   │
   ▼
7. Wait for Contractor
   │
   ├──► Accepted ──► Job Status: ACCEPTED
   │
   └──► Rejected ──► Job Status: REJECTED ──► Refund Triggered
```

### Contractor Job Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    CONTRACTOR JOB FLOW                            │
└──────────────────────────────────────────────────────────────────┘

1. Contractor Login (OTP)
   │
   ▼
2. View Pending Jobs
   myContractorMarketplaceJobs(status: REQUESTED)
   │
   ▼
3. Review Job Details
   │
   ├──► Accept Job
   │    acceptMarketplaceJob(jobId)
   │    Job Status: ACCEPTED
   │    │
   │    ▼
   │    4. Generate START OTP
   │       generateMarketplaceOtp(jobId, START)
   │       │
   │       ├──► SMS sent to consumer
   │       │
   │       ▼
   │    5. Consumer shares OTP
   │       │
   │       ▼
   │    6. Verify START OTP
   │       verifyMarketplaceOtp(jobId, START, code)
   │       Job Status: STARTED
   │       │
   │       ▼
   │    7. Complete Work
   │       │
   │       ▼
   │    8. Generate COMPLETE OTP
   │       generateMarketplaceOtp(jobId, COMPLETE)
   │       │
   │       ▼
   │    9. Consumer shares OTP
   │       │
   │       ▼
   │   10. Verify COMPLETE OTP
   │       verifyMarketplaceOtp(jobId, COMPLETE, code)
   │       Job Status: COMPLETED
   │
   └──► Reject Job
        rejectMarketplaceJob(jobId, reason)
        Job Status: REJECTED
        Refund triggered automatically
```

### SLA Deadlines

```
Job Created ──► Payment ──► REQUESTED
                               │
                               ├── acceptanceDeadline (4-8 hours)
                               │   If missed: AUTO_EXPIRED
                               │
                               ▼
                           ACCEPTED
                               │
                               ├── startDeadline (8-24 hours)
                               │   If missed: SLA breach recorded
                               │
                               ▼
                            STARTED
                               │
                               ├── completionDeadline (24-72 hours)
                               │   If missed: SLA breach recorded
                               │
                               ▼
                           COMPLETED
                               │
                               ▼
                            CLOSED (after rating)
```

---

## Testing

### Environment Variables

```bash
# For test script
export ADMIN_EMAIL="admin@smartmeter.app"
export ADMIN_PASSWORD="admin123"
export CONSUMER_PHONE="9876543210"
export CONTRACTOR_PHONE="9876543211"
```

### Run Test Script

```bash
cd /path/to/user-service
node scripts/test-marketplace-flow.js
```

---

## Notes

1. **Payment Integration:** Currently using `simulatePaymentSuccess` for testing. Cashfree integration pending.

2. **SMS/OTP:** In dev mode (`OTP_MODE=DUMMY`), OTP is always `123456`.

3. **SLA Scheduler:** Background job runs every 5 minutes to check for SLA breaches.

4. **Refunds:** Auto-triggered on contractor rejection or consumer cancellation (after payment).

5. **Ratings:** Consumer can only rate after job is `COMPLETED`. One rating per job.
