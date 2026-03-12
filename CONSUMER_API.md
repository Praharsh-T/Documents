# Smart Meter System - Consumer API Documentation

**Version:** 3.3.0  
**Last Updated:** 20 February 2026  
**GraphQL Endpoint:** `http://localhost:3001/graphql`

---

## Overview

This document describes the API available for Consumer applications. Consumers use **Phone + OTP** based authentication and can view their sites, track installation status, upload site photos, and book electrical services via the Marketplace.

### Key Changes (v3.3.0)
- **Marketplace Added:** Consumers can now browse services, search contractors, and book jobs via the Marketplace module
- **Payment-First Model:** Consumers pay before the job is accepted; if rejected by contractor, auto-refund is issued
- **Data Isolation Fix:** ADMIN role now only sees consumers they personally created/imported (via `createdBy` field)
- **Consumer `createdBy` Column:** New column tracks which ADMIN created the consumer record

### Key Changes (v3.2.0)
- **Delayed Registration:** Consumers can only login after RO activates their site
- **New Login Mutations:** Use `requestLoginOtp` and `verifyLoginOtp` for login
- **Site Activation:** Sites have `isActivated` field - only activated sites are visible

---

## Complete Workflow (Consumer Perspective)

```
1. IMPORT (Admin imports via Excel - consumer record created, NO user account)
   ↓
2. RO REVIEW (RO reviews site, clicks "Send Link" to activate)
   ↓
3. ACTIVATION (System creates user account, sends SMS with registration link)
   ↓
4. LOGIN (Phone + OTP: 123456 in dev)
   ↓
5. VIEW SITE (Check assigned site details - only activated sites visible)
   ↓
6. UPDATE SITE (Upload photo, confirm address)
   ↓
7. WAIT FOR RO (RO verifies site, schedules appointment)
   ↓
8. INSTALLATION (Contractor installs meter)
   ↓
9. VERIFICATION (RO verifies installation)
   ↓
10. ACTIVE (Site is VERIFIED → BILLED)
```

---

## Data Model

### Consumer
A **Consumer** is a person or organization that receives electricity. Each consumer:
- Has a unique `consumerId` (RR Number from Client)
- Is linked to a **User** account for OTP login
- Can have multiple **Sites** (physical locations)
- Is created via Excel import by ADMIN

### Site
A **Site** is a physical location where a meter is installed:
- Belongs to a single Consumer
- Is located in a **Subdivision** (administrative hierarchy)
- Goes through a lifecycle from CREATED → VERIFIED → BILLED

### Hierarchy
```
Circle → Division → Subdivision → Sites
```

---

## Authentication

Consumers use **Phone + OTP** based authentication.

> **Important:** Consumers can only login after their site is activated by RO.
> Before activation, `requestLoginOtp` will return "Phone number not registered for this role".

### 1. Request Login OTP
Sends a 6-digit OTP to the consumer's phone number.

```graphql
mutation RequestLoginOtp($phone: String!, $loginType: LoginType!) {
  requestLoginOtp(input: { phone: $phone, loginType: CONSUMER }) {
    success
    message
  }
}
```

**Input:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| phone | String | Yes | Consumer's phone number (10 digits, Indian format) |
| loginType | LoginType | Yes | Must be `CONSUMER` |

**Response:**
| Field | Type | Description |
|-------|------|-------------|
| success | Boolean | Whether OTP was sent successfully |
| message | String | Success/error message |

**Error Cases:**
| Error | Cause |
|-------|-------|
| "Phone number not registered for this role" | Consumer not activated yet or wrong phone |
| "Account is deactivated" | Consumer account has been disabled |

**Note:** In development mode (`OTP_MODE=DUMMY`), OTP is always `123456`.

---

### 2. Verify Login OTP
Verifies the OTP and returns access + refresh tokens.

```graphql
mutation VerifyLoginOtp($phone: String!, $otp: String!, $loginType: LoginType!) {
  verifyLoginOtp(input: { phone: $phone, otp: $otp, loginType: CONSUMER }) {
    accessToken
    refreshToken
    user {
      id
      email
      name
      role
      phone
    }
  }
}
```

**Headers for Authenticated Requests:**
```
Authorization: Bearer <accessToken>
```

### 3. Refresh Access Token
```graphql
mutation RefreshToken($refreshToken: String!) {
  refreshToken(input: { refreshToken: $refreshToken }) {
    accessToken
    refreshToken
  }
}
```

---

### 3. Get Current User

```graphql
query Me {
  me {
    id
    email
    name
    role
    phone
  }
}
```

---

