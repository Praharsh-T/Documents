# Cashfree Payment Integration: Complete Technical Reference

> **Last Updated:** 10 March 2026 — Reflects current production-ready code after full integration + bug fixes.

---

## 1. Overview

The platform has **two completely separate payment flows**, both using the Cashfree PG SDK (`cashfree-pg`) and sharing the same `mp_payments` DB table and webhook endpoint.

| Flow | Who pays | What they pay for | Trigger mutation |
|---|---|---|---|
| **Job Payment** | Consumer | A marketplace service job | `initiateMarketplacePayment` |
| **Subscription Payment** | Contractor | PREMIUM plan upgrade | `initiateSubscriptionUpgrade` |

Both flows follow the same 3-phase lifecycle:

```
INTENT (sync)          INTERACTION (client-side)       RESOLUTION (async)
─────────────────      ─────────────────────────       ──────────────────────────
GraphQL mutation  →    Cashfree SDK overlay/checkout → POST /payments/cashfree/webhook
creates mp_payments    (backend is idle)                flips status + triggers side-effects
row as PENDING,
returns sessionId
```

---

## 2. Architecture — Files and Responsibilities

```
PaymentsModule
├── cashfree.service.ts          — Thin SDK wrapper (createOrder, verifyWebhook, createRefund)
├── cashfree-webhook.controller.ts — REST controller for Cashfree callbacks (POST /payments/cashfree/webhook)
├── payments.service.ts          — Orchestrates job payments, webhook processing, refunds
├── payments.resolver.ts         — GraphQL mutations/queries exposed to frontend
├── payments.types.ts            — GraphQL ObjectTypes, enums, InputTypes
└── payments.module.ts           — NestJS module wiring (forwardRef with SubscriptionsModule)

SubscriptionsModule
└── subscriptions.service.ts     — Handles subscription upgrade payment intent + finalizeUpgrade transaction
```

---

## 3. Environment Setup

```env
# Required in all environments
CASHFREE_APP_ID=TEST109944128e970adfb7a77d02c31a21449901   # Use your TEST key for sandbox
CASHFREE_SECRET=cfsk_ma_test_9913b30e69050ef6072b6a1d3ad58137_720892f6

# NODE_ENV controls Cashfree environment automatically:
# development/staging → CFEnvironment.SANDBOX
# production         → CFEnvironment.PRODUCTION

NODE_ENV=development
```

**Important:** The service reads `CASHFREE_SECRET` (not `CASHFREE_SECRET_KEY`). If either key is missing, `CashfreeService.onModuleInit()` logs a warning and **every subsequent API call will throw `500 InternalServerErrorException`** with a clear message instead of a silent `TypeError`.

---

## 4. Database Schema — `mp_payments` & `mp_refunds`

**`mp_payments`** (`user-service/src/database/schema/marketplace/payments.ts`):

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | Auto-generated |
| `job_id` | uuid FK → mp_jobs | Nullable — NULL for SUBSCRIPTION payments |
| `subscription_id` | uuid | Nullable — for future sub-linkage |
| `user_id` | uuid FK → **users** | Always the `users.id` (NOT `contractors.id`) |
| `plan_id` | uuid | Snapshot of subscription plan at purchase |
| `plan_version` | int | Version of plan at purchase time |
| `payment_for` | enum `mp_payment_for` | `JOB` or `SUBSCRIPTION` |
| `amount` | decimal(10,2) | Price snapshot — never trusted from client |
| `gateway_provider` | varchar | Always `'CASHFREE'` |
| `gateway_order_id` | varchar | e.g. `MP_JOB_<uuid>_<ts>` or `SUB_<ts>_<id8>` |
| `gateway_payment_id` | varchar | Cashfree `cf_payment_id` — stored on webhook |
| `gateway_session_id` | varchar | Ephemeral (30 min TTL) — returned to client |
| `gateway_payment_method` | varchar | `UPI`, `CARD`, `NB`, etc — stored on webhook |
| `status` | enum | `PENDING → SUCCESS / FAILED / REFUNDED` |

**`mp_refunds`** — separate table, only created when admin initiates a refund. Status goes `PENDING → PROCESSING → SUCCESS/FAILED` via refund webhook.

