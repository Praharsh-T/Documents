# Contractor Payout System — Complete Technical Reference

> **Last Updated:** 27 March 2026  
> **Applies to:** `user-service` — `PayoutsModule` (`src/modules/marketplace/payouts/`)  
> **Access:** All payout operations are **ADMIN / SUPER_ADMIN only**

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Environment Setup](#2-environment-setup)
3. [Database Schema](#3-database-schema)
4. [Deduction Formula](#4-deduction-formula)
5. [Full Lifecycle — Step by Step](#5-full-lifecycle--step-by-step)
   - [Step 0 — Contractor Bank Setup](#step-0--contractor-bank-setup-prerequisite)
   - [Step 1 — Jobs Accumulate](#step-1--jobs-accumulate-payout_status--unpaid)
   - [Step 2 — Preview Batch](#step-2--preview-batch-read-only)
   - [Step 3 — Generate Batch (DRAFT)](#step-3--generate-batch-draft)
   - [Step 4 — Submit for Approval](#step-4--submit-for-approval-draft--pending_approval)
   - [Step 5 — Approve](#step-5--approve-pending_approval--approved)
   - [Step 6 — Process (Trigger Transfers)](#step-6--process-approved--processing--completefailed)
   - [Step 7 — Webhook Resolution](#step-7--webhook-resolution-async)
   - [Cancel](#cancel-available-until-processing)
6. [State Machines](#6-state-machines)
7. [Cashfree Payouts v2 API](#7-cashfree-payouts-v2-api)
8. [GraphQL API Reference](#8-graphql-api-reference)
9. [Subcontractor Routing](#9-subcontractor-routing)
10. [SKIPPED Items — What Causes Them](#10-skipped-items--what-causes-them)
11. [Audit Trail](#11-audit-trail)
12. [Production Checklist](#12-production-checklist)

---

## 1. Overview & Architecture

Payouts are **not automatic per job**. They are **batched** — an admin collects multiple completed jobs across multiple contractors into a single batch, reviews it, approves it, then triggers all bank transfers at once via the **Cashfree Payouts v2 API** (IMPS mode).

```
COMPLETED jobs (payoutStatus=UNPAID)
        ↓
   [Admin: Generate Batch]         → DRAFT batch created
        ↓
   [Admin: Submit]                 → PENDING_APPROVAL
        ↓
   [Admin: Approve]                → APPROVED + jobs LOCKED
        ↓
   [Admin: Process]                → PROCESSING
        ↓ (Cashfree Payouts v2)
   Per-contractor IMPS transfer    → SUCCESS / FAILED / PENDING
        ↓ (Cashfree webhook)
   jobs.payoutStatus = PAID        → COMPLETED / FAILED
```

### Files

```
src/modules/marketplace/payouts/
├── payouts.service.ts     — All business logic (generate, submit, approve, process, cancel)
├── payouts.resolver.ts    — GraphQL mutations + queries (ADMIN/SUPER_ADMIN only)
├── payouts.types.ts       — GraphQL ObjectTypes, InputTypes, enums
└── payouts.module.ts      — NestJS wiring
```

---

## 2. Environment Setup

```env
# Cashfree Payouts v2 — separate product from PG, separate credentials
CASHFREE_PAYOUT_CLIENT_ID=CF_PAYOUT_CLIENT_ID_HERE
CASHFREE_PAYOUT_CLIENT_SECRET=CF_PAYOUT_CLIENT_SECRET_HERE

# NODE_ENV controls which Cashfree base URL is used:
#   development/staging → https://payout-gamma.cashfree.com
#   production          → https://payout-api.cashfree.com
NODE_ENV=development
```

> **Important:** Cashfree Payouts uses **completely different credentials** from the Payment Gateway (`CASHFREE_APP_ID`/`CASHFREE_SECRET`). They are separate products in the Cashfree dashboard.

---

## 3. Database Schema

### `mp_payout_batches` — One row per batch

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid PK | |
| `reference_id` | varchar | e.g. `PAYOUT-20260327-001` |
| `schedule_type` | enum | `IMMEDIATE` or `WEEKLY` |
| `status` | enum | `DRAFT → PENDING_APPROVAL → APPROVED → PROCESSING → COMPLETED / FAILED / CANCELLED` |
| `cutoff_at` | timestamp | Only jobs completed before this time are eligible |
| `total_jobs` | int | Total job count across all contractors |
| `total_contractors` | int | Number of contractor items |
| `total_gross_amount` | decimal | Sum of all job prices |
| `total_commission_amount` | decimal | Sum of platform commissions |
| `total_gst_on_commission` | decimal | GST on platform commission (18%) |
| `total_tds_amount` | decimal | TDS withheld (1% of contractor earnings) |
| `total_net_amount` | decimal | What actually goes to contractors |
| `approved_by` | uuid FK → users | Set on approval |
| `approved_at` | timestamp | |
| `processing_started_at` | timestamp | Set when process is triggered |
| `completed_at` | timestamp | Set when all items settle |
| `cancelled_by` | uuid FK → users | |
| `cancelled_at` | timestamp | |
| `notes` | text | Admin notes |
| `created_by` | uuid FK → users | Null for system-generated batches |

### `mp_payout_batch_items` — One row per contractor per batch

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid PK | |
| `batch_id` | uuid FK → batches | |
| `contractor_id` | uuid FK | The contractor receiving payment |
| `bank_account_number` | varchar | Snapshotted at batch creation |
| `bank_ifsc_code` | varchar | Snapshotted at batch creation |
| `bank_account_holder_name` | varchar | Snapshotted at batch creation |
| `job_count` | int | Number of jobs in this item |
| `gross_amount` | decimal | |
| `commission_amount` | decimal | |
| `gst_on_commission` | decimal | |
| `tds_amount` | decimal | |
| `net_amount` | decimal | Amount transferred to contractor |
| `status` | enum | `PENDING / PROCESSING / SUCCESS / FAILED / SKIPPED` |
| `gateway_transfer_id` | varchar | Cashfree `cf_transfer_id` |
| `gateway_reference_id` | varchar | Our `PO-{itemId}` idempotency key |
| `gateway_utr` | varchar | UTR number — bank transfer reference |
| `processed_at` | timestamp | When transfer succeeded |
| `failure_reason` | text | Error message if FAILED |

### `mp_payout_job_ledger` — One row per job per batch item

| Column | Type | Notes |
|--------|------|-------|
| `batch_item_id` | uuid FK | |
| `job_id` | uuid FK | |
| `gross_amount` | decimal | |
| `commission_amount` | decimal | |
| `gst_on_commission` | decimal | |
| `contractor_earnings` | decimal | gross − commission − GST |
| `tds_amount` | decimal | |
| `net_amount` | decimal | |

### `mp_payout_audit_logs` — Full audit trail

| Column | Type | Notes |
|--------|------|-------|
| `batch_id` | uuid FK | |
| `batch_item_id` | uuid FK | Nullable — null for batch-level events |
| `action` | varchar | e.g. `BATCH_CREATED`, `ITEM_SUCCESS` |
| `actor_id` | uuid FK | Null for system/Cashfree events |
| `actor_role` | varchar | `ADMIN`, `SYSTEM` |
| `details` | jsonb | Free-form event data |

---

## 4. Deduction Formula

Applied **per job**, then summed into the batch item totals.

```
gross              = job.totalPriceSnapshot      (what consumer paid — never trusted from client)
commission         = job.commissionAmount         (snapshotted at job creation)
gstOnCommission    = commission × 18%
contractorEarnings = gross − commission − gstOnCommission
tds                = contractorEarnings × 1%     (TDS deducted at source)
net                = contractorEarnings − tds     ← amount transferred to bank
```

**Example — Job worth ₹1,000, platform commission ₹100:**
```
gross              = ₹1,000.00
commission         = ₹100.00
gstOnCommission    = ₹100 × 18%   = ₹18.00
contractorEarnings = ₹1,000 − ₹100 − ₹18 = ₹882.00
tds                = ₹882 × 1%   = ₹8.82
net (transferred)  = ₹882 − ₹8.82 = ₹873.18
```

All amounts are rounded to 2 decimal places at each step.

---

## 5. Full Lifecycle — Step by Step

---

### Step 0 — Contractor Bank Setup (Prerequisite)

Before any payout can be sent to a contractor, they must have:
1. Bank account details saved in `mp_contractor_profile` (`bankAccountNumber`, `bankIfscCode`, `bankAccountHolderName`)
2. `beneficiaryRegistered = true` — registered as a Cashfree Payouts beneficiary

If a contractor has no bank details at batch **generation** time, their item is marked `SKIPPED` immediately and they receive no payout for that batch cycle.

**Contractor saves bank details (mobile app mutation):**
```graphql
mutation UpdateBankDetails {
  updateContractorBankDetails(input: {
    bankAccount: {
      accountNumber: "123456789012"
      ifscCode: "SBIN0001234"
      accountHolderName: "Rajesh Kumar"
      bankName: "State Bank of India"
    }
  }) {
    success
    message
    beneficiaryRegistered   # true = ready for payouts
    beneficiaryWarning      # non-null if Cashfree registration failed
  }
}
```

**Check registration status (contractor profile query):**
```graphql
query {
  marketplaceContractorProfile(contractorId: "<id>") {
    bankAccountNumber
    bankIfscCode
    bankAccountHolderName
    beneficiaryRegistered
  }
}
```

---

### Step 1 — Jobs Accumulate (`payoutStatus = UNPAID`)

Every job that reaches `COMPLETED` status automatically gets `payoutStatus = UNPAID`. Jobs accumulate silently until an admin generates a batch.

**Eligibility rules for a job to be included in a batch:**
- `status = COMPLETED`
- `payoutStatus = UNPAID` (not already `LOCKED` or `PAID`)
- `completedAt ≤ cutoffAt`

**For immediate batches:** `cutoffAt = current hour floor − 2 hours`
- e.g. admin triggers at 3:47 PM → cutoff = 1:00 PM
- Only jobs completed before 1:00 PM qualify
- This 2-hour buffer prevents processing a job the contractor just finished

---

### Step 2 — Preview Batch (Read-Only)

Admin sees a preview of what the next batch would contain **without writing anything**.

```graphql
query PayoutPreview {
  payoutPreview(cutoffAt: "2026-03-27T11:00:00Z") {
    cutoffAt
    eligibleJobCount
    totalNetAmount
    byContractor {
      contractorId
      contractorName
      jobCount
      grossAmount
      commissionAmount
      gstOnCommission
      tdsAmount
      netAmount
      hasBankDetails     # false = contractor will be SKIPPED if batch generated now
    }
  }
}
```

**Response example:**
```json
{
  "payoutPreview": {
    "cutoffAt": "2026-03-27T11:00:00Z",
    "eligibleJobCount": 47,
    "totalNetAmount": 38420.50,
    "byContractor": [
      {
        "contractorId": "uuid-1",
        "contractorName": "Kumar Electricals",
        "jobCount": 12,
        "grossAmount": 12000.00,
        "commissionAmount": 1200.00,
        "gstOnCommission": 216.00,
        "tdsAmount": 105.84,
        "netAmount": 10478.16,
        "hasBankDetails": true
      },
      {
        "contractorId": "uuid-2",
        "contractorName": "Sharma Services",
        "jobCount": 3,
        "grossAmount": 2500.00,
        "commissionAmount": 250.00,
        "gstOnCommission": 45.00,
        "tdsAmount": 22.05,
        "netAmount": 2182.95,
        "hasBankDetails": false    ← will be SKIPPED
      }
    ]
  }
}
```

---

### Step 3 — Generate Batch (DRAFT)

Admin triggers batch creation. This **writes to the database**.

```graphql
mutation GeneratePayout {
  generateImmediatePayout(input: {
    cutoffAt: "2026-03-27T11:00:00Z"   # optional — defaults to now-floor-2h
    notes: "Weekly batch for March W4"  # optional
  }) {
    id
    referenceId          # e.g. "PAYOUT-20260327-001"
    status               # "DRAFT"
    scheduleType         # "IMMEDIATE"
    cutoffAt
    totalJobs
    totalContractors
    totalGrossAmount
    totalCommissionAmount
    totalGstOnCommission
    totalTdsAmount
    totalNetAmount
    createdAt
    notes
  }
}
```

**What the backend does internally:**
1. Fetches all eligible UNPAID COMPLETED jobs up to `cutoffAt`
2. Groups by `COALESCE(payoutContractorId, contractorId)` — subcontractor jobs go to parent
3. Reads each contractor's bank details from `mp_contractor_profile` (snapshot at this moment)
4. Calculates per-job deductions → sums into per-contractor totals
5. Inside a single DB transaction:
   - Inserts `mp_payout_batches` row (DRAFT)
   - For each contractor: inserts `mp_payout_batch_items` row (PENDING if has bank details, SKIPPED if not)
   - For each job: inserts `mp_payout_job_ledger` row
   - Inserts `BATCH_CREATED` audit log

**Response includes `referenceId`** — save this ID for the next steps.

---

### Step 4 — Submit for Approval (DRAFT → PENDING_APPROVAL)

Admin who generated the batch (or another admin) submits it for a second-level review.

```graphql
mutation SubmitBatch {
  submitPayoutBatch(batchId: "<batch-uuid>") {
    id
    referenceId
    status         # "PENDING_APPROVAL"
    updatedAt
  }
}
```

No financial operations happen here — this is a pure status transition for workflow control.

---

### Step 5 — Approve (PENDING_APPROVAL → APPROVED)

A second admin (or same admin) approves the batch. **This locks all jobs in the batch.**

```graphql
mutation ApproveBatch {
  approvePayoutBatch(input: {
    batchId: "<batch-uuid>"
    notes: "Reviewed — all amounts correct"   # optional
  }) {
    id
    referenceId
    status          # "APPROVED"
    approvedAt
    approvedBy
  }
}
```

**What happens on approval:**
- All jobs in the batch have `payoutStatus` flipped from `UNPAID` → `LOCKED`
- `LOCKED` jobs cannot appear in any other batch
- Audit log: `BATCH_APPROVED` with count of locked jobs
- **Cancel is still possible** at this stage — jobs will be unlocked back to `UNPAID`

---

### Step 6 — Process (APPROVED → PROCESSING → COMPLETE/FAILED)

Admin triggers the actual bank transfers. **Money starts moving.**

```graphql
mutation ProcessBatch {
  processPayoutBatch(batchId: "<batch-uuid>") {
    id
    referenceId
    status              # "PROCESSING" or "COMPLETED" / "FAILED" (if all resolve synchronously)
    processingStartedAt
    completedAt
    items {
      contractorId
      bankAccountHolderName
      netAmount
      status            # PENDING / PROCESSING / SUCCESS / FAILED / SKIPPED
      gatewayTransferId
      gatewayUtr
      failureReason
    }
  }
}
```

**Internal processing per contractor item:**

```
For each PENDING item:
  1. Ensure Cashfree beneficiary registered
       POST https://payout-gamma.cashfree.com/payout/beneficiary
       { beneficiary_id: "BENE-{contractorId20chars}",
         beneficiary_name: "Rajesh Kumar",
         beneficiary_instrument_details: { bank_account_number, bank_ifsc } }
       → 409 "already exists" = silently ignored, proceed

  2. Initiate IMPS transfer
       POST /payout/transfers
       { transfer_id: "PO-{batchItemId20chars}",   ← idempotency key
         transfer_amount: 10478.16,
         transfer_mode: "IMPS",
         beneficiary_id: "BENE-...",
         transfer_remarks: "Contractor payout — batch abc12345" }

  3. Handle response:
       transfer_status = "SUCCESS"  → item SUCCESS, jobs PAID, record UTR
       transfer_status = "PENDING"  → item stays PROCESSING, wait for webhook
       anything else               → item FAILED, error logged, CONTINUE to next item
```

> A failure on one contractor does **not abort** the others — all items are processed regardless.

**After all items processed:**
- If all items are `SUCCESS` or `SKIPPED` → batch `COMPLETED`
- If any item is `FAILED` → batch `FAILED`
- `PROCESSING` items (async) leave the batch in `PROCESSING` until webhook resolves them

---

### Step 7 — Webhook Resolution (Async)

For items awaiting bank confirmation (`transfer_status = PENDING`), Cashfree calls the backend webhook:

```
POST /payments/cashfree/payout-webhook   (or configured in Cashfree Payouts dashboard)
```

```typescript
// Internal webhook handler (payouts.service.ts)
async handlePayoutTransferWebhook(payload: CashfreePayoutWebhookPayload) {
  // Matches transfer by transfer_id
  // On SUCCESS: item → SUCCESS, jobs → PAID, record UTR
  // On FAILED:  item → FAILED, record reason
  // Then: reconcileBatchStatus() — checks if all items settled → batch COMPLETED/FAILED
}
```

**After all items settle, query final state:**

```graphql
query BatchStatus {
  payoutBatch(batchId: "<batch-uuid>") {
    id
    referenceId
    status        # COMPLETED or FAILED
    completedAt
    totalNetAmount
  }
}
```

---

### Cancel (Available Until PROCESSING)

```graphql
mutation CancelBatch {
  cancelPayoutBatch(input: {
    batchId: "<batch-uuid>"
    reason: "Amount discrepancy found — regenerating"
  }) {
    id
    referenceId
    status        # "CANCELLED"
    cancelledAt
  }
}
```

**What happens:**
- If batch was `APPROVED`: jobs are **unlocked** back to `UNPAID` — they will appear in the next batch
- If batch was `DRAFT` or `PENDING_APPROVAL`: jobs were never locked, nothing to undo
- Cancel is **NOT available** once `PROCESSING` starts — transfers are already in-flight

---

## 6. State Machines

### Batch Status

```
                    ┌─────────────────┐
                    │      DRAFT      │
                    └────────┬────────┘
                             │ submitPayoutBatch()
                    ┌────────▼────────┐
                    │ PENDING_APPROVAL │
                    └────────┬────────┘
                             │ approvePayoutBatch()
                    ┌────────▼────────┐      ┌───────────┐
                    │    APPROVED     │──────▶│ CANCELLED │
                    └────────┬────────┘       └───────────┘
                             │ processPayoutBatch()
                    ┌────────▼────────┐
                    │   PROCESSING    │
                    └────────┬────────┘
              ┌──────────────┴───────────────┐
     ┌────────▼────────┐           ┌─────────▼────────┐
     │    COMPLETED    │           │      FAILED      │
     └─────────────────┘           └──────────────────┘

DRAFT, PENDING_APPROVAL, APPROVED → CANCELLED  (cancel is available)
```

### Per-Item Status (within a batch)

```
PENDING        → set at batch generation (has valid bank details)
SKIPPED        → set at batch generation (no valid bank details → no money sent)
PROCESSING     → transfer initiated, waiting for bank / webhook
SUCCESS        → money sent, UTR recorded, jobs marked PAID
FAILED         → Cashfree transfer failed
```

### Job-Level Payout Status

```
UNPAID    → default for all COMPLETED jobs
LOCKED    → approval locks the job into a specific batch
PAID      → transfer succeeded; contractor received the money
```

---

## 7. Cashfree Payouts v2 API

> This is **Cashfree Payouts** — a completely different product from Cashfree PG (payment gateway). Different credentials, different base URL, different auth headers.

### Auth

No Bearer token step (v1 is deprecated). Every request uses direct credentials:

```http
x-client-id: CASHFREE_PAYOUT_CLIENT_ID
x-client-secret: CASHFREE_PAYOUT_CLIENT_SECRET
x-api-version: 2024-01-01
Content-Type: application/json
```

### Base URLs

```
Sandbox:    https://payout-gamma.cashfree.com
Production: https://payout-api.cashfree.com
```

### Beneficiary Registration

```http
POST /payout/beneficiary
```

```json
{
  "beneficiary_id": "BENE-{first20charsOfContractorIdNoDashes}",
  "beneficiary_name": "Rajesh Kumar",
  "beneficiary_instrument_details": {
    "bank_account_number": "123456789012",
    "bank_ifsc": "SBIN0001234"
  }
}
```

**Response codes:**
- `200` — registered successfully
- `409` — already registered (silently ignored, proceed to transfer)
- `4xx` — validation error (bad IFSC, duplicate account, etc.)

### Transfer

```http
POST /payout/transfers
```

```json
{
  "transfer_id": "PO-{first20charsOfBatchItemIdNoDashes}",
  "transfer_amount": 10478.16,
  "transfer_mode": "IMPS",
  "beneficiary_id": "BENE-...",
  "transfer_remarks": "Contractor payout — batch abc12345"
}
```

**Response:**
```json
{
  "transfer_id": "PO-...",
  "cf_transfer_id": "CF_1234567890",
  "transfer_status": "SUCCESS",   // or "PENDING"
  "transfer_utr": "UTR123456789"  // bank reference number
}
```

---

## 8. GraphQL API Reference

> All operations require `Authorization: Bearer <jwt>` with `ADMIN` or `SUPER_ADMIN` role.

### Queries

#### `payoutPreview` — Preview before generating

```graphql
query PayoutPreview($cutoffAt: DateTime) {
  payoutPreview(cutoffAt: $cutoffAt) {
    cutoffAt
    eligibleJobCount
    totalNetAmount
    byContractor {
      contractorId
      contractorName
      jobCount
      grossAmount
      commissionAmount
      gstOnCommission
      tdsAmount
      netAmount
      hasBankDetails
    }
  }
}
```

#### `payoutBatches` — List all batches with filters

```graphql
query PayoutBatches($filters: PayoutBatchFilterInput) {
  payoutBatches(filters: $filters) {
    items {
      id
      referenceId
      status
      scheduleType
      cutoffAt
      totalJobs
      totalContractors
      totalNetAmount
      createdAt
      completedAt
    }
    pagination {
      total
      page
      limit
      hasNext
    }
  }
}
```

Variables:
```json
{
  "filters": {
    "status": "COMPLETED",
    "page": 1,
    "limit": 20
  }
}
```

#### `payoutBatch` — Single batch detail

```graphql
query PayoutBatch($batchId: ID!) {
  payoutBatch(batchId: $batchId) {
    id
    referenceId
    status
    totalJobs
    totalContractors
    totalGrossAmount
    totalCommissionAmount
    totalGstOnCommission
    totalTdsAmount
    totalNetAmount
    cutoffAt
    approvedBy
    approvedAt
    processingStartedAt
    completedAt
    notes
    createdAt
  }
}
```

#### `payoutBatchItems` — Per-contractor breakdown

```graphql
query PayoutBatchItems($batchId: ID!) {
  payoutBatchItems(batchId: $batchId) {
    id
    contractorId
    bankAccountHolderName
    bankAccountNumber
    bankIfscCode
    jobCount
    grossAmount
    commissionAmount
    gstOnCommission
    tdsAmount
    netAmount
    status
    gatewayTransferId
    gatewayUtr
    processedAt
    failureReason
  }
}
```

#### `payoutBatchItemLedger` — Per-job breakdown for one contractor

```graphql
query ItemLedger($batchItemId: ID!) {
  payoutBatchItemLedger(batchItemId: $batchItemId) {
    jobId
    grossAmount
    commissionAmount
    gstOnCommission
    contractorEarnings
    tdsAmount
    netAmount
  }
}
```

#### `payoutAuditLogs` — Full audit trail for a batch

```graphql
query AuditLogs($batchId: ID!) {
  payoutAuditLogs(batchId: $batchId) {
    action
    actorId
    actorRole
    details
    createdAt
  }
}
```

#### `payoutScheduleConfig` — Scheduled payout settings

```graphql
query {
  payoutScheduleConfig {
    scheduleType      # IMMEDIATE / WEEKLY
    isEnabled
    cronExpression
    defaultCutoffHours
    updatedAt
  }
}
```

---

### Mutations

#### `generateImmediatePayout` — Create DRAFT batch

```graphql
mutation GeneratePayout($input: GenerateImmediatePayoutInput) {
  generateImmediatePayout(input: $input) {
    id
    referenceId
    status
    totalJobs
    totalContractors
    totalNetAmount
    cutoffAt
    createdAt
  }
}
```

Variables:
```json
{
  "input": {
    "cutoffAt": "2026-03-27T11:00:00Z",
    "notes": "March week 4 batch"
  }
}
```

> If `cutoffAt` is omitted, defaults to `floor(now to current hour) − 2 hours`.

---

#### `submitPayoutBatch` — DRAFT → PENDING_APPROVAL

```graphql
mutation SubmitBatch($batchId: ID!) {
  submitPayoutBatch(batchId: $batchId) {
    id
    referenceId
    status
    updatedAt
  }
}
```

---

#### `approvePayoutBatch` — PENDING_APPROVAL → APPROVED

```graphql
mutation ApproveBatch($input: ApprovePayoutBatchInput!) {
  approvePayoutBatch(input: $input) {
    id
    referenceId
    status
    approvedAt
    approvedBy
  }
}
```

Variables:
```json
{
  "input": {
    "batchId": "<uuid>",
    "notes": "Verified against job report — approved"
  }
}
```

---

#### `processPayoutBatch` — APPROVED → PROCESSING → COMPLETED/FAILED

```graphql
mutation ProcessBatch($batchId: ID!) {
  processPayoutBatch(batchId: $batchId) {
    id
    referenceId
    status
    processingStartedAt
    completedAt
    totalNetAmount
  }
}
```

---

#### `cancelPayoutBatch` — Cancel before PROCESSING

```graphql
mutation CancelBatch($input: CancelPayoutBatchInput!) {
  cancelPayoutBatch(input: $input) {
    id
    referenceId
    status
    cancelledAt
  }
}
```

Variables:
```json
{
  "input": {
    "batchId": "<uuid>",
    "reason": "Discrepancy found — regenerating"
  }
}
```

---

#### `updatePayoutScheduleConfig` — Update scheduled payout settings

```graphql
mutation UpdateSchedule($input: UpdatePayoutScheduleConfigInput!) {
  updatePayoutScheduleConfig(input: $input) {
    scheduleType
    isEnabled
    cronExpression
    defaultCutoffHours
    updatedAt
  }
}
```

---

## 9. Subcontractor Routing

When a subcontractor completes a job, their payout goes to the **parent contractor's** bank account, not the subcontractor's own account.

**How it works:**

Jobs have a `payoutContractorId` field. When a subcontractor accepts a job, this field is set to the parent contractor's ID. The batch generator uses:

```sql
COALESCE(jobs.payout_contractor_id, jobs.contractor_id)
```

So:
- **Regular contractor's job:** `payoutContractorId = null` → uses `contractorId` (their own)
- **Subcontractor's job:** `payoutContractorId = parentContractorId` → routed to parent's bank

The parent contractor sees the combined payout for all their subcontractor's jobs in their single batch item. Subcontractors themselves have no payout item.

---

## 10. SKIPPED Items — What Causes Them

A contractor item is marked `SKIPPED` at **batch generation time** if their profile has:
- No `bankAccountNumber`, OR
- No `bankIfscCode`, OR
- `bankVerified = false`

SKIPPED contractors:
- Appear in the batch record (visible in admin UI)
- Receive no money in this batch cycle
- Their jobs are still `LOCKED` after approval (they participated in the batch)
- **Their jobs are NOT eligible for the next batch** unless admin cancels the batch first (which unlocks them)

> This is the critical reason contractors must set up bank details before the first batch that includes their jobs is generated.

**Note:** Even if `beneficiaryRegistered = false`, a contractor is NOT skipped at generation time — the batch generator only checks `bankVerified` and the presence of account/IFSC. The `ensureBeneficiary` call happens at **process time** (Step 6) as a safety net.

---

## 11. Audit Trail

Every significant action writes to `mp_payout_audit_logs`. Audit events:

| Event | When |
|-------|------|
| `BATCH_CREATED` | generateBatch() |
| `BATCH_SUBMITTED` | submitBatch() |
| `BATCH_APPROVED` | approveBatch() — includes `lockedJobs` count |
| `BATCH_CANCELLED` | cancelBatch() — includes `unlockedJobs` count + reason |
| `BATCH_PROCESSING_STARTED` | processBatch() begins |
| `ITEM_TRANSFER_INITIATED` | Per-item: transfer about to be sent |
| `ITEM_SUCCESS` | Per-item: transfer succeeded — includes UTR |
| `ITEM_FAILED` | Per-item: transfer failed — includes error |

Query the full audit trail:

```graphql
query {
  payoutAuditLogs(batchId: "<uuid>") {
    action
    actorRole
    details        # JSON — varies per event
    createdAt
  }
}
```

---

## 12. Production Checklist

| Item | Status |
|------|--------|
| `CASHFREE_PAYOUT_CLIENT_ID` set in env | ⚠️ Required before deploying |
| `CASHFREE_PAYOUT_CLIENT_SECRET` set in env | ⚠️ Required before deploying |
| `NODE_ENV=production` for live Cashfree URLs | ⚠️ Required before deploying |
| Cashfree Payouts webhook URL registered in dashboard | ⚠️ Required |
| Cashfree Payouts account activated (not just PG account) | ⚠️ Separate product — needs separate activation |
| DB migration `0026_black_toxin.sql` applied | ✅ Adds `beneficiary_registered` column |
| `beneficiaryRegistered` flag on contractor profiles | ✅ Implemented |
| Cashfree Payouts v2 API (no Bearer token) | ✅ Implemented |
| Idempotent transfer IDs (`PO-{batchItemId}`) | ✅ Prevents duplicate transfers on retry |
| Per-item error isolation (one failure doesn't abort others) | ✅ Implemented |
| Subcontractor job routing to parent bank | ✅ Implemented via `payoutContractorId` |
| Full audit log on every state change | ✅ Implemented |
| Jobs unlocked on batch cancel | ✅ Implemented |
| 2-hour cutoff buffer for immediate batches | ✅ Implemented (hour-floored) |
| Cashfree Payout webhook handler for async transfer resolution | ⚠️ Handler exists but webhook URL must be configured |
