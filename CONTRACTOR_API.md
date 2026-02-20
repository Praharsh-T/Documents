# MESCOM Smart Meter System - Contractor API Documentation

**Version:** 3.3.0  
**Last Updated:** 20 February 2026  
**GraphQL Endpoint:** `http://localhost:3001/graphql`

---

## Overview

This document describes the API available for Contractor applications. Contractors can use **Phone + OTP** or **Email + Password** based authentication, manage meter installations, and participate in the **Marketplace** to receive and execute service jobs.

### Key Changes (v3.3.0)
- **Marketplace Module Added:** Contractors can now register a marketplace profile, receive job requests, accept/reject them, and manage subscriptions
- **Premium Subscription:** Contractors can upgrade to PREMIUM (Monthly/Quarterly/Yearly) to access all service categories
- **Job OTP Flow:** Start and completion of marketplace jobs verified via OTP
- **Subscription Scheduler:** Expired PREMIUM subscriptions auto-downgrade to FREE daily at midnight

### Key Changes (v3.2.0)
- **OTP Login Support:** Contractors can now login via Phone + OTP (in addition to email/password)
- **New Login Mutations:** Use `requestLoginOtp` and `verifyLoginOtp` for OTP login
- **Refresh Tokens:** Now stored in separate table with device tracking

---

## Complete Workflow (Contractor Perspective)

```
1. LOGIN (Phone + OTP OR Email + Password)
   ↓
2. VIEW JOBS (See assigned installations)
   ↓
3. START INSTALLATION (Update status to IN_PROGRESS)
   ↓
4. CAPTURE EVIDENCE (Upload photos via REST API)
   ↓
5. COMPLETE INSTALLATION (Submit reading + status)
   ↓
6. WAIT FOR RO (RO verifies work)
   ↓
7. VERIFIED ✓
```

---

## Data Model

### Contractor
A **Contractor** is a licensed electrician/firm that performs meter installations. Each contractor:
- Has a unique `code` (e.g., CONT-001)
- Has a mandatory `licenseNumber` (KERC license, alphanumeric only)
- Has a mandatory `phone` (for login)
- Has optional `companyName` and `email`
- Is linked to a **User** account for login
- Can be created via Excel import by ADMIN or manually by RO

### Installation
An **Installation** is a work assignment:
- Links a Site to a Contractor and Meter
- Created when RO assigns meter to site
- Tracks status from SCHEDULED → VERIFIED
- Contains evidence (photos, readings)

---

## Authentication

Contractors can use **Phone + OTP** OR **Email + Password** based authentication.

### Option A: Login with Phone + OTP (Recommended)

#### 1a. Request Login OTP
```graphql
mutation RequestLoginOtp($phone: String!, $loginType: LoginType!) {
  requestLoginOtp(input: { phone: $phone, loginType: CONTRACTOR }) {
    success
    message
  }
}
```

#### 1b. Verify Login OTP
```graphql
mutation VerifyLoginOtp($phone: String!, $otp: String!, $loginType: LoginType!) {
  verifyLoginOtp(input: { phone: $phone, otp: $otp, loginType: CONTRACTOR }) {
    accessToken
    refreshToken
    user {
      id
      phone
      name
      role
      email
    }
  }
}
```

**Note:** In development mode (`OTP_MODE=DUMMY`), OTP is always `123456`.

---

### Option B: Login with Email + Password

```graphql
mutation Login($email: String!, $password: String!, $loginType: LoginType!) {
  login(input: { email: $email, password: $password, loginType: CONTRACTOR }) {
    accessToken
    refreshToken
    user {
      id
      phone
      name
      role
      email
    }
  }
}
```

**Input:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| email | String | Yes | Contractor's registered email |
| password | String | Yes | Password (min 6 characters) |
| loginType | LoginType | Yes | Must be `CONTRACTOR` |

**Response:**
| Field | Type | Description |
|-------|------|-------------|
| accessToken | String | JWT token for authentication |
| refreshToken | String | Token for refreshing access |
| user.id | ID | User's unique identifier |
| user.name | String | User's full name |
| user.role | String | User role (CONTRACTOR) |
| user.phone | String | Phone number |

---