---

## 5. FLOW A: Consumer Job Payment

### Step-by-Step (Consumer)

**Step 1 — Create Job:**
```graphql
mutation {
  createMarketplaceJob(input: {
    contractorId: "<contractor-id>"
    serviceId: "<service-id>"
    locationId: "<location-id>"
    consumerAddress: "123 Main St"
    consumerPhone: "9876543210"
    quantity: 1
  }) {
    jobId
    jobNumber
    amount
    paymentSessionId    # ← Pass this to Cashfree SDK
    paymentUrl          # ← Redirect URL (nullable in SDK v5, use sessionId instead)
  }
}
```

What the backend does:
- Validates consumer, contractor PREMIUM + quota, service pricing, SLA
- Creates `mp_jobs` row with status `PAYMENT_PENDING`
- Calls `PaymentsService.initiatePayment()` → inserts `mp_payments` row as `PENDING` → calls `cashfreeService.createOrder()` → stores `payment_session_id`

**Step 2 — Launch Cashfree Checkout (Client-side):**
```javascript
// Mobile (React Native) — using cashfree-pg-api-contract
const { load } = useCFCheckoutSheet();
await load({ sessionId: paymentSessionId });
```

**Step 3 — Payment completes → Cashfree calls webhook:**

`POST /payments/cashfree/webhook`

Backend processes:
- Verifies HMAC-SHA256 signature
- Looks up `mp_payments` by `gateway_order_id`
- Stores `gatewayPaymentId` + `gatewayPaymentMethod` from payload
- Flips `mp_payments.status → SUCCESS`
- Flips `mp_jobs.status → REQUESTED` + sets `paidAt`
- Job is now live and visible to contractor

**Step 4 (Sandbox only) — Skip payment, simulate:**
```graphql
# SUPER_ADMIN only — blocked in production
mutation {
  simulatePaymentSuccess(jobId: "<job-id>") {
    id
    status  # → REQUESTED
  }
}
```

**Step 5 — Check payment status:**
```graphql
query {
  marketplacePaymentByJob(jobId: "<job-id>") {
    id
    status
    amount
    gatewayPaymentId
    gatewayPaymentMethod
    completedAt
  }
}
```

---

## 6. FLOW B: Contractor Subscription Upgrade

### Step-by-Step (Contractor)

**Step 1 — Initiate upgrade:**
```graphql
mutation {
  initiateSubscriptionUpgrade(input: {
    duration: MONTHLY           # MONTHLY | QUARTERLY | YEARLY
    # bankAccount is optional — only needed if not set yet
    bankAccount: {
      accountNumber: "1234567890"
      ifscCode: "SBIN0001234"
      accountHolderName: "John Contractor"
      bankName: "SBI"
    }
  }) {
    success
    message               # e.g. "Payment intent created for 1 Month Premium Access. Amount: ₹499"
    paymentIntent {
      id                  # paymentIntentId — save this for confirmSubscriptionPayment
      orderId             # Cashfree order ID
      amount              # ₹499 / ₹1347 / ₹4491
      expiresAt           # 30 minutes from now
      status
    }
  }
}
```

> `contractorId` is **never sent by the client** — it's injected from the JWT in the resolver. This is a security requirement.

Business logic guards that run:
- Contractor profile must exist (auto-created as FREE if missing, with a new `qrProfileToken`)
- If already PREMIUM with > 30 days remaining → `ConflictException` (prevents wasteful duplicate purchase)
- Resolves `users.id` from `contractors.userId` for the FK (critical bug fixed 10 Mar 2026)

**Step 2 — Launch Cashfree checkout** using `paymentIntent.orderId` or `sessionId` (same as consumer flow).

**Step 3a — Automatic resolution via webhook (production):**

Same webhook endpoint processes it. On `PAID`:
- Calls `SubscriptionsService.finalizeSubscriptionUpgrade(paymentId)`
- **Atomic DB transaction:**
  1. `mp_payments.status → SUCCESS`
  2. `mp_contractor_profile.subscriptionType → PREMIUM`, `isMarketplaceActive → true`, `jobsUsedThisCycle → 0`
  3. Calculates `endDate` (additive if existing days remain)
  4. Deactivates all previous `mp_contractor_subscriptions` rows
  5. Inserts new history row for invoice trail

