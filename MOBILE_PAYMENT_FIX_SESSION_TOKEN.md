# Mobile Payment Error Fix - "token is not present"

**Date:** March 11, 2026  
**Issue:** Mobile app receiving `order_token_invalid` error when initiating subscription payment  
**Status:** ✅ **FIXED**

---

## Problem Analysis

### Error Details:
```json
{
  "status": "FAILED",
  "message": "token is not present",
  "code": "order_token_invalid",
  "type": "request_failed"
}
```

### Root Cause:
The backend was creating a Cashfree order successfully and storing the `payment_session_id` in the database, but **NOT returning it to the mobile app** in the GraphQL response.

The mobile app received only:
```json
{
  "paymentIntent": {
    "id": "e75f8e55-b4d4-4c3d-a343-c24ed9366d43",
    "status": "PENDING",
    "amount": 1300,
    "duration": "QUARTERLY"
    // ❌ Missing: sessionId
  }
}
```

When the mobile app tried to use the payment intent `id` as the order token with Cashfree SDK, it failed because Cashfree needs the `payment_session_id` (not the database payment ID).

---

## Solution Implemented

### 1. Updated GraphQL Type
**File:** `user-service/src/modules/marketplace/subscriptions/subscriptions.types.ts`

Added `sessionId` field to `SubscriptionPaymentIntent`:

```typescript
@ObjectType()
export class SubscriptionPaymentIntent {
  @Field(() => ID)
  id: string;

  @Field(() => ID)
  contractorId: string;

  @Field(() => SubscriptionDuration)
  duration: SubscriptionDuration;

  @Field(() => Float)
  amount: number;

  planId?: string;
  planVersion?: number;

  @Field(() => SubscriptionPaymentStatus)
  status: SubscriptionPaymentStatus;

  @Field({ nullable: true })
  paymentLink?: string;

  @Field({ nullable: true })
  orderId?: string;

  @Field({ nullable: true, description: 'Cashfree payment session ID for checkout' })
  sessionId?: string; // ✅ NEW FIELD

  @Field()
  expiresAt: Date;

  @Field()
  createdAt: Date;
}
```

### 2. Updated Service to Return Session ID
**File:** `user-service/src/modules/marketplace/subscriptions/subscriptions.service.ts`

Modified `initiateUpgrade()` method:

```typescript
const paymentIntent: SubscriptionPaymentIntent = {
  id: paymentRecord.id,
  contractorId,
  duration: input.duration,
  amount: plan.finalPrice,
  planId: 'id' in plan ? plan.id : undefined,
  planVersion: 'version' in plan ? plan.version : undefined,
  status: SubscriptionPaymentStatus.PENDING,
  paymentLink: undefined,
  orderId: gatewayOrderId,
  sessionId: cfOrder.payment_session_id, // ✅ NOW INCLUDED
  expiresAt: new Date(Date.now() + 30 * 60 * 1000),
  createdAt: new Date(),
};
```

### 3. Verified Bank Verification V2 Code
**File:** `user-service/src/modules/payments/cashfree.service.ts`

Bank verification endpoint updated to V2 sync:
- ✅ No TypeScript errors
- ✅ Proper ES module import (`import axios from 'axios'`)
- ✅ Correct endpoint: `/verification/bankaccount/v2/sync`
- ✅ Response parsing handles both snake_case and camelCase
- ✅ Error handling for 400/401 status codes

---

## How It Works Now

### Backend Flow:
1. Mobile app calls `initiateSubscriptionUpgrade` mutation
2. Backend creates payment record in database
3. Backend calls Cashfree API to create order
4. Cashfree returns `cf_order_id` and `payment_session_id`
5. Backend stores `payment_session_id` in `payments.gatewaySessionId`
6. **Backend now returns `sessionId` in GraphQL response** ✅

### Mobile App Flow:
1. Receives response with `sessionId`
2. Initializes Cashfree SDK with `sessionId`
3. Opens payment checkout
4. User completes payment
5. Cashfree sends webhook to backend
6. Backend updates payment status and subscription

---

## Testing Instructions

### 1. Restart Backend
```bash
cd user-service
pnpm run dev
```

