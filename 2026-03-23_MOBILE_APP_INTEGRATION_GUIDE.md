# Mobile App Integration Guide (v2 Marketplace Flow)
**Date:** 2026-03-23

This document outlines the major flow changes and new GraphQL APIs required by the **Consumer App** and **Contractor App** developers following the massive architectural update to the Marketplace system.

---

## 1. The Post-Acceptance Payment Flow (CRITICAL UPDATE)

The biggest change in this version is the shifting of the payment step. Consumers no longer pay *before* the contractor accepts the job. 

### Old Flow (Deprecated):
1. Consumer searches & books.
2. Consumer pays upfront via Cashfree.
3. Job is created in `REQUESTED` state.
4. Contractor `acceptJob` or `rejectJob`.

### New Flow (Current):
1. Consumer searches & books.
2. Job is created in `REQUESTED` state immediately (No payment prompt yet).
3. Contractor receives notification and calls `acceptJob`.
4. Job state moves to `PAYMENT_PENDING`.
5. Consumer receives notification that the contractor has accepted the request and is waiting for payment.
6. Consumer opens app, goes to effectively a "Pay Now" screen, and taps Pay.
7. System initiates payment, job moves to `PAID`, and then work execution goes to `STARTED`.

---

## 2. CONSUMER APP — New APIs & Flow Updates

### A. Location GPS Auto-Detection (New Feature)
Instead of forcing the user to manually select a state/district/village, the app can now use their GPS location to find the exact village ID they are currently standing in.

**Query:** `nearestVillage` (Open API, no auth required)
```graphql
query NearestVillage($lat: Float!, $lng: Float!, $radiusKm: Float) {
  nearestVillage(lat: $lat, lng: $lng, radiusKm: $radiusKm) {
    id
    name
    category
    distance_km
  }
}
```
*App UI impact:* Add a "Use My Location" button that grabs GPS, calls this query, and auto-fills the village dropdowns.

### B. Searching for Contractors
The backend `searchContractors` query has been heavily modified internally. It automatically handles hiding blacklisted contractors and automatically includes "Subcontractors" operating under a premium parent.

**Query:** `searchContractors` (Existing, but added fields)
```graphql
query SearchContractors($filters: ContractorSearchInput!) {
  searchContractors(filters: $filters) {
    items {
      contractorId
      contractorName
      ratingAvg
      # NEW FIELDS
      isAvailableForJobs
      payoutContractorId
      commissionPercent
    }
  }
}
```
*App UI Impact:* If `isAvailableForJobs` is `false`, the contractor should appear in the list but with a greyed-out "Not Available" badge, and the "Book" button should be disabled.

### C. Booking the Job
Still use the existing `createMarketplaceJob` mutation. But note: it will NO LONGER return a `paymentSessionId` or `paymentUrl`. It simply returns the Job ID in state `REQUESTED`.

### D. The "Pay Now" Trigger
When the Job hits `PAYMENT_PENDING` (after the contractor accepts it), the Consumer app must present a "Pay Now" UI.

**Mutation:** `initiateMarketplacePayment` (New)
```graphql
mutation InitiateMarketplacePayment($jobId: ID!) {
  initiateMarketplacePayment(jobId: $jobId) {
    paymentSessionId
    paymentUrl
  }
}
```
*App UI Impact:* Open the `paymentUrl` via in-app browser or SDK. On success, the backend webhook changes the state to `PAID` / `STARTED`. 

> [!IMPORTANT]
> **Webhook Pending Automation**: The backend currently relies on manual **Cashfree Dashboard** configuration for webhooks. The automated `notify_url` injection is **PENDING** and awaits `BACKEND_URL` environment setup. until then, ensure the server's public URL is manually registered in the Dashboard.

---

## 3. CONTRACTOR APP — New APIs & Flow Updates

### A. Online/Offline Toggle (New Feature)
Contractors can now manually broadcast if they are too busy to take new requests. Turning this off hides them from Consumer searches.

**Mutation:** `toggleMyAvailability` (New)
```graphql
mutation ToggleMyAvailability($available: Boolean!) {
  toggleMyAvailability(available: $available) {
    success
    message
  }
}
```
*App UI Impact:* Add a large toggle switch on the Contractor Dashboard Home. Retrieve initial state from `marketplaceContractorProfile.isAvailableForJobs`.

### B. Subcontractor Management (For Premium Contractors Only)
A premium contractor can manually link FREE tier contractors under their umbrella. The FREE contractors will borrow the parent's subscription benefits and coverage areas, but all paychecks route to the parent.

**Query:** `searchSubcontractorCandidates` (Or general search by phone)
You can search all users by phone/email to find the ID of the person you want to add. *(Standard user queries apply)*

**Mutation:** `createSubContractor` (New)
The Premium Parent creates a new subcontractor and assigns their working locations in one go.
```graphql
mutation CreateSubContractor($input: CreateSubContractorInput!) {
  createSubContractor(input: $input) {
    success
    message
  }
}

# Input Example
# {
#   "input": {
#     "name": "John Doe",
#     "phone": "9876543210",
#     "coverageAreas": [
#       { "selectionType": "VILLAGE", "referenceCode": "VIL123", "displayName": "Village A" }
#     ]
#   }
# }
```

**Mutation:** `removeSubContractor` (New)
```graphql
mutation RemoveSubContractor($subContractorId: ID!) {
  removeSubContractor(subContractorId: $subContractorId) {
    success
    message
  }
}
```
*App UI Impact:* Create a new screen in the navigation called "My Crew" or "Subcontractors". Allow the Premium Parent to add/remove IDs. Note: The backend will throw an error if the parent tries to add more subcontractors than their Subscription Plan quota allows.

### C. Accepting Jobs
Still use the existing `acceptJob` mutation. 
*App UI Impact:* After calling `acceptJob`, the UI should inform the contractor: *"Waiting for Consumer Payment. You will be notified when they have paid."* The job will sit in `PAYMENT_PENDING` until the consumer completes step 1.D (above). 

### D. Subcontractor App View
If the logged-in user is technically a "Subcontractor" to someone else:
1. They cannot buy subscriptions.
  2. They do not need to configure "Coverage Areas" (the Parent assigns these during creation or edit).
3. Jobs will arrive to them normally. They accept/reject jobs normally.
4. *App UI Impact:* The app should detect if they have a parent mapping and perhaps show a banner: "You are operating as a Subcontractor for [Agency Name]. All payouts are managed centrally."