**Step 3b — Sync fallback via return URL (when webhook is delayed):**
```graphql
mutation {
  confirmSubscriptionPayment(input: {
    paymentIntentId: "<id from Step 1>"
  }) {
    success
    newSubscriptionType   # PREMIUM
    validTill             # ISO date
  }
}
```

This polls Cashfree's `getOrder()` API to check `order_status`. If `PAID`, calls `finalizeSubscriptionUpgrade` immediately.

**Step 4 — Check subscription status:**
```graphql
query {
  mySubscriptionStatus {
    subscriptionType        # FREE | PREMIUM
    subscriptionValidTill
    isMarketplaceActive
    jobsUsedThisCycle
  }
}
```

---

## 7. Subscription Plans

| Duration | `SubscriptionDuration` enum | Amount | Months |
|---|---|---|---|
| Monthly | `MONTHLY` | ₹499 | 1 |
| Quarterly | `QUARTERLY` | ₹1,347 | 3 (10% off) |
| Yearly | `YEARLY` | ₹4,491 | 12 (25% off) |

Plans are stored in `mp_subscription_plans` and fetched live at payment time. If admin updates pricing, the new price applies immediately to the next purchase.

Optional plan fields (admin-configurable):
- `planName` — custom label e.g. "Starter", "Pro"
- `maxJobsPerCycle` — if set, contractor cannot accept more jobs than this limit per billing cycle

---

## 8. Admin Refund Flow

Refunds are **admin-initiated only** via:
```graphql
mutation {
  initiateMarketplaceRefund(
    jobId: "<job-id>"
    reason: CONTRACTOR_REJECTED  # CONTRACTOR_REJECTED | CONSUMER_CANCELLED | ADMIN_CANCELLED | SLA_BREACH | OTHER
    notes: "Contractor no-show"
  ) {
    id
    status      # → PROCESSING (not SUCCESS yet — async)
    amount
    reason
  }
}
```

**Refund lifecycle** (fixed 10 Mar 2026):

```
Admin calls mutation
  → Drizzle inserts mp_refunds row (PENDING)
  → cashfreeService.createRefund() called
  → Row updated to PROCESSING (not SUCCESS — Cashfree refunds are async)
  → Cashfree sends REFUND_STATUS_WEBHOOK
  → handleRefundWebhook() fires
  → On SUCCESS: mp_refunds → SUCCESS, mp_payments → REFUNDED, mp_jobs → REFUNDED
  → On FAILED:  mp_refunds → FAILED
```

> **Why not mark SUCCESS immediately?** Cashfree refunds take minutes to hours to process. Marking immediately as SUCCESS would show the consumer a confirmed refund that hasn't hit their bank yet.

---

## 9. Webhook Controller — Security and Routing

**File:** `user-service/src/modules/payments/cashfree-webhook.controller.ts`  
**Endpoint:** `POST /payments/cashfree/webhook`

```
Incoming request
  ↓
Check headers: x-webhook-signature + x-webhook-timestamp
  ↓ missing → 400 Bad Request
HMAC-SHA256 verify (CashfreeService.verifyWebhookSignature)
  ↓ invalid → 400 Invalid signature (logged as security warning)
Parse payload.type
  ├── REFUND_STATUS_WEBHOOK → paymentsService.handleRefundWebhook(cf_refund_id, refund_id, status)
  └── PAYMENT_SUCCESS_WEBHOOK / PAYMENT_FAILED_WEBHOOK / etc
         Extract: order_id, order_status, cf_payment_id, payment_method
         ├── PAID          → handlePaymentWebhook(orderId, 'SUCCESS', paymentId, method)
         └── CANCELLED/FAILED/EXPIRED → handlePaymentWebhook(orderId, 'FAILED')
Return 200 OK
```

**Why raw body matters:**  
`main.ts` uses `bodyParser.json({ verify: (req, res, buf) => { req.rawBody = buf.toString() } })`. Cashfree's HMAC check runs over the raw bytes before JSON parsing. If we verified the parsed JSON string, any serialization difference would cause false failures.