### 2. Test Subscription Upgrade from Mobile
```graphql
mutation {
  initiateSubscriptionUpgrade(input: {
    duration: QUARTERLY
    bankAccount: {
      accountNumber: "123456789012"
      ifscCode: "SBIN0001234"
      accountHolderName: "Ravi Kumar"
      bankName: "State Bank of India"
    }
  }) {
    success
    message
    paymentIntent {
      id
      status
      amount
      duration
      sessionId  # ✅ NOW AVAILABLE
      orderId
    }
  }
}
```

### 3. Mobile App Should Now Receive:
```json
{
  "paymentIntent": {
    "id": "e75f8e55-b4d4-4c3d-a343-c24ed9366d43",
    "status": "PENDING",
    "amount": 1300,
    "duration": "QUARTERLY",
    "sessionId": "session_abc123...", // ✅ Cashfree token
    "orderId": "SUB_1773213000_12345678"
  }
}
```

### 4. Initialize Cashfree in Mobile App
```kotlin
// Android (Kotlin)
val cfSession = CFSession.CFSessionBuilder()
    .setPaymentSessionId(paymentIntent.sessionId) // ✅ Use sessionId
    .setEnvironment(CFSession.Environment.SANDBOX)
    .build()

cfPaymentGateway.doPayment(cfSession)
```

```swift
// iOS (Swift)
let session = CFSession(
    sessionId: paymentIntent.sessionId, // ✅ Use sessionId
    environment: .sandbox,
    orderAmount: paymentIntent.amount
)

CFPaymentGateway.shared.doPayment(session)
```

---

## Comparison: Consumer vs Contractor Flows

### Consumer Job Payment (Already Working ✅)
**Mutation:** `initiatePayment(jobId: ID!)`  
**Returns:** `InitiatePaymentResult` with `sessionId` field  
**Status:** Already included session ID - no fix needed

### Contractor Subscription Upgrade (Now Fixed ✅)
**Mutation:** `initiateSubscriptionUpgrade(input: InitiateSubscriptionUpgradeInput!)`  
**Returns:** `SubscriptionUpgradeResponse` with `paymentIntent.sessionId`  
**Status:** Fixed - now includes session ID

---

## Cashfree Integration Summary

### Backend Components:
1. **CashfreeService** - Wraps Cashfree SDK, creates orders
2. **PaymentsService** - Manages consumer job payments
3. **SubscriptionsService** - Manages contractor subscription upgrades
4. **Bank Verification** - Uses Cashfree V2 sync endpoint

### API Endpoints Used:
- **Payment Orders:** `POST /pg/orders` (via SDK)
- **Bank Verification:** `POST /verification/bankaccount/v2/sync` (direct HTTP)

### Environment Selection:
- `NODE_ENV=development` → Sandbox (`https://sandbox.cashfree.com`)
- `NODE_ENV=production` → Production (`https://api.cashfree.com`)

### Credentials Required:
- `CASHFREE_APP_ID` - TEST prefix for sandbox
- `CASHFREE_SECRET` - Secret key for authentication
- API Version: `2023-08-01`

---

## Next Steps

1. **Test on Mobile App:**
   - Update GraphQL schema in mobile app (regenerate types)
   - Use `paymentIntent.sessionId` instead of `paymentIntent.id`
   - Test payment flow end-to-end

2. **Verify Webhook Processing:**
   - Ensure webhook endpoint is configured in Cashfree dashboard
   - Test payment success and failure scenarios
   - Verify subscription activation after successful payment

3. **Production Readiness:**
   - Generate production Cashfree credentials
   - Update `.env` with production credentials
   - Set `NODE_ENV=production`
   - Configure production webhook URL

---

## Common Errors & Solutions

### Error: "token is not present"
**Cause:** Using payment ID instead of session ID  
**Fix:** Use `paymentIntent.sessionId` from GraphQL response ✅

### Error: 401 Authentication Failed
**Cause:** Invalid or expired Cashfree credentials  
**Fix:** Regenerate credentials from Cashfree dashboard

### Error: "Phone number is required"
**Cause:** User profile missing phone number  
**Fix:** Ensure user has valid phone in database

### Error: Order already processed
**Cause:** Trying to pay for same order twice  
**Fix:** Create new payment intent for retry

---

## Documentation References

- [Cashfree Payment Gateway Docs](https://docs.cashfree.com/docs/payment-gateway)
- [Cashfree Android SDK](https://docs.cashfree.com/docs/android-sdk)
- [Cashfree iOS SDK](https://docs.cashfree.com/docs/ios-sdk)
- [Bank Verification V2 API](https://docs.cashfree.com/reference/pgverifyupi-v2)

