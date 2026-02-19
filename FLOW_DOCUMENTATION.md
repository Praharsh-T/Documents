# MESCOM Application Flow Documentation

## Overview

This document explains the complete flow of the MESCOM (Mangalore Electricity Supply Company) meter installation management system, covering both frontend and backend implementations.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                 │
│                   (Next.js 14 + Apollo Client)                   │
│                      http://localhost:3000                       │
├─────────────────────────────────────────────────────────────────┤
│  Roles: Admin | RO (Retail Outlet) | Consumer | Contractor      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ GraphQL (Apollo Client)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         BACKEND                                  │
│                   (NestJS + GraphQL + Drizzle)                   │
│                      http://localhost:3001                       │
├─────────────────────────────────────────────────────────────────┤
│  Modules: Auth | Consumers | Sites | Meters | Installations     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Drizzle ORM
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       PostgreSQL 17                              │
│                    Database: mescom                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## User Roles & Access

### Role Hierarchy
```
SUPER_ADMIN
    │
    ▼ creates
  ADMIN
    │
    ▼ creates
RETAIL_OUTLET (RO)
    │
    ▼ creates
CONTRACTOR / CONSUMER / SUB_USER
```

| Role | Login URL | Dashboard | Key Actions |
|------|-----------|-----------|-------------|
| **Super Admin** | `/login` → `/admin` | Admin Dashboard | Create Admins, view all data |
| **Admin** | `/login` → `/admin` | Admin Dashboard | Create ROs, manage master data, import |
| **RO (Retail Outlet)** | `/login` → `/ro` | RO Dashboard | Verify sites, assign meters, create contractors |
| **Consumer** | `/login` → `/consumer` | Consumer Dashboard | Upload photo, verify location, track status |
| **Contractor** | `/login` → `/contractor` | Contractor Dashboard | Perform installations |

### Data Isolation
- **Super Admin**: Sees Admins they created
- **Admin**: Sees only ROs they created and their data
- **RO**: Sees only their meters, consumers, contractors
- **Contractor**: Sees only jobs assigned to them
- **Consumer**: Sees only their own sites

### Test Credentials
```
Super Admin: superadmin@mescom.gov / admin123
Admin:       admin@mescom.gov / admin123
RO:          ro@mescom.gov / retail123
Contractor:  contractor@example.com / contractor123
Consumer:    (any phone number) / OTP: 123456
```

### Sample Meters (for manual entry in Assign Meter page)
```
SM-10001 - THREE_PHASE - AVAILABLE
SM-10002 - SINGLE_PHASE - AVAILABLE
SM-10003 - SINGLE_PHASE - AVAILABLE
```

---

## Complete Installation Flow (6 Steps per REQUIREMENT.md)

### Step 1: Consumer Registration & Data Import
**Actor:** Admin  
**Location:** `/admin/import`

```
Admin uploads Excel file with consumer data
         │
         ▼
┌─────────────────────────────────────┐
│  Excel Columns:                     │
│  - Consumer Number → consumerId     │
│  - Name → name                      │
│  - Phone → phone                    │
│  - Email → email                    │
│  - Address → addressLine1           │
│  - City → city                      │
│  - District → district              │
│  - Pincode → pincode                │
│  - Tariff Code → tariffCode         │
└─────────────────────────────────────┘
         │
         ▼
Backend creates Consumer + Site records
Site State: CREATED
```

**Backend Flow:**
```
Frontend: importConsumers mutation
    ↓
excel.service.ts → parseExcel()
    ↓
imports.service.ts → processConsumerImport()
    ↓
consumers.service.ts → create() 
    ↓
sites.service.ts → create()
```

---

### Step 2: Consumer Site Verification
**Actor:** Consumer  
**Location:** `/consumer/sites/[siteId]`

```
Consumer logs in and sees their site
         │
         ▼
┌─────────────────────────────────────┐
│  Consumer Actions:                  │
│  1. Upload site photo (camera)      │
│  2. Capture GPS location            │
│  3. Submit for verification         │
└─────────────────────────────────────┘
         │
         ▼
Site State: SITE_PENDING
```

**Frontend Components:**
- `GeoPhotoCapture.tsx` - Camera + GPS capture
- `useCamera.ts` - Camera hook
- `useGeoLocation.ts` - GPS hook

**GraphQL Mutations:**
```graphql
mutation UploadSitePhoto($id: ID!, $photo: Upload!) {
  uploadSitePhoto(id: $id, photo: $photo) { ... }
}

mutation UpdateSiteLocation($id: ID!, $input: UpdateSiteLocationInput!) {
  updateSiteLocation(id: $id, input: $input) { ... }
}

mutation SubmitSiteForVerification($id: ID!) {
  submitSiteForVerification(id: $id) { ... }
}
```

---