---

## 10. CashfreeService — SDK Wrapper

**File:** `user-service/src/modules/payments/cashfree.service.ts`

| Method | SDK Call | Purpose |
|---|---|---|
| `createOrder(input)` | `PGCreateOrder()` | Create order → returns `payment_session_id` |
| `getOrder(orderId)` | `PGFetchOrder()` | Poll order status (sync fallback) |
| `verifyWebhookSignature(sig, body, ts)` | `PGVerifyWebhookSignature()` | HMAC-SHA256 check |
| `createRefund(orderId, amount, refundId)` | `PGOrderCreateRefund()` | Initiate refund |
| `getRefund(orderId, refundId)` | `PGOrderFetchRefund()` | Lookup refund status |

**`ensureInitialized()` guard** — every method calls this first. If `CASHFREE_APP_ID` or `CASHFREE_SECRET` is missing at startup, the first call throws `InternalServerErrorException` with a clear message instead of a confusing `TypeError: Cannot read properties of undefined`.

**API version pinned to `'2023-08-01'`** — prevents unexpected breaking changes if Cashfree releases a new API version while we're in production.

**Environment auto-detection:**
```typescript
const env = NODE_ENV === 'production' ? CFEnvironment.PRODUCTION : CFEnvironment.SANDBOX;
```

---

## 11. GraphQL API Reference

### Mutations

| Mutation | Auth | Role | Input | Returns |
|---|---|---|---|---|
| `initiateMarketplacePayment` | JWT | Any | `jobId: ID!` | `{ paymentId, sessionId, amount }` |
| `initiateSubscriptionUpgrade` | JWT | CONTRACTOR | `{ duration, bankAccount? }` | `{ success, paymentIntent { id, orderId, amount } }` |
| `confirmSubscriptionPayment` | JWT | CONTRACTOR | `{ paymentIntentId }` | `{ success, newSubscriptionType, validTill }` |
| `initiateMarketplaceRefund` | JWT | ADMIN/SUPER_ADMIN | `jobId, reason, notes?` | `MarketplaceRefund` |
| `simulatePaymentSuccess` | JWT | **SUPER_ADMIN only** | `jobId` | `MarketplaceJob` |

### Queries

| Query | Auth | Role | Returns |
|---|---|---|---|
| `marketplacePayments(filters?)` | JWT | ADMIN/SUPER_ADMIN | Paginated payment list |
| `marketplacePaymentByJob(jobId)` | JWT | Any | `MarketplacePayment` |
| `marketplaceRefunds(filters?)` | JWT | ADMIN/SUPER_ADMIN | Paginated refund list |
| `marketplaceRefundByJob(jobId)` | JWT | Any | `MarketplaceRefund` |
| `mySubscriptionStatus` | JWT | CONTRACTOR | Current subscription info |
| `subscriptionPaymentIntent(id)` | JWT | Any | Payment intent lookup |

