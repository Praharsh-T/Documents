# Cashfree Payment Integration: The Definitive Technical Manual

This guide provides an exhaustive, line-by-line breakdown of the Marketplace and Subscriptions payment system. It is designed for developers who need to understand exactly how the money flows—and how the code ensures security and data integrity at every step.

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