### Refresh Access Token
```graphql
mutation RefreshToken($refreshToken: String!) {
  refreshToken(input: { refreshToken: $refreshToken }) {
    accessToken
    refreshToken
  }
}
```

**Headers for Authenticated Requests:**
```
Authorization: Bearer <accessToken>
```

---

### 2. Get Current User

```graphql
query Me {
  me {
    id
    phone
    name
    role
    email
  }
}
```

---

## Contractor Profile

### 3. Get Contractor Profile

```graphql
query MyContractor {
  myContractor {
    id
    code
    name
    companyName
    licenseNumber
    licenseExpiry
    address
    phone
    email
    isActive
    userId
    divisionId
    divisionName
    createdAt
    updatedAt
  }
}
```

**Response:**
| Field | Type | Description |
|-------|------|-------------|
| id | ID | Contractor record ID |
| code | String | Unique contractor code |
| name | String? | Contractor name (from linked user) |
| companyName | String | Company/firm name |
| licenseNumber | String | KERC license number (required, alphanumeric) |
| licenseExpiry | Date? | License expiry date |
| phone | String | Contact phone |
| email | String? | Contact email |
| address | String? | Business address |
| isActive | Boolean | Whether contractor is active |
| divisionId | ID? | Assigned division |
| divisionName | String? | Division name |

---

## Assigned Installations

### 4. Get My Installations

```graphql
query MyInstallations($filters: InstallationFiltersInput) {
  myInstallations(filters: $filters) {
    items {
      id
      siteId
      meterId
      roId
      contractorId
      status
      scheduledDate
      installationDate
      latitude
      longitude
      photoUrl
      initialReading
      notes
      createdAt
      updatedAt
    }
    page
    limit
    total
  }
}
```

**Filter by Status:**
```json
{
  "filters": {
    "status": "SCHEDULED",
    "page": 1,
    "limit": 20
  }
}
```

**Installation Statuses:**
| Status | Description |
|--------|-------------|
| SCHEDULED | Job assigned, ready to start |
| IN_PROGRESS | Installation ongoing |
| COMPLETED | Done, awaiting RO verification |
| VERIFIED | Verified by RO ✓ |
| REJECTED | Rejected, needs rework |
| FAILED | Installation failed |

---

### 5. Get Single Installation Details

```graphql
query GetInstallation($id: ID!) {
  installation(id: $id) {
    id
    siteId
    meterId
    roId
    contractorId
    status
    scheduledDate
    installationDate
    latitude
    longitude
    photoUrl
    initialReading
    finalReading
    notes
    createdAt
    updatedAt
    site {
      id
      siteId
      addressLine1
      addressLine2
      city
      district
      pincode
      consumerId
    }
  }
}
```

---

### 6. Get All Installations (with full details)

```graphql
query GetInstallations($filters: InstallationFiltersInput) {
  installations(filters: $filters) {
    items {
      id
      siteId
      meterId
      roId
      contractorId
      status
      scheduledDate
      installationDate
      latitude
      longitude
      photoUrl
      initialReading
      notes
      createdAt
      updatedAt
      site {
        id
        siteId
        addressLine1
        addressLine2
        city
        district
        pincode
        consumerId
      }
      meter {
        id
        serialNumber
        type
        state
      }
      contractor {
        id
        companyName
        phone
        email
      }
      consumer {
        id
        consumerId
        name
        phone
        email
      }
    }
    page
    limit
    total
  }
}
```

---

## Installation Actions

### 7. Start Installation

Marks the installation as IN_PROGRESS and updates site state.

```graphql
mutation UpdateInstallationStatus($id: ID!, $input: UpdateInstallationInput!) {
  updateInstallationStatus(id: $id, input: $input) {
    id
    status
    installationDate
    updatedAt
  }
}
```

**Input for Starting:**
```json
{
  "id": "installation-uuid",
  "input": {
    "status": "IN_PROGRESS"
  }
}
```

**Side Effects:**
- Site state transitions: `METER_ASSIGNED → INSTALLATION_IN_PROGRESS`
- If site was at `INSTALLATION_SCHEDULED`, it auto-transitions through `METER_ASSIGNED`

---

### 8. Complete Installation