### REST

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/payments/cashfree/webhook` | None (HMAC-signed) | Cashfree event receiver |

---

## 12. Bugs Fixed (10 March 2026)

| # | Bug | Root Cause | Fix |
|---|---|---|---|
| 1 | `initiateSubscriptionUpgrade` FK violation | `contractors.id` used as `mp_payments.user_id` FK (which points to `users.id`) | Resolve `contractors.userId` first, use that for payment insert and Cashfree user fetch |
| 2 | Migrations not applied → `job_id NOT NULL` violation | Migrations 0021 + 0022 were generated but never run | `pnpm db:migrate` applied; `job_id` is now nullable for SUBSCRIPTION rows |
| 3 | `CashfreeService` crashes on undefined | `this.cashfree` stays `undefined` if env vars missing | `ensureInitialized()` guard throws clean `InternalServerErrorException` |
| 4 | Refund marked `SUCCESS` immediately | Refunds are async but code set `SUCCESS` right after Cashfree API call | Status now set to `PROCESSING`; `handleRefundWebhook()` flips to `SUCCESS` on Cashfree callback |
| 5 | No refund webhook handler | `CashfreeWebhookController` only handled payment events | Added `REFUND_STATUS_WEBHOOK` branch; `handleRefundWebhook()` added to `PaymentsService` |
| 6 | `gatewayPaymentId`/`gatewayPaymentMethod` never stored | Webhook handler didn't extract payment details | Extracted from payload, passed to `handlePaymentWebhook`, persisted on `mp_payments` row |
| 7 | `simulatePaymentSuccess` production-accessible | No role guard, no env check | Locked to `SUPER_ADMIN` role + runtime `NODE_ENV === 'production'` block |
| 8 | `simulatePaymentSuccess` left `mp_payments` stale | Only updated `mp_jobs`, never touched `mp_payments` | Now updates `mp_payments.status → SUCCESS` and `completedAt` atomically |
| 9 | `qrProfileToken` null on auto-created profiles | Both auto-create paths (Excel import + subscription upgrade) omitted the field | `uuidv4()` token generated in all 3 creation paths |

---

## 13. QR Profile Token

`qrProfileToken` is a UUID stored in `mp_contractor_profile.qr_profile_token` (UNIQUE). It enables the public QR badge API (`getContractorPublicProfile`) — no auth required, scanned by consumers via QR code.

**Why it was null:** Only `ContractorProfileService.createProfile()` (explicit admin creation) generated the token. The two auto-create paths did not:
- `SubscriptionsService.initiateUpgrade()` — auto-creates FREE profile if missing
- `imports.resolver.ts` → batch Excel import

**Fix:** `uuidv4()` is now included in all three insert paths.

**Backfill for existing production nulls:**
```sql
UPDATE mp_contractor_profile
SET qr_profile_token = gen_random_uuid()
WHERE qr_profile_token IS NULL;
```
Run this once against your production/remote DB.

---

## 14. Module Wiring

`PaymentsModule` and `SubscriptionsModule` use `forwardRef()` to break the circular dependency:

- `PaymentsService` depends on `SubscriptionsService` (to call `finalizeSubscriptionUpgrade` on webhook)
- `SubscriptionsService` depends on `CashfreeService` (to create Cashfree orders for upgrades)

```typescript
// payments.module.ts
imports: [DatabaseModule, forwardRef(() => SubscriptionsModule)]
exports: [PaymentsService, CashfreeService]

// subscriptions.module.ts
imports: [DatabaseModule, forwardRef(() => PaymentsModule)]
```

`CashfreeWebhookController` is registered in `PaymentsModule` — it's a REST controller alongside the GraphQL resolvers.

---

## 15. Production Checklist

| Item | Status |
|---|---|
| Cashfree SDK integrated (not mock) | ✅ Done |
| HMAC webhook signature verification | ✅ Done |
| Async refund status via webhook | ✅ Done |
| `gatewayPaymentId` stored per payment | ✅ Done |
| `simulatePaymentSuccess` locked to SUPER_ADMIN | ✅ Done |
| FK fix: contractor `userId` used in payments | ✅ Done |
| `qrProfileToken` set on all auto-create paths | ✅ Done |
| Cashfree sandbox → production env switch | Auto (via `NODE_ENV=production`) |
| Real `CASHFREE_APP_ID` / `CASHFREE_SECRET` in prod env | ⚠️ Set before deploying |
| Cashfree webhook URL registered in Cashfree dashboard | ⚠️ Set to `https://<domain>/payments/cashfree/webhook` |
| Refund SMS notifications to consumer | ❌ Not wired (TODO in jobs.service.ts) |
| Job payment retry flow (re-book after failed payment) | ❌ Not implemented (FK unique constraint blocks re-insert) |

---

## 16. React Native / Mobile Integration

### Available Mutations

The backend **does not have** generic `createPayment` or `updatePayment` mutations. It has **two specific flows**:

#### 1. Job Payment (Consumer)

```graphql
mutation InitiateMarketplacePayment($jobId: ID!) {
  initiateMarketplacePayment(jobId: $jobId) {
    paymentId      # Internal mp_payments.id
    sessionId      # Cashfree payment_session_id (pass to SDK)
    paymentLink    # nullable (legacy, ignore for SDK v5)
    amount
  }
}
```

#### 2. Subscription Payment (Contractor)