## Consumer Profile

### 4. Get Consumer Profile

```graphql
query MyConsumer {
  myConsumer {
    id
    consumerId
    name
    email
    phone
    state
    identityType
    identityNumber
    # Hierarchy (read-only, from Excel import)
    circleId
    divisionId
    subdivisionId
    circleName
    divisionName
    subdivisionName
    # Billing Address
    billingAddressLine1
    billingAddressLine2
    billingCity
    billingDistrict
    billingState
    billingPincode
    createdAt
    updatedAt
  }
}
```

**Consumer States:**
| State | Description |
|-------|-------------|
| REGISTERED | Consumer registered, pending verification |
| ACTIVE | Consumer active and verified |
| SUSPENDED | Consumer temporarily suspended |
| DEACTIVATED | Consumer deactivated |

**Note:** Hierarchy fields (circleId, divisionId, subdivisionId) are set during Excel import and cannot be modified by the consumer or RO. They are read-only.

---

## Consumer Sites

### 5. Get Consumer's Sites

```graphql
query MySites {
  mySites {
    id
    siteId
    consumerId
    addressLine1
    addressLine2
    city
    district
    pincode
    state
    landmark
    latitude
    longitude
    locationAccuracy
    sitePhotoUrl
    requiredPhase
    loadKw
    tariffCode
    tariffLocked
    siteReadinessStatus
    siteState
    # Hierarchy
    subdivisionId
    subdivisionName
    divisionName
    circleName
    # Tracking fields
    locationUpdatedBy
    addressUpdatedBy
    photoUploadedBy
    createdAt
    updatedAt
  }
}
```

**Site States (Lifecycle):**
| State | Description | Consumer Action |
|-------|-------------|-----------------|
| CREATED | Site registered via Excel import | Update address, upload photo |
| SITE_PENDING | Awaiting site verification by RO | Wait for RO review |
| SITE_VERIFIED | Site verified, ready for scheduling | Wait for appointment |
| INSTALLATION_SCHEDULED | Appointment scheduled | Prepare for contractor visit |
| METER_ASSIGNED | Meter allocated to site | Wait for contractor |
| INSTALLATION_IN_PROGRESS | Contractor working | Be available at site |
| INSTALLED | Meter installed, pending verification | Wait for RO verification |
| VERIFICATION_PENDING | RO reviewing installation | Wait for verification |
| VERIFIED | Fully verified and complete | ✅ Complete! |
| BILLED | Site active and generating bills | Pay bills |
| FAILED | Installation failed | Contact support |

---

### 6. Get Single Site Details

```graphql
query Site($id: ID!) {
  site(id: $id) {
    id
    siteId
    consumerId
    roId
    contractorId
    contractor {
      id
      companyName
      licenseNumber
      phone
    }
    addressLine1
    addressLine2
    city
    district
    pincode
    state
    landmark
    latitude
    longitude
    locationAccuracy
    locationVerifiedAt
    locationUpdatedBy
    addressUpdatedBy
    photoUploadedBy
    sitePhotoUrl
    requiredPhase
    requiredVoltage
    loadKw
    connectionType
    tariffCode
    tariffLocked
    isRapdrp
    siteReadinessStatus
    siteState
    networkProvider
    electricalReadiness
    siteAccessible
    estimatedInstallationDate
    numberOfConnections
    # Hierarchy
    subdivisionId
    subdivisionName
    divisionName
    circleName
    createdAt
    updatedAt
  }
}
```

---

## Consumer Actions

### 7. Update Site Address

Consumers can update their site address. This is required before installation can proceed.

**Important:** Use the site's `id` (UUID) from `mySites` query, NOT the `siteId` (RR Number).

```graphql
mutation UpdateSiteAddress($id: ID!, $input: UpdateSiteAddressInput!) {
  updateSiteAddress(id: $id, input: $input) {
    id
    siteId
    addressLine1
    addressLine2
    city
    district
    pincode
    landmark
    addressUpdatedBy
    updatedAt
  }
}
```