### Step 3: RO Site Review & Verification
**Actor:** RO (Retail Outlet)  
**Location:** `/ro/site-review/[siteId]`

```
RO reviews submitted sites
         │
         ▼
┌─────────────────────────────────────┐
│  RO Verification Form:              │
│  - Site Accessibility ✓             │
│  - Electrical Readiness ✓           │
│  - Network Provider (Airtel/Jio/..) │
│  - Required Phase (1P/3P)           │
│  - Voltage (240V/415V)              │
│  - Estimated Installation Date      │
│  - Number of Connections            │
│  - Connection Details               │
│  - Additional Notes                 │
│  - All Data Verified ✓              │
│  - Partial Data Verified ✓          │
│  - Sent for Updation ✓              │
└─────────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
 APPROVE    REJECT
    │         │
    ▼         ▼
SITE_VERIFIED  FAILED
(Tariff Locked)
```

**GraphQL Mutation:**
```graphql
mutation VerifySiteWithDetails($id: ID!, $input: VerifySiteInput!) {
  verifySiteWithDetails(id: $id, input: $input) {
    id
    siteState
    networkProvider
    numberOfConnections
    allDataVerified
    ...
  }
}
```

---

### Step 4: Meter Assignment
**Actor:** RO  
**Location:** `/ro/assign`

```
RO assigns meter from inventory to site
         │
         ▼
┌─────────────────────────────────────┐
│  Select:                            │
│  - Site (verified sites only)       │
│  - Meter (from RO's inventory)      │
│  - Contractor (for installation)    │
└─────────────────────────────────────┘
         │
         ▼
Site State: METER_ASSIGNED
Creates: site_meter_connection record
```

**GraphQL Mutation:**
```graphql
mutation AssignMeter($siteId: ID!, $meterId: ID!, $contractorId: ID) {
  assignMeter(siteId: $siteId, meterId: $meterId, contractorId: $contractorId) {
    id
    siteState
  }
}
```

---

### Step 5: Installation by Contractor
**Actor:** Contractor  
**Location:** `/contractor/installations/[id]`

```
Contractor performs physical installation
         │
         ▼
┌─────────────────────────────────────┐
│  Contractor uploads evidence:       │
│  - Photo of installed meter         │
│  - Photo of meter reading           │
│  - Photo of seal                    │
│  - GPS verification                 │
└─────────────────────────────────────┘
         │
         ▼
Site State: INSTALLED
```

**GraphQL Mutation:**
```graphql
mutation UploadInstallationEvidence($input: UploadEvidenceInput!) {
  uploadInstallationEvidence(input: $input) {
    id
    evidenceType
    photoUrl
  }
}

mutation CompleteConnectionInstallation($connectionId: ID!) {
  completeConnectionInstallation(connectionId: $connectionId) {
    id
    status
  }
}
```

---

### Step 6: Verification & Billing Handoff
**Actor:** RO/Admin  
**Location:** `/ro/verify` or `/admin/billing`

```
RO/Admin verifies installation
         │
         ▼
┌─────────────────────────────────────┐
│  Verification checks:               │
│  - Review installation photos       │
│  - Verify meter reading             │
│  - Confirm seal integrity           │
│  - Check GPS matches site           │
└─────────────────────────────────────┘
         │
         ▼
Site State: VERIFIED
         │
         ▼
Billing system handoff
Site State: BILLED
```

---

## Site State Machine

```
CREATED
    │
    ▼ (Consumer uploads photo/location)
SITE_PENDING
    │
    ▼ (RO verifies)
SITE_VERIFIED  ◄─── Ready for Scheduling!
    │               Go to /ro/schedule
    ▼ (RO contacts consumer & schedules)
INSTALLATION_SCHEDULED  ◄─── Ready for Meter Assignment!
    │                        Go to /ro/assign
    ▼ (RO assigns meter)
METER_ASSIGNED
    │
    ▼ (Contractor starts work)
INSTALLATION_IN_PROGRESS
    │
    ▼ (Contractor completes)
INSTALLED
    │
    ▼ (RO/Admin verifies)
VERIFICATION_PENDING
    │
    ▼ (Approved)
VERIFIED
    │
    ▼ (Billing handoff)
BILLED

FAILED ◄─── Can happen at any stage
```

### RO Navigation Flow After Each Action

| After Action | Current State | Next Page | Purpose |
|--------------|---------------|-----------|---------|
| Site Verified | SITE_VERIFIED | `/ro/schedule` | Contact consumer & schedule appointment |
| Appointment Scheduled | INSTALLATION_SCHEDULED | `/ro/assign` | Assign meter to site |
| Meter Assigned | METER_ASSIGNED | Contractor takes over | Contractor performs installation |
| Installation Complete | INSTALLED | `/ro/verify` | Verify installation |
| Installation Verified | VERIFIED | Billing handoff | Complete process |

