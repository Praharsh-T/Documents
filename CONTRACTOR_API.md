# MESCOM Smart Meter System - Contractor API Documentation

**Version:** 3.2.0  
**Last Updated:** 12 February 2026  
**GraphQL Endpoint:** `http://localhost:3001/graphql`

---

## Overview

This document describes the API available for Contractor mobile applications. Contractors can use **Phone + OTP** or **Email + Password** based authentication and can view assigned installations, execute work, capture evidence photos, and submit completion.

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

## Complete GraphQL Operations Reference

### Queries
| Operation | Description |
|-----------|-------------|
| `me` | Get current logged-in user |
| `myContractor` | Get contractor profile |
| `myInstallations(filters)` | Get assigned installations |
| `installation(id)` | Get single installation details |
| `installations(filters)` | Get all installations (with nested data) |
| `installationEvidenceList(installationId)` | Get evidence photos |
| `installationEvidenceSummary(installationId)` | Get evidence count summary |

### Mutations
| Operation | Description |
|-----------|-------------|
| `login(input)` | Login with phone + password |
| `updateInstallationStatus(id, input)` | Start/complete installation |
| `uploadInstallationEvidence(input)` | Link evidence to installation |

### REST Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/files/upload/private/installation-evidence` | POST | Upload evidence photo |

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