**Variables:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input": {
    "addressLine1": "123 Main Street",
    "addressLine2": "Near Bus Stand",
    "city": "Mangalore",
    "district": "Dakshina Kannada",
    "pincode": "575001",
    "state": "Karnataka",
    "landmark": "Opposite Town Hall"
  }
}
```

**Input Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| addressLine1 | String | Yes | Primary address line |
| addressLine2 | String | No | Additional address details |
| city | String | Yes | City name |
| district | String | Yes | District name |
| pincode | String | Yes | 6-digit PIN code |
| state | String | No | State name |
| landmark | String | No | Nearby landmark |

---

### 8. Upload Site Photo (Base64 - Recommended)

Consumers upload a photo of their site with GPS location. This is required before installation.

```graphql
mutation UploadSitePhotoBase64($id: ID!, $input: UploadSitePhotoInput!) {
  uploadSitePhotoBase64(id: $id, input: $input) {
    id
    sitePhotoUrl
    latitude
    longitude
    locationAccuracy
    siteState
    siteReadinessStatus
    updatedAt
  }
}
```

**Variables:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input": {
    "base64Data": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAA...",
    "latitude": 12.9716,
    "longitude": 77.5946,
    "accuracy": 10.5,
    "capturedAt": "2026-02-05T10:30:00Z"
  }
}
```

**Input Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| base64Data | String | Yes | Base64 encoded photo (with data URL prefix) |
| latitude | Float | No | GPS latitude where photo was taken |
| longitude | Float | No | GPS longitude where photo was taken |
| accuracy | Float | No | GPS accuracy in meters |
| capturedAt | DateTime | No | When the photo was captured |

---

### 9. Upload Site Photo (REST + GraphQL - Alternative)

For large files, use REST upload first, then update site with storage key.

**Step 1: Upload via REST**
```bash
POST /files/upload/private/site-photos
Content-Type: multipart/form-data
Authorization: Bearer <accessToken>

file: <image file>
```

**Response:**
```json
{
  "storageKey": "private/site-photos/abc123.jpg",
  "url": "https://storage.example.com/signed-url..."
}
```

**Step 2: Update site with storage key via GraphQL**
```graphql
mutation UploadSitePhoto($id: ID!, $input: UploadSitePhotoUrlInput!) {
  uploadSitePhoto(id: $id, input: $input) {
    id
    sitePhotoUrl
    latitude
    longitude
    locationAccuracy
    siteState
    siteReadinessStatus
    photoUploadedBy
    locationUpdatedBy
    updatedAt
  }
}
```

**Variables:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input": {
    "photoUrl": "private/site-photos/abc123.jpg",
    "latitude": 12.9716,
    "longitude": 77.5946,
    "accuracy": 10.5
  }
}
```

**Input Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| photoUrl | String | Yes | The `storageKey` from REST upload response |
| latitude | Float | No | GPS latitude where photo was taken |
| longitude | Float | No | GPS longitude where photo was taken |
| accuracy | Float | No | GPS accuracy in meters |
```

---

### 10. Update Site Location

Update GPS coordinates separately (if needed).

```graphql
mutation UpdateSiteLocation($id: ID!, $input: UpdateSiteLocationInput!) {
  updateSiteLocation(id: $id, input: $input) {
    id
    latitude
    longitude
    locationAccuracy
    locationVerifiedAt
    siteReadinessStatus
  }
}
```

