# Marketplace Consumer Eligibility & Job Booking Flow

## Overview
This document explains the marketplace eligibility feature for consumers and the job booking flow in the mobile app.

## Consumer Eligibility Rules

### Who Can Access Marketplace?

**✅ ELIGIBLE FOR MARKETPLACE:**
1. **Self-Registered Consumers** (via mobile app)
   - `selfRegistered = true` in database
   - No admin-uploaded data linkage
   - Can access marketplace features immediately after registration

2. **Admin-Imported Consumers WITHOUT Hierarchy Data**
   - `selfRegistered = false` but no `circleId`, `divisionId`, or `subdivisionId`
   - Rare case, but possible
   - These consumers are not part of existing DISCOM system

**❌ NOT ELIGIBLE FOR MARKETPLACE:**
- **Admin-Imported Consumers WITH Hierarchy Data**
  - `selfRegistered = false`
  - Has `circleId`, `divisionId`, or `subdivisionId` set
  - These are existing DISCOM customers managed through traditional system
  - Should use traditional meter installation/replacement flow

### Implementation

#### Database Schema
```sql
-- consumers table already has the field
selfRegistered BOOLEAN NOT NULL DEFAULT false
```

#### GraphQL Type
```graphql
type Consumer {
  id: ID!
  consumerId: String!
  name: String!
  # ... other fields ...
  
  # NEW FIELD - computed dynamically
  isMarketplaceEligible: Boolean!
}
```

#### Calculation Logic
Location: `ConsumersService.calculateMarketplaceEligibility()`

```typescript
private calculateMarketplaceEligibility(consumer: any): boolean {
  // Self-registered consumers are always marketplace-eligible
  if (consumer.selfRegistered === true) {
    return true;
  }

  // Admin-imported consumers with hierarchy data are NOT eligible
  const hasHierarchyData =
    consumer.circleId || consumer.divisionId || consumer.subdivisionId;

  if (hasHierarchyData) {
    return false;
  }

  // Admin-imported consumer with NO hierarchy data = marketplace eligible
  return true;
}
```

## API: Get Consumer Profile with Eligibility

### Query: `myConsumer`
**Role Required:** `CONSUMER` (authenticated)

**GraphQL Query:**
```graphql
query GetMyProfile {
  myConsumer {
    id
    consumerId
    name
    email
    phone
    state
    isMarketplaceEligible  # ← NEW FIELD
    billingAddressLine1
    billingAddressLine2
    billingCity
    billingDistrict
    billingState
    billingPincode
    createdAt
  }
}
```

**Response Example (Self-Registered Consumer):**
```json
{
  "data": {
    "myConsumer": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "consumerId": "CONS-2024-0001",
      "name": "Rajesh Kumar",
      "email": "rajesh@example.com",
      "phone": "+919876543210",
      "state": "ACTIVE",
      "isMarketplaceEligible": true,  // ← Can access marketplace
      "billingCity": "Bangalore",
      "billingState": "Karnataka",
      "createdAt": "2024-03-10T10:30:00Z"
    }
  }
}
```

**Response Example (Admin-Imported Consumer with Hierarchy):**
```json
{
  "data": {
    "myConsumer": {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "consumerId": "CONS-ADMIN-0001",
      "name": "Suresh Patel",
      "phone": "+919123456789",
      "state": "ACTIVE",
      "isMarketplaceEligible": false,  // ← Cannot access marketplace
      "billingCity": "Mysore",
      "billingState": "Karnataka",
      "createdAt": "2024-01-15T08:00:00Z"
    }
  }
}
```

## Job Booking Flow (Marketplace)

### Prerequisites
Before a consumer can book a job, they need:

1. **Consumer Profile** - Must have valid consumer account
2. **Marketplace Eligibility** - `isMarketplaceEligible = true`
3. **Authentication** - Valid JWT token with `CONSUMER` role
4. **Payment Details** - Phone number for payment processing

### Step 1: Check Marketplace Eligibility
**Mobile App Logic:**
```typescript
const { data } = await apolloClient.query({
  query: GET_MY_CONSUMER_PROFILE
});

if (!data?.myConsumer?.isMarketplaceEligible) {
  // Show message: "Marketplace not available for your account"
  // Redirect to traditional flow or home screen
  return;
}

// Proceed to marketplace
navigateToMarketplace();
```

### Step 2: Browse Contractors & Services
Consumer can:
- View active contractors in marketplace
- See services offered by each contractor
- Check service pricing and ratings
- Filter by location

**Relevant APIs:**
- `contractors(filters: { marketplaceActive: true })`
- `contractorServices(contractorId: ID!)`
- `locationMaster` (for area-based pricing)

### Step 3: Create Job (Book Service)

#### API: `createMarketplaceJob`
**Role Required:** `CONSUMER`

**GraphQL Mutation:**
```graphql
mutation BookJob($input: CreateJobInput!) {
  createMarketplaceJob(input: $input) {
    jobId
    jobNumber
    amount
    paymentSessionId  # For Cashfree payment
    paymentUrl
  }
}
```

**Input Type:**
```graphql
input CreateJobInput {
  contractorId: ID!        # Selected contractor
  serviceId: ID!           # Selected service (e.g., "Meter Installation")
  locationId: ID!          # Consumer's location for pricing
  quantity: Int            # Default: 1
  consumerAddress: String  # Optional: delivery/service address
  consumerPhone: String    # Optional: contact phone
}
```