Submits the installation with meter reading.

```graphql
mutation UpdateInstallationStatus($id: ID!, $input: UpdateInstallationInput!) {
  updateInstallationStatus(id: $id, input: $input) {
    id
    status
    installationDate
    initialReading
    finalReading
    latitude
    longitude
    notes
    updatedAt
  }
}
```

**Input for Completion:**
```json
{
  "id": "installation-uuid",
  "input": {
    "status": "COMPLETED",
    "initialReading": "0.12",
    "comments": "Installation completed successfully"
  }
}
```

**⚠️ IMPORTANT:**
| Field | Type | Notes |
|-------|------|-------|
| status | String | Must be `"COMPLETED"` |
| initialReading | **String** | Must be STRING (e.g., `"0.12"` not `0.12`) |
| comments | String | Use `comments` NOT `notes` |

**Do NOT send:**
- `installationDate` - auto-set by backend
- `notes` - use `comments` instead

**Side Effects:**
- Site state transitions: `INSTALLATION_IN_PROGRESS → INSTALLED`
- Meter state transitions: `ASSIGNED → INSTALLED`

---

## Evidence Upload (REST API)

### 9. Upload Installation Evidence

Evidence photos are uploaded via REST API (not GraphQL) for better file handling.

**REST Endpoint:**
```bash
POST /files/upload/private/installation-evidence
Content-Type: multipart/form-data
Authorization: Bearer <accessToken>

file: <image file>
```

**Response:**
```json
{
  "storageKey": "private/installation-evidence/abc123.jpg",
  "url": "https://storage.example.com/signed-url..."
}
```

**Then link to installation via GraphQL:**

```graphql
mutation UploadInstallationEvidence($input: CreateInstallationEvidenceInput!) {
  uploadInstallationEvidence(input: $input) {
    success
    message
    evidence {
      id
      evidenceType
      fileUrl
      ocrReading
      createdAt
    }
  }
}
```

**Input:**
```json
{
  "input": {
    "installationId": "installation-uuid",
    "evidenceType": "METER_PHOTO",
    "storageKey": "private/installation-evidence/abc123.jpg",
    "latitude": 12.9716,
    "longitude": 77.5946,
    "capturedAt": "2026-02-05T10:30:00Z"
  }
}
```

**Evidence Types:**
| Type | Description |
|------|-------------|
| METER_PHOTO | Photo of installed meter |
| SEAL_PHOTO | Photo of meter seal |
| READING_PHOTO | Photo of initial reading |
| LOCATION_PHOTO | Photo of installation location |
| WIRING_PHOTO | Photo of wiring |
| OTHER | Other evidence |

---

### 10. Get Evidence for Installation

```graphql
query GetInstallationEvidenceList($installationId: ID!) {
  installationEvidenceList(installationId: $installationId) {
    items {
      id
      evidenceType
      fileUrl
      fileName
      latitude
      longitude
      capturedAt
      ocrReading
      ocrConfidence
      uploadedBy
      createdAt
    }
    total
  }
}
```

---

### 11. Get Evidence Summary

```graphql
query GetInstallationEvidenceSummary($installationId: ID!) {
  installationEvidenceSummary(installationId: $installationId) {
    totalPhotos
    meterPhotos
    sealPhotos
    readingPhotos
    locationPhotos
    hasAllRequired
    missingTypes
  }
}
```

---