---

## Frontend Directory Structure

```
/Mescom/src/
├── app/
│   ├── (admin)/           # Admin pages (Super Admin & Admin)
│   │   └── admin/
│   │       ├── page.tsx           # Dashboard
│   │       ├── admins/page.tsx    # Super Admin → Manage Admins
│   │       ├── ros/page.tsx       # Admin → Manage ROs
│   │       ├── master/page.tsx    # Master Data (Meters, ROs, Contractors)
│   │       ├── import/page.tsx    # Excel import
│   │       └── consumers/         # Consumer management
│   │
│   ├── (ro)/              # Retail Outlet pages
│   │   └── ro/
│   │       ├── page.tsx           # Dashboard
│   │       ├── site-review/       # Site verification
│   │       │   ├── page.tsx       # List sites to review
│   │       │   └── [siteId]/      # Individual site review
│   │       ├── assign/page.tsx    # Meter assignment
│   │       ├── inventory/         # Meter inventory
│   │       └── consumers/         # Consumer list
│   │
│   ├── (consumer)/        # Consumer pages
│   │   └── consumer/
│   │       ├── page.tsx           # Dashboard
│   │       └── sites/[siteId]/    # Site photo/location upload
│   │
│   └── (contractor)/      # Contractor pages
│       └── contractor/
│           └── installations/     # Installation tasks
│
├── graphql/               # GraphQL queries/mutations
│   ├── auth.ts
│   ├── consumers.ts
│   ├── sites.ts           # All site-related operations
│   ├── meters.ts
│   └── installations.ts
│
├── components/
│   ├── ui/                # Reusable UI components
│   └── shared/            # Shared components
│       ├── GeoPhotoCapture.tsx
│       ├── QRScanner.tsx
│       └── Timeline.tsx
│
└── lib/
    ├── apollo/            # Apollo Client setup
    └── auth/              # Auth provider
```

---

## Backend Directory Structure

```
/mescom-backend/src/
├── modules/
│   ├── auth/              # Authentication
│   │   ├── auth.resolver.ts
│   │   └── auth.service.ts
│   │
│   ├── consumers/         # Consumer management
│   │   ├── consumers.resolver.ts
│   │   ├── consumers.service.ts
│   │   └── consumers.types.ts
│   │
│   ├── sites/             # Site management (CORE)
│   │   ├── sites.resolver.ts
│   │   ├── sites.service.ts    # State machine logic
│   │   └── sites.types.ts
│   │
│   ├── meters/            # Meter inventory
│   │   ├── meters.resolver.ts
│   │   └── meters.service.ts
│   │
│   ├── installations/     # Installation tracking
│   │   ├── installations.resolver.ts
│   │   └── installations.service.ts
│   │
│   ├── site-meter-connections/  # Meter-Site assignments
│   │   ├── site-meter-connections.resolver.ts
│   │   └── site-meter-connections.service.ts
│   │
│   └── imports/           # Excel import
│       ├── imports.resolver.ts
│       ├── imports.service.ts
│       └── excel.service.ts
│
├── database/
│   ├── schema/index.ts    # Drizzle schema definitions
│   └── migrations/        # Database migrations
│
└── common/
    ├── guards/            # Auth guards
    ├── decorators/        # @Roles, @CurrentUser
    └── enums/             # UserRole, SiteState, etc.
```

---

## Key GraphQL Operations

### Queries
```graphql
# Get sites for review (RO)
query GetSites($filter: SitesFilterInput, $pagination: SitesPaginationInput) {
  sites(filter: $filter, pagination: $pagination) {
    items { id, siteId, siteState, ... }
    total
  }
}

# Get single site details
query GetSite($id: ID!) {
  site(id: $id) { ... }
}

# Get consumer details
query GetConsumer($id: ID!) {
  consumer(id: $id) { id, name, phone, gstn, ... }
}
```

### Mutations
```graphql
# Import consumers from Excel
mutation ImportConsumers($file: Upload!, $roId: ID) {
  importConsumers(file: $file, roId: $roId) { ... }
}

# Consumer uploads photo
mutation UploadSitePhoto($id: ID!, $photo: Upload!) {
  uploadSitePhoto(id: $id, photo: $photo) { ... }
}

# Consumer updates location
mutation UpdateSiteLocation($id: ID!, $input: UpdateSiteLocationInput!) {
  updateSiteLocation(id: $id, input: $input) { ... }
}

# RO verifies site with all details
mutation VerifySiteWithDetails($id: ID!, $input: VerifySiteInput!) {
  verifySiteWithDetails(id: $id, input: $input) { ... }
}

# RO updates site fields (post-verification)
mutation UpdateSiteROFields($id: ID!, $input: UpdateSiteROFieldsInput!) {
  updateSiteROFields(id: $id, input: $input) { ... }
}

# Assign meter to site
mutation AssignMeter($siteId: ID!, $meterId: ID!, $contractorId: ID) {
  assignMeter(siteId: $siteId, meterId: $meterId, contractorId: $contractorId) { ... }
}
```