```graphql
mutation InitiateSubscriptionUpgrade($input: InitiateSubscriptionUpgradeInput!) {
  initiateSubscriptionUpgrade(input: $input) {
    success
    message
    paymentIntent {
      id           # Payment intent ID (for confirmSubscriptionPayment)
      orderId      # Cashfree order ID (use this in CFSession)
      amount
      expiresAt    # 30-minute TTL
      status
    }
  }
}
```

**Input:**
```typescript
{
  duration: "MONTHLY" | "QUARTERLY" | "YEARLY"
  bankAccount?: {  // Optional, only if not set yet
    accountNumber: string
    ifscCode: string
    accountHolderName: string
    bankName: string
  }
}
```

### ⚠️ No Manual Update Needed

Payment status is **automatically updated via webhook**. When Cashfree calls `POST /payments/cashfree/webhook`, the backend:
1. Verifies HMAC signature
2. Updates `mp_payments.status` to `SUCCESS`/`FAILED`
3. For subscriptions: calls `finalizeSubscriptionUpgrade()` to activate PREMIUM
4. For jobs: updates job status to `REQUESTED`

**The mobile app should NOT call any update mutation after payment.** Just handle the `onVerify` callback.

### React Native Hook Implementation

```typescript
import { useMutation } from "@apollo/client";
import { CFEnvironment, CFSession } from "cashfree-pg-api-contract";
import {
  CFErrorResponse,
  CFPaymentGatewayService,
} from "react-native-cashfree-pg-sdk";
import { gql } from "@apollo/client";

// GraphQL Mutations
const INITIATE_JOB_PAYMENT = gql`
  mutation InitiateMarketplacePayment($jobId: ID!) {
    initiateMarketplacePayment(jobId: $jobId) {
      paymentId
      sessionId
      amount
    }
  }
`;

const INITIATE_SUBSCRIPTION_UPGRADE = gql`
  mutation InitiateSubscriptionUpgrade($input: InitiateSubscriptionUpgradeInput!) {
    initiateSubscriptionUpgrade(input: $input) {
      success
      message
      paymentIntent {
        id
        orderId
        amount
        expiresAt
      }
    }
  }
`;

type JobPaymentInput = {
  type: "JOB";
  jobId: string;
};

type SubscriptionPaymentInput = {
  type: "SUBSCRIPTION";
  duration: "MONTHLY" | "QUARTERLY" | "YEARLY";
  bankAccount?: {
    accountNumber: string;
    ifscCode: string;
    accountHolderName: string;
    bankName: string;
  };
};

type PaymentInput = JobPaymentInput | SubscriptionPaymentInput;

const usePayment = () => {
  const [initiateJobPayment] = useMutation(INITIATE_JOB_PAYMENT);
  const [initiateSubscriptionUpgrade] = useMutation(INITIATE_SUBSCRIPTION_UPGRADE);

  const initiatePayment = async (input: PaymentInput) => {
    let sessionId: string;
    let orderId: string;

    if (input.type === "JOB") {
      // Job Payment Flow
      const { data } = await initiateJobPayment({
        variables: { jobId: input.jobId },
      });

      sessionId = data?.initiateMarketplacePayment.sessionId;
      orderId = data?.initiateMarketplacePayment.paymentId; // Use paymentId as orderId
    } else {
      // Subscription Payment Flow
      const { data } = await initiateSubscriptionUpgrade({
        variables: {
          input: {
            duration: input.duration,
            bankAccount: input.bankAccount,
          },
        },
      });

      if (!data?.initiateSubscriptionUpgrade.success) {
        throw new Error(
          data?.initiateSubscriptionUpgrade.message || "Subscription initiation failed"
        );
      }

      const intent = data.initiateSubscriptionUpgrade.paymentIntent;
      sessionId = intent.orderId; // orderId is the Cashfree order ID
      orderId = intent.orderId;
    }

    if (!sessionId || !orderId) {
      throw new Error("Invalid payment session or order ID");
    }

    // Initialize Cashfree Session
    const session = new CFSession(
      sessionId,
      orderId,
      __DEV__ ? CFEnvironment.SANDBOX : CFEnvironment.PRODUCTION
    );

    return new Promise<{ success: boolean; orderID: string }>(
      (resolve, reject) => {
        CFPaymentGatewayService.setCallback({
          onVerify: async (verifiedOrderID: string) => {
            // Webhook handles backend update — no manual mutation needed
            console.log("✅ Payment verified:", verifiedOrderID);
            resolve({ success: true, orderID: verifiedOrderID });
          },
          onError: (error: CFErrorResponse, failedOrderID: string) => {
            console.error(
              "❌ Payment Error:",
              error.getMessage(),
              "Order:",
              failedOrderID
            );
            reject(error);
          },
        });

        // Start Cashfree payment flow
        CFPaymentGatewayService.doWebPayment(session);
      }
    );
  };

  return { initiatePayment };
};

export default usePayment;
```