**Variables:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input": {
    "latitude": 12.9716,
    "longitude": 77.5946,
    "accuracy": 10.5
  }
}
```

---

## Installation Tracking

### 11. Get Installation Status by Consumer

```graphql
query GetInstallationStatusByConsumer($consumerId: ID!) {
  installationStatusByConsumer(consumerId: $consumerId) {
    hasInstallation
    installations {
      id
      siteId
      meterId
      status
      scheduledDate
      installationDate
      initialReading
      createdAt
    }
    summary {
      total
      pending
      scheduled
      inProgress
      completed
      failed
      verified
    }
  }
}
```

---

### 12. Get Active Meter Connections for Site

```graphql
query SiteActiveConnections($siteId: ID!) {
  siteActiveConnections(siteId: $siteId) {
    id
    meterId
    connectionType
    tariffCode
    tariffLocked
    status
    initialReading
    assignedAt
    installedAt
    verifiedAt
  }
}
```

---

### 13. Get Site Connection History

```graphql
query SiteConnectionHistory($siteId: ID!) {
  siteConnectionHistory(siteId: $siteId) {
    connections {
      id
      meterId
      connectionType
      tariffCode
      status
      initialReading
      assignedAt
      installedAt
      verifiedAt
      replacedAt
      replacementReason
      rolledBackAt
      rollbackReason
    }
    activeConnection {
      id
      meterId
      tariffCode
      status
    }
    totalReplacements
  }
}
```

---

## Site Readiness Check

Before installation can proceed, the consumer must complete:

| Requirement | Field to Check | How to Complete |
|-------------|----------------|-----------------|
| ✅ Address Updated | `addressUpdatedBy != null` | Call `updateSiteAddress` |
| ✅ Photo Uploaded | `sitePhotoUrl != null` | Call `uploadSitePhotoBase64` |
| ✅ Location Set | `latitude != null` | Included in photo upload |

**Readiness Status Values:**
| Status | Description |
|--------|-------------|
| NOT_READY | Missing required data |
| PARTIAL | Some data provided |
| READY | All requirements met |
| VERIFIED | Site verified by RO |

---

## Consumer Import (Admin Only)

Consumers are created by ADMIN users via Excel import.

### Excel Format for Consumer Import:
| Column | Required | Description |
|--------|----------|-------------|
| consumerId | Yes | Unique Consumer ID (RR Number) |
| name | Yes | Consumer's full name |
| phone | Yes | 10-digit phone number (for OTP login) |
| email | No | Email address |
| tariffCode | Yes | Tariff category code |
| circle | Yes | Circle name (hierarchy) |
| division | Yes | Division name (hierarchy) |
| subdivision | Yes | Subdivision name (hierarchy) |
| addressLine1 | No | Site address |
| city | No | City (defaults to subdivision) |
| district | No | District (defaults to division) |
| pincode | No | PIN code |
| loadKw | No | Load in KW |
| connectionType | No | Connection type |
| requiredPhase | No | SINGLE_PHASE or THREE_PHASE |

---

## Data Isolation Rules

| Role | Can See Consumers |
|------|-------------------|
| SUPER_ADMIN | All consumers |
| ADMIN | Only consumers they created / imported (`createdBy = userId`) |
| RETAIL_OUTLET | Consumers whose sites are in their subdivision |
| SUB_USER | Same as parent RO's subdivision |
| CONSUMER | Only their own profile |

> **Note (v3.3.0):** ADMIN data isolation changed from state-based to creator-based. Each ADMIN now sees only the consumers they imported or manually created. This is enforced via the `createdBy` column added to the `consumers` table.

---

## Error Codes

| Code | Description |
|------|-------------|
| UNAUTHENTICATED | Invalid or missing token |
| FORBIDDEN | Insufficient permissions |
| NOT_FOUND | Resource not found |
| BAD_USER_INPUT | Invalid input data |
| INVALID_OTP | OTP is incorrect or expired |
| PHONE_NOT_REGISTERED | Phone number not found in system |

---

## Testing Credentials

| Phone | OTP |
|-------|-----|
| Any registered number | 123456 |

**Note:** In production, OTPs are sent via SMS and expire in 5 minutes.

---

---

## Marketplace (Consumer)

Consumers can browse electrical services and book licensed contractors through the marketplace.

### Marketplace Flow

```
1. Browse service categories (public)
2. Choose service → Search contractors in your village
3. Select contractor → createMarketplaceJob (payment-first)
   → Returns paymentUrl + jobId