---

## RO Document Fields (Editable by RO)

| Field | Database Column | Location |
|-------|-----------------|----------|
| Customer Name | consumers.name | Display only |
| Customer Phone | consumers.phone | Display only |
| Customer Address | sites.addressLine1, city, etc. | Display only |
| GSTN | consumers.gstn | Editable |
| Site Readiness | sites.siteReadinessStatus | Set during verification |
| Number of Connections | sites.numberOfConnections | Editable |
| Connection Details | sites.connectionDetails | Editable |
| Contractor Name | contractors.name | Via assignment |
| Contractor Phone | contractors.phone | Via assignment |
| Network Provider | sites.networkProvider | Editable |
| Expected Installation Date | sites.estimatedInstallationDate | Editable |
| Photo of Location | sites.sitePhotoUrl | Consumer uploads |
| Additional Info | sites.additionalNotes | Editable |
| All Data Verified | sites.allDataVerified | Checkbox |
| Partial Data Verified | sites.partialDataVerified | Checkbox |
| Sent for Updation | sites.sentForUpdation | Checkbox |
| WhatsApp Confirmation | sites.whatsappConfirmationSentAt | Auto-set |

---

## Running the Application

### Backend
```bash
cd mescom-backend
npm install
npm run start:dev
# Runs on http://localhost:3001
# GraphQL Playground: http://localhost:3001/graphql
```

### Frontend
```bash
cd Mescom
npm install
npm run dev
# Runs on http://localhost:3000
```

### Database
```bash
# PostgreSQL connection
postgres://praharsh@localhost:5432/mescom

# Push schema changes
cd mescom-backend
npx drizzle-kit push
```

---

## File Upload Flow

```
Frontend (apollo-upload-client)
         │
         │ multipart/form-data
         ▼
Backend (graphqlUploadExpress middleware)
         │
         ▼
files.service.ts → saves to /uploads/
         │
         ▼
Returns file URL → stored in database
```

**Configuration:**
- Frontend: `/Mescom/src/lib/apollo/client.ts` uses `createUploadLink`
- Backend: `/mescom-backend/src/main.ts` uses `graphqlUploadExpress`

---

## Common Issues & Solutions

### 1. "Upload value invalid"
**Cause:** Apollo Client not configured for file uploads  
**Solution:** Use `createUploadLink` from `apollo-upload-client`

### 2. Excel import columns not mapping
**Cause:** Column names don't match expected format  
**Solution:** Check `/mescom-backend/src/modules/imports/excel.service.ts` for column mappings

### 3. Site states not transitioning
**Cause:** Missing required data (photo, location)  
**Solution:** Ensure consumer has uploaded photo and location before RO verification

### 4. TypeScript errors for apollo-upload-client
**Cause:** Missing type declarations  
**Solution:** Created `/Mescom/src/types/apollo-upload-client.d.ts`

---

## Database Schema (Key Tables)

```sql
-- consumers
id, consumerId, name, email, phone, gstn, ...

-- consumer_sites
id, siteId, consumerId, roId, 
addressLine1, city, district, pincode,
latitude, longitude, sitePhotoUrl,
siteState, siteReadinessStatus,
numberOfConnections, connectionDetails,
allDataVerified, partialDataVerified, ...

-- meters
id, meterNumber, phase, status, roId, ...

-- site_meter_connections
id, siteId, meterId, contractorId, status, ...

-- installations
id, connectionId, status, completedAt, ...

-- installation_evidence
id, installationId, evidenceType, photoUrl, ...
```

---

---

## API Documentation

For external integrations, see:
- [Consumer API Documentation](CONSUMER_API_DOCS.md) - OTP auth, site queries, mutations
- [Contractor API Documentation](CONTRACTOR_API_DOCS.md) - Job queries, installation mutations

---

## Recent Changes (January 28, 2026)

### Role Hierarchy Implementation
- Super Admin → Admin → RO → Contractor/Consumer
- Each level can only create users at the next level down

### Data Isolation
- All queries now filter by creator/owner
- Users only see data they created or that belongs to them
- Implemented in: users, meters, consumers, retail-outlets services

### Auto-Assign Location
- When RO creates a contractor, the contractor automatically inherits the RO's division
- No manual location selection needed

### New Admin Pages
- `/admin/admins` - Super Admin creates/manages Admins
- `/admin/ros` - Admin creates/manages Retail Outlets
- `/admin/master` - Real-time master data dashboard

---

*Last Updated: January 28, 2026*