### Usage Examples

**Consumer Job Payment:**
```typescript
const { initiatePayment } = usePayment();

await initiatePayment({
  type: "JOB",
  jobId: "uuid-of-job",
});
```

**Contractor Subscription Upgrade:**
```typescript
await initiatePayment({
  type: "SUBSCRIPTION",
  duration: "MONTHLY",
  bankAccount: {
    accountNumber: "1234567890",
    ifscCode: "SBIN0001234",
    accountHolderName: "John Doe",
    bankName: "SBI",
  },
});
```

### Key Integration Points

| Aspect | Implementation |
|---|---|
| **Environment Detection** | Use `__DEV__` or `NODE_ENV` to switch between `SANDBOX` and `PRODUCTION` |
| **Session Creation** | Pass `sessionId` (from backend) + `orderId` to `CFSession` constructor |
| **Callback Handling** | Use `onVerify` to handle success — webhook updates DB automatically |
| **Error Handling** | `onError` receives `CFErrorResponse` with failure details |
| **No Manual Update** | Do NOT call any backend mutation after payment — webhook handles it |

---

## 1. System-Wide Life Cycle (The "Happy Path")

A payment doesn't just "happen"; it moves through three major states in our system:

1.  **INTENT (Synchronous)**:
    - User clicks "Pay" or "Upgrade".
    - `initiateMarketplacePayment` or `initiateSubscriptionUpgrade` mutation is called.
    - We record the intent in `mp_payments` as `PENDING`.
    - We "handshake" with Cashfree and get a `payment_session_id`.
2.  **INTERACTION (Client-Side)**:
    - The Mobile App or Web Frontend uses the `sessionId` to render the Cashfree overlay.
    - Our backend is **idle** during this time.
3.  **RESOLUTION (Asynchronous)**:
    - Cashfree calls our Webhook.
    - We double-check the signature (Security).
    - We flip the `PENDING` payment to `SUCCESS`.
    - We trigger side-effects (Activate Job or Upgrade Subscription).

---

## 2. Micro-Analysis: File-by-File Deep Dive

### A. The Orchestrator: `PaymentsService`

_File: `src/modules/marketplace/payments/payments.service.ts`_

#### `initiatePayment(jobId, userId)`

- **Line 53–57**: We query the `jobs` table using Drizzle. This is critical because we need the **Price Snapshot** (`job[0].priceSnapshot`). We never trust the client to send the price; we only use what's in our DB.
- **Line 63–65**: Early return (Guard Clause). We prevent re-paying for a job that isn't `PAYMENT_PENDING`.
- **Line 68–80**: DB Insert. We set `gatewayProvider` to `'CASHFREE'`. We generate a `gatewayOrderId`. Note: If this jobId ever appears in Cashfree again with the same OrderID, Cashfree will block it, preventing "Replay Attacks".
- **Line 90–98**: We call `cashfreeService.createOrder`. This is an external API call.
- **Line 101–106**: Once we have the `payment_session_id`, we update our DB. This ID is ephemeral (valid for 30 minutes).

#### `handlePaymentWebhook(orderId, status)`

- **Line 129–133**: We lookup the payment record by its unique `gatewayOrderId`.
- **Line 148–157 (Side-Effect: Jobs)**: If successful, we flip the Job status to `REQUESTED` and record the `paidAt` timestamp.
- **Line 158–166 (Side-Effect: Subscriptions)**: This is a **Cross-Module Call**. We tell `SubscriptionsService` to finalize the upgrade.

---