4. Pay via payment link
5. Job status: PAYMENT_PENDING → REQUESTED
6. Contractor accepts (ACCEPTED) or rejects (REJECTED → auto-refund)
7. If accepted: Contractor generates Start OTP → you verify → STARTED
8. Contractor generates Completion OTP → you verify → COMPLETED
9. Rate the contractor (1-5 stars)
10. Job CLOSED
```

---

### M1. Browse Services with Prices

Get all services with their current price for your area type.

```graphql
query GetMarketplaceServicesWithPrices($areaType: MarketplaceAreaType!) {
  marketplaceServicesWithPrices(areaType: $areaType) {
    id
    name
    description
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
    }
    isFreeAvailable
    currentPrice
  }
}
```

**Variables:**
```json
{ "areaType": "URBAN" }
```

**Area Types:** `URBAN` | `SEMI_URBAN` | `RURAL`

**Note:** `isFreeAvailable: true` = service is available to all contractors. `isFreeAvailable: false` = PREMIUM contractor only.

---

### M2. Browse Services by Category

```graphql
query GetMarketplaceServicesByCategory($areaType: MarketplaceAreaType!) {
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
      isFreeAvailable
      currentPrice
      uom {
        name
        code
      }
    }
  }
}
```

---

### M3. Search Contractors

Search for contractors who offer a specific service in your village.

```graphql
query SearchMarketplaceContractors($input: ContractorSearchInput!) {
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
      page
      limit
      total
      totalPages
    }
  }
}
```

**Variables:**
```json
{
  "input": {
    "serviceId": "uuid-of-service",
    "locationId": "uuid-of-your-village",
    "page": 1,
    "limit": 10
  }
}
```

**`locationId`** is the UUID of your village from the marketplace locations table.

---

### M4. Get Contractor Profile

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

---

### M5. Get Contractor Reviews

```graphql
query GetMarketplaceContractorReviews($contractorId: ID!, $page: Int, $limit: Int) {
  marketplaceContractorReviews(contractorId: $contractorId, page: $page, limit: $limit) {
    items {
      id
      rating
      comment
      consumerName
      serviceName
      createdAt
    }
    pagination {
      page
      limit
      total
    }
    summary {
      averageRating
      totalReviews
      fiveStars
      fourStars
      threeStars
      twoStars
      oneStar
    }
  }
}
```

---

### M6. Book a Job (Payment-First)

Creates a job and returns a payment link. **Consumer must pay before contractor sees the job.**

```graphql
mutation CreateMarketplaceJob($input: CreateJobInput!) {
  createMarketplaceJob(input: $input) {
    jobId
    jobNumber
    amount
    paymentUrl
    paymentSessionId
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
    "consumerAddress": "123 Main Street, near temple",
    "consumerPhone": "9876543210"
  }
}
```

**Response:**
| Field | Description |
|-------|-------------|
| `jobId` | UUID of created job |
| `jobNumber` | Human-readable number (e.g., `JOB-20260220-0001`) |
| `amount` | Amount to pay (in ₹) |
| `paymentUrl` | Redirect consumer here to pay (Cashfree - placeholder in dev) |
| `paymentSessionId` | Cashfree session ID |

> **After Payment:** Job status moves from `PAYMENT_PENDING` → `REQUESTED`. Contractor is notified.

---

### M7. View My Marketplace Jobs

```graphql
query GetMyMarketplaceJobs($filters: JobsFilterInput) {
  myMarketplaceJobs(filters: $filters) {
    items {
      id
      jobNumber
      status
      serviceName
      contractorName
      contractorPhone
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

**Job Status Lifecycle:**
| Status | Description |
|--------|-------------|
| `PAYMENT_PENDING` | Created, waiting for consumer payment |
| `REQUESTED` | Paid, waiting for contractor acceptance |
| `ACCEPTED` | Contractor accepted |
| `REJECTED` | Contractor rejected → refund initiated |
| `STARTED` | Work in progress |
| `COMPLETED` | Work done, awaiting closure |
| `CLOSED` | Fully completed |
| `CANCELLED` | Cancelled by consumer |
| `REFUNDED` | Refund processed |
| `SLA_BREACH` | SLA violation detected |

---

### M8. Cancel a Job

Consumer can cancel a job (before it starts).

```graphql
mutation CancelMarketplaceJob($input: CancelJobInput!) {
  cancelMarketplaceJob(input: $input) {
    id
    jobNumber
    status
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

---

### M9. Rate the Contractor

After job is completed, consumer can rate the contractor (1–5 stars).

```graphql
mutation CreateMarketplaceRating($input: CreateRatingInput!) {
  createMarketplaceRating(input: $input) {
    id
    jobId
    jobNumber
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

---

## Complete GraphQL Operations Reference

### Queries
| Operation | Description |
|-----------|-------------|
| `me` | Get current logged-in user |
| `myConsumer` | Get consumer profile for current user |
| `mySites` | Get all sites for current consumer |
| `site(id)` | Get single site details |
| `siteActiveConnections(siteId)` | Get active meter connections |
| `siteConnectionHistory(siteId)` | Get full connection history |
| `installationStatusByConsumer(consumerId)` | Get installation summary |
| `marketplaceServicesWithPrices(areaType)` | Browse services with prices |
| `marketplaceServicesByCategory(areaType)` | Services grouped by category |
| `searchMarketplaceContractors(input)` | Search contractors by service + location |
| `marketplaceContractorProfile(contractorId)` | View contractor profile |
| `marketplaceContractorReviews(contractorId)` | View contractor reviews |
| `myMarketplaceJobs(filters)` | View my booked jobs |

### Mutations
| Operation | Description |
|-----------|-------------|
| `requestLoginOtp(input)` | Request OTP for login |
| `verifyLoginOtp(input)` | Verify OTP and get token |
| `refreshToken(input)` | Refresh access token |
| `updateSiteAddress(id, input)` | Update site address |
| `uploadSitePhotoBase64(id, input)` | Upload photo with base64 |
| `uploadSitePhoto(id, input)` | Upload photo with pre-uploaded URL |
| `updateSiteLocation(id, input)` | Update GPS coordinates |
| `createMarketplaceJob(input)` | Book a service (payment-first) |
| `cancelMarketplaceJob(input)` | Cancel a pending job |
| `createMarketplaceRating(input)` | Rate contractor after job |