## Installation Workflow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    RO ASSIGNS METER                      │
│         (Creates installation with SCHEDULED status)     │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│               CONTRACTOR VIEWS JOB                       │
│     Query: myInstallations(filters: {status: SCHEDULED}) │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│              CONTRACTOR STARTS WORK                      │
│   Mutation: updateInstallationStatus(status: IN_PROGRESS)│
│   Site: METER_ASSIGNED → INSTALLATION_IN_PROGRESS        │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│            CONTRACTOR CAPTURES EVIDENCE                  │
│   1. POST /files/upload/private/installation-evidence    │
│   2. uploadInstallationEvidence(type: METER_PHOTO, ...)  │
│   Required: METER_PHOTO, SEAL_PHOTO, READING_PHOTO       │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│            CONTRACTOR COMPLETES WORK                     │
│   Mutation: updateInstallationStatus(                    │
│     status: COMPLETED,                                   │
│     initialReading: "0.12",                              │
│     comments: "Done"                                     │
│   )                                                      │
│   Site: INSTALLATION_IN_PROGRESS → INSTALLED             │
│   Meter: ASSIGNED → INSTALLED                            │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│               RO VERIFIES INSTALLATION                   │
│   RO creates verification → approves                     │
│   Site: INSTALLED → VERIFICATION_PENDING → VERIFIED      │
│   Meter: INSTALLED → ACTIVE                              │
│   Installation: COMPLETED → VERIFIED                     │
└─────────────────────────────────────────────────────────┘
```

---

## Site State Transitions (Automatic)

| Installation Status | Site State Transition |
|--------------------|-----------------------|
| SCHEDULED | Site at INSTALLATION_SCHEDULED or METER_ASSIGNED |
| IN_PROGRESS | → INSTALLATION_IN_PROGRESS |
| COMPLETED | → INSTALLED |
| VERIFIED | → VERIFICATION_PENDING → VERIFIED |
| FAILED | → FAILED |

---

## Contractor Import (Admin Only)

Contractors are created by ADMIN users via Excel import.

### Excel Format for Contractor Import:
| Column | Required | Validation |
|--------|----------|------------|
| code | Yes | Unique contractor code |
| userName | Yes | Display name for the user |
| phone | **Yes** | Valid phone number (for login) |
| licenseNumber | **Yes** | Alphanumeric only (A-Z, 0-9), unique |
| companyName | No | Optional company name |
| email | No | Valid email format if provided |
| address | No | Optional address |
| userPassword | Yes | Default password (min 6 chars) |

**License Number Validation:**
- Required field (cannot be empty)
- Must be alphanumeric only (A-Z, a-z, 0-9)
- Must be unique across all contractors

---

## Error Codes

| Code | Description |
|------|-------------|
| UNAUTHENTICATED | Invalid or expired token |
| FORBIDDEN | Not authorized for this action |
| NOT_FOUND | Installation or site not found |
| LICENSE_REQUIRED | License number is required |
| LICENSE_INVALID | License number must be alphanumeric |
| LICENSE_EXISTS | License number already exists |
| INVALID_STATUS_TRANSITION | Cannot transition to requested status |

---

## Data Isolation Rules

Contractors see only their own installations. They do not have access to:
- Other contractors' work
- Consumer personal data (except site address)
- Meter inventory (only assigned meters)
- RO internal data

---

## Testing Credentials

| Phone/Email | Password |
|-------------|----------|
| contractor@example.com | contractor123 |
| (contractor phone) | contractor123 |

---

---

## Marketplace (Contractor)

Contractors participate in the marketplace by maintaining a profile, accepting job requests, and completing work via OTP verification. A **PREMIUM subscription** unlocks all service categories.

---

### Subscription Tiers

| Plan | Duration | Price | Available Services |
|------|----------|-------|-------------------|
| FREE | Indefinite | ₹0 | Energy Meter Installation only (`isFreeAvailable: true`) |
| PREMIUM | 30 / 90 / 365 days | ₹499 / ₹1,347 / ₹4,491 | All service categories |

---

### MP1. Get My Marketplace Profile

```graphql
query GetMyMarketplaceProfile($contractorId: ID!) {
  marketplaceContractorProfile(contractorId: $contractorId) {
    id
    contractorId
    contractorName
    companyName
    phone
    email
    subscriptionType
    subscriptionValidTill
    ratingAvg
    totalRatings
    slaBreachCount
    rejectionCount
    totalJobsCompleted
    qrProfileToken
    isMarketplaceActive
    createdAt
    updatedAt
  }
}
```

**Note:** `contractorId` is the contractor's record UUID (not user ID).

---

### MP2. Toggle Marketplace Active/Inactive

Contractors can go offline (stop receiving new jobs) at any time.

```graphql
mutation UpdateContractorMarketplaceProfile($input: UpdateContractorProfileInput!) {
  updateContractorMarketplaceProfile(input: $input) {
    id
    contractorId
    isMarketplaceActive
    updatedAt
  }
}
```

**Variables:**
```json
{
  "input": {
    "contractorId": "contractor-uuid",
    "isMarketplaceActive": false
  }
}
```

---

### MP3. View My Marketplace Jobs

```graphql
query GetMyContractorMarketplaceJobs($filters: JobsFilterInput) {
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

**Filter examples:**
```json
{ "filters": { "status": "REQUESTED", "page": 1, "limit": 20 } }
```

**Job Status Lifecycle:**
| Status | Description |
|--------|-------------|
| `REQUESTED` | Paid by consumer, waiting for your acceptance |
| `ACCEPTED` | You accepted — start deadline begins |
| `REJECTED` | You rejected — consumer refunded |
| `STARTED` | Work in progress |
| `COMPLETED` | Work done — awaiting closure |
| `CLOSED` | Fully closed |
| `SLA_BREACH` | Deadline missed |

---

### MP4. Accept a Job

```graphql
mutation AcceptMarketplaceJob($input: AcceptJobInput!) {
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
{ "input": { "jobId": "job-uuid" } }
```

**Rules:**
- Job must be in `REQUESTED` status
- Must be assigned to this contractor
- Acceptance deadline applies (set at job creation)

---

### MP5. Reject a Job

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

**Rules:**
- Reason is required
- Triggers automatic refund to consumer
- Rejection count on contractor profile increments

---

### MP6. Generate OTP (Start or Complete)

Before starting or completing a job, generate an OTP that the consumer verifies.

```graphql
mutation GenerateMarketplaceOtp($input: GenerateOtpInput!) {
  generateMarketplaceOtp(input: $input) {
    id
    jobId
    otpType
    expiresAt
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

**OTP Types:** `START_JOB` | `COMPLETE_JOB`

> The OTP is sent via SMS to the **consumer's** phone. The contractor then asks the consumer to read it out.

---

### MP7. Verify OTP (Start or Complete Job)

```graphql
mutation VerifyMarketplaceOtp($input: VerifyMarketplaceOtpInput!) {
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

**Flow:**
- `START_JOB` OTP verified → Job `ACCEPTED` → `STARTED`
- `COMPLETE_JOB` OTP verified → Job `STARTED` → `COMPLETED`

---

### MP8. View Subscription Plans

```graphql
query GetSubscriptionPlans {
  subscriptionPlans {
    duration
    durationMonths
    price
    finalPrice
    discountPercent
    description
  }
}
```

**Response:**
| Duration | Months | Price | Discount | Final |
|----------|--------|-------|----------|-------|
| MONTHLY | 1 | ₹499 | 0% | ₹499 |
| QUARTERLY | 3 | ₹1,497 | 10% | ₹1,347 |
| YEARLY | 12 | ₹5,988 | 25% | ₹4,491 |

---

### MP9. Get My Subscription Status

```graphql
query GetMySubscriptionStatus {
  mySubscriptionStatus {
    currentType
    isActive
    validTill
    daysRemaining
    canUpgrade
    canRenew
  }
}
```

---

### MP10. Initiate Subscription Upgrade

```graphql
mutation InitiateSubscriptionUpgrade($input: InitiateSubscriptionUpgradeInput!) {
  initiateSubscriptionUpgrade(input: $input) {
    success
    message
    paymentIntent {
      id
      amount
      duration
      paymentLink
      orderId
      expiresAt
      status
    }
  }
}
```

**Variables:**
```json
{
  "input": {
    "contractorId": "contractor-uuid",
    "duration": "MONTHLY"
  }
}
```

**Response `paymentIntent`:**
- `id` — Payment intent UUID (needed for confirm step)
- `paymentLink` — Redirect contractor here to pay
- `expiresAt` — Payment intent expires 30 minutes after creation

---

### MP11. Confirm Subscription Payment

After payment is made, confirm to activate PREMIUM.

```graphql
mutation ConfirmSubscriptionPayment($input: ConfirmSubscriptionPaymentInput!) {
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
    "paymentIntentId": "intent-uuid"
  }
}
```

**After confirmation:**
- `subscriptionType` on contractor profile → `PREMIUM`
- `subscriptionValidTill` → set to now + duration
- If already PREMIUM: expiry is **extended** from existing expiry date (not reset)

---

### MP12. View My Subscription History

```graphql
query GetMySubscriptionHistory {
  mySubscriptionHistory {
    id
    subscriptionType
    startDate
    endDate
    isActive
    createdAt
  }
}
```

---

### MP13. View My Received Ratings

```graphql
query GetMyContractorReviews($contractorId: ID!, $page: Int, $limit: Int) {
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

## Marketplace OTP Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│  Consumer creates job + pays → Job: REQUESTED        │
└─────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│  Contractor views job → acceptMarketplaceJob()       │
│  Job: REQUESTED → ACCEPTED                          │
└─────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│  Contractor arrives at site                          │
│  generateMarketplaceOtp(START_JOB)                  │
│  → OTP sent to consumer's phone                     │
│  Consumer reads OTP to contractor                    │
│  verifyMarketplaceOtp(code: "123456")                │
│  Job: ACCEPTED → STARTED                            │
└─────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│  Work completed                                      │
│  generateMarketplaceOtp(COMPLETE_JOB)               │
│  → OTP sent to consumer's phone                     │
│  Consumer reads OTP to contractor                    │
│  verifyMarketplaceOtp(code: "654321")                │
│  Job: STARTED → COMPLETED                           │
└─────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│  Consumer rates contractor → Job: CLOSED            │
└─────────────────────────────────────────────────────────┘
```

---

## Marketplace Assigned Locations

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

---

## Marketplace Assigned Services

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

## Complete GraphQL Operations Reference

### Queries
| Operation | Description |
|-----------|-------------|
| `me` | Get current logged-in user |
| `myContractor` | Get contractor profile |
| `myInstallations(filters)` | Get assigned meter installations |
| `installation(id)` | Get single installation details |
| `installations(filters)` | Get all installations (with nested data) |
| `installationEvidenceList(installationId)` | Get evidence photos |
| `installationEvidenceSummary(installationId)` | Get evidence count summary |
| `marketplaceContractorProfile(contractorId)` | Get my marketplace profile |
| `myContractorMarketplaceJobs(filters)` | View marketplace jobs |
| `subscriptionPlans` | View subscription pricing |
| `mySubscriptionStatus` | My current subscription status |
| `mySubscriptionHistory` | My subscription history |
| `marketplaceContractorLocations(contractorId)` | My serviceable locations |
| `marketplaceContractorServices(contractorId)` | My offered services |
| `marketplaceContractorReviews(contractorId)` | My reviews from consumers |

### Mutations
| Operation | Description |
|-----------|-------------|
| `requestLoginOtp(input)` | Request OTP for login |
| `verifyLoginOtp(input)` | Verify OTP and get token |
| `login(input)` | Login with email + password |
| `refreshToken(input)` | Refresh access token |
| `updateInstallationStatus(id, input)` | Start/complete meter installation |
| `uploadInstallationEvidence(input)` | Link evidence to installation |
| `acceptMarketplaceJob(input)` | Accept a marketplace job |
| `rejectMarketplaceJob(input)` | Reject a marketplace job |
| `generateMarketplaceOtp(input)` | Generate OTP for start/complete |
| `verifyMarketplaceOtp(input)` | Verify OTP to start/complete job |
| `updateContractorMarketplaceProfile(input)` | Toggle active/inactive |
| `initiateSubscriptionUpgrade(input)` | Start subscription upgrade payment |
| `confirmSubscriptionPayment(input)` | Confirm payment and activate PREMIUM |

### REST Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/files/upload/private/installation-evidence` | POST | Upload installation evidence photo |

---

## Common Issues & Solutions

### Issue: "initialReading must be a string"
**Solution:** Send `"0.12"` not `0.12` in the input.

### Issue: "Invalid status transition"
**Solution:** Check current status. Can only go:
- SCHEDULED → IN_PROGRESS
- IN_PROGRESS → COMPLETED
- COMPLETED → VERIFIED (RO only)

### Issue: "Installation not found"
**Solution:** Ensure you're using the installation UUID, not site UUID.

### Issue: "Evidence upload 401 error"
**Solution:** Check Bearer token is valid. Token format: `Bearer eyJ...`