### B. The Specialty Domain: `SubscriptionsService`

_File: `src/modules/marketplace/subscriptions/subscriptions.service.ts`_

#### `initiateUpgrade(input)`

- **Line 467–476**: We fetch the **Current Active Plan** from `subscription_plans`. If the admin just changed the price, the user gets the _new_ price instantly.
- **Line 495–504**: **Business Logic Guard**. We calculate `daysRemaining`. If they have > 30 days of Premium, we block the purchase to prevent waste. This saves customer support tickets.

#### `finalizeSubscriptionUpgrade(paymentId)`

- **Line 666–713: The Transaction Block**
  - We use `await this.db.transaction(async (tx) => { ... })`. This is the most important 50 lines of code in the module.
  - **Atomic Step 1**: Update `mp_payments` to `SUCCESS`.
  - **Atomic Step 2**: Update `contractor_marketplace_profile`. We set `subscriptionType = 'PREMIUM'`.
  - **Atomic Step 3**: Date Calculation (`endDate`). If they already had time, we add to it. If not, we start from today.
  - **Atomic Step 4**: Deactivate old rows. We set `isActive = false` on all previous entries in `contractor_subscriptions`.
  - **Atomic Step 5**: History Log. We insert a fresh record so the user can see their invoice history.

---

### C. The Secure Gateway: `CashfreeService`

_File: `src/modules/marketplace/payments/cashfree.service.ts`_

- **Line 21**: We fix the API version to `'2023-08-01'`. This ensures that if Cashfree updates their API, our integration doesn't break unexpectedly.
- **Line 104: The "Crypto" Check**
  - `this.cashfree.PGVerifyWebhookSignature(signature, rawBody, timestamp);`
  - This isn't just a simple string compare. It uses HMAC-SHA256 to verify the body. If a single character in the body is changed by a hacker, this will throw an error.

---

## 3. Implementation APIs Reference

| Endpoint Type    | Identifier                        | Inputs     | Logical Path                                         |
| :--------------- | :-------------------------------- | :--------- | :--------------------------------------------------- |
| **GQL Mutation** | `initiateMarketplacePayment`      | `jobId`    | `PaymentsService` -> `Cashfree SDK`                  |
| **GQL Mutation** | `initiateSubscriptionUpgrade`     | `duration` | `SubscriptionsService` -> `PaymentsService` -> `SDK` |
| **GQL Query**    | `mySubscriptionStatus`            | N/A        | `SubscriptionsService.getSubscriptionStatus`         |
| **REST API**     | `POST /payments/cashfree/webhook` | Raw JSON   | `WebhookController` -> `Security Check` -> `Service` |

---

## 4. Pending Works & Production Roadmap

Our integration is "Feature Complete" for Sandbox, but the following are required for **Production Go-Live**:

1.  **Idempotency (DB Level)**: Add a unique constraint on `mp_payments.gateway_order_id`. Currently handled by logic, but a DB constraint is the "Final Line of Defense".
2.  **Refund Webhook Listener**: We can initiate refunds, but we need to listen for the `REFUND_SUCCESS` webhook to update the `mp_refunds` table asynchronously.
3.  **Plan Versioning Lock**: When a user initiates an upgrade, we should "lock" the `planVersion`. If the admin changes the price 1 minute later, the user should still pay what was shown on their screen.
4.  **Error Notifications**: Integrate with Sentry or a Slack Webhook in `PaymentsService.handlePaymentWebhook` to alert the team if a signature verification fails (indicates a potential attack).

---

## 5. Summary of Implementation

| Feature               | Logic Location              | Implementation Status |
| :-------------------- | :-------------------------- | :-------------------- |
| Job Payments          | `PaymentsService`           | ✅ COMPLETE           |
| Subscription Upgrades | `SubscriptionsService`      | ✅ COMPLETE           |
| Cashfree SDK Wrapper  | `CashfreeService`           | ✅ COMPLETE           |
| Webhook Security      | `CashfreeWebhookController` | ✅ COMPLETE           |
| Admin Refunds         | `PaymentsResolver`          | ✅ COMPLETE           |
| Automated Expiry      | `SubscriptionsScheduler`    | ✅ COMPLETE           |