**Required Details from Consumer:**
1. **Contractor Selection** - Which contractor to hire
2. **Service Selection** - What service (installation, repair, etc.)
3. **Location** - For area-based pricing calculation
4. **Quantity** (Optional) - Default is 1
5. **Address** (Optional) - Service location if different from billing
6. **Phone** (Optional) - Contact number if different from profile

#### Example Mobile App Request:
```typescript
const { data } = await apolloClient.mutate({
  mutation: CREATE_MARKETPLACE_JOB,
  variables: {
    input: {
      contractorId: selectedContractor.id,
      serviceId: selectedService.id,
      locationId: userLocation.id,
      quantity: 1,
      consumerAddress: "123 MG Road, Bangalore",
      consumerPhone: "+919876543210"
    }
  }
});

const { jobId, paymentSessionId, amount } = data.createMarketplaceJob;
```

### Step 4: Payment Flow

After job creation, the response includes:
- `paymentSessionId` - Cashfree session token
- `amount` - Total amount to pay
- `jobId` - Job reference

**Mobile App Payment Integration:**
```typescript
// Android (Kotlin)
val cfSession = CFSession(
  paymentSessionId = paymentSessionId,
  orderId = jobId,
  environment = CFEnvironment.SANDBOX
)

CFPaymentGatewayService.getInstance().doPayment(
  activity,
  cfPaymentComponent
)

// iOS (Swift)
let cfSession = CFSession(
  sessionId: paymentSessionId,
  orderId: jobId,
  environment: .sandbox
)

CFPaymentGatewayService.getInstance().doPayment(
  session: cfSession,
  callback: self
)
```

**Payment Success → Job Status Updates to `PENDING_ACCEPTANCE`**

### Step 5: Job Lifecycle

After successful payment:
1. **Job Status:** `PENDING_ACCEPTANCE`
2. Contractor receives notification
3. Contractor can accept/reject job
4. If accepted → Status: `ACCEPTED` → Work begins
5. Contractor submits OTP after completion
6. Consumer verifies OTP
7. Status: `COMPLETED` → Payment released to contractor

## Mobile App Implementation Checklist

### 1. Profile Screen
- [ ] Fetch `myConsumer` query with `isMarketplaceEligible`
- [ ] Store eligibility in local state
- [ ] Show/hide marketplace navigation based on eligibility

### 2. Marketplace Access Gate
```typescript
// Before allowing marketplace access
if (!consumer?.isMarketplaceEligible) {
  showAlert({
    title: "Marketplace Unavailable",
    message: "Your account is managed through the traditional DISCOM system. Please contact your subdivision office for meter services.",
    actions: ["OK"]
  });
  return;
}
```

### 3. Job Booking Screen
**Required UI Fields:**
- Contractor selection (dropdown/list)
- Service selection (dropdown/list)
- Location selection (from location master)
- Address input (text field, optional)
- Phone number (text field, pre-filled from profile)
- Price display (calculated automatically)

**Validation:**
- All required fields must be filled
- Validate phone number format
- Verify user has marketplace eligibility before submission

### 4. Payment Integration
- Use Cashfree SDK for Android/iOS
- Handle payment success/failure callbacks
- Update job status after payment
- Show confirmation screen with job number

## Error Handling

### Common Errors

**1. Consumer Not Marketplace Eligible**
```json
{
  "errors": [{
    "message": "You are not eligible for marketplace services. Please contact your subdivision office.",
    "extensions": {
      "code": "MARKETPLACE_NOT_ELIGIBLE"
    }
  }]
}
```

**2. Consumer Not Found**
```json
{
  "errors": [{
    "message": "Consumer profile not found",
    "extensions": {
      "code": "CONSUMER_NOT_FOUND"
    }
  }]
}
```

**3. Invalid Contractor/Service**
```json
{
  "errors": [{
    "message": "Contractor not found or inactive",
    "extensions": {
      "code": "CONTRACTOR_NOT_FOUND"
    }
  }]
}
```

## Testing

### Test Scenarios

1. **Self-Registered Consumer**
   - Create consumer via mobile app registration
   - Verify `isMarketplaceEligible = true`
   - Able to create jobs successfully

2. **Admin-Imported with Hierarchy**
   - Admin imports consumer with circle/division/subdivision
   - Verify `isMarketplaceEligible = false`
   - Job creation should fail with proper error

3. **Admin-Imported without Hierarchy**
   - Admin creates consumer without hierarchy data
   - Verify `isMarketplaceEligible = true`
   - Able to create jobs successfully

## Summary

### What Changed?
✅ Added `isMarketplaceEligible` boolean to Consumer GraphQL type  
✅ Implemented calculation logic based on `selfRegistered` flag and hierarchy data  
✅ Updated all consumer service methods to include eligibility  
✅ No database migration needed - uses existing `selfRegistered` field  

### What Mobile App Needs to Do?
1. Update GraphQL schema to include `isMarketplaceEligible` field
2. Fetch this field in `myConsumer` query
3. Gate marketplace access based on eligibility
4. Show appropriate message if not eligible

### Existing Job Booking APIs
✅ `createMarketplaceJob` mutation already exists  
✅ Accepts all required fields (contractor, service, location)  
✅ Returns payment session ID for Cashfree  
✅ No changes needed to job booking flow  

### Required Details for Job Booking
- Contractor ID (user selects from list)
- Service ID (user selects service type)
- Location ID (for pricing calculation)
- Address (optional, for service location)
- Phone (optional, defaults to consumer profile phone)
