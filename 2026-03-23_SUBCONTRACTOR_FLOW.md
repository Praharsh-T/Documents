# Subcontractor Architecture & Flow

> **Bug Fix (27 March 2026):** `getSubcontractor` query returned "not found or access denied" for valid subcontractors. Fixed in `contractor-profile.service.ts`:
> - Moved `parentContractorId` filter into the `INNER JOIN` condition (was in `WHERE`, causing lookup failures).
> - Now accepts `contractorMarketplaceProfile.id`, `contractors.id`, **or** `users.id` as the `subcontractorId` argument — robust for all mobile app ID formats.

This document details the features, permission model, and Consumer/Contractor application flows to transition from the old Marketplace system to the new system regarding Subcontractors.

## 1. Feature Definition & Permissions

Subcontractors operate under the umbrella of a **PREMIUM** Contractor (the "Parent" or "Agency").
Subcontractors do not require separate user accounts or distinct login systems. Instead, they operate entirely based on mappings tying them to their Parent.

### How it is Implemented:

1. **Mapping Over Identity:**
    - Premium Parents create a brand new subcontractor account from scratch via the `createSubContractor` mutation.
    - This automatically provisions the User (Phone/Name), Contractor Profile, and Mapping in one transaction.
   
2. **Subscription & Commission Propagation:**
   - Subcontractors inherent the ZERO commission benefits of their parent's PREMIUM plan.
   - If the Parent's PREMIUM plan expires, the backend cron-job `checkExpiredSubscriptions()` flags all mapped Subcontractors with `is_suspended_pending_plan = true`. They are hidden from Consumer search results and cannot receive jobs until the Parent renews.

3. **Job Routing (`payoutContractorId`):**
   - When a Consumer views a Subcontractor and initiates a job, the Subcontractor receives the job assignment directly.
   - However, the newly added `mp_jobs.payout_contractor_id` is automatically set to the Parent’s ID.
   - Financial payouts are routed exclusively to the Parent. The Parent handles offline/external compensation to the Subcontractor.

4. **Location Scope:**
   - Subcontractors use their own explicitly assigned coverage areas (set by the Parent during creation) for search matching. They do NOT inherit the parent's entire territory, allowing fine-grained regional control for large agencies.

5. **Limits per Plan:**
   - The Parent's active Subscription Plan dictates the maximum number of Subcontractors they can employ via the `max_subcontractors` field (configurable by Admins).

---

## 2. Transition Plan: What to Change in the Applications

To migrate from the "Old Flow" to the "New Flow" in the Consumer & Contractor Apps, implement the following steps:

### Contractor App Changes

1. **Subcontractor Management Dashboard:**
   - Add a new route (`/contractors/marketplace/subcontractors`) for PREMIUM Parent contractors.
   - **Features:**
     - **Create New:** Ability to create a brand new subcontractor by providing their Name, Phone, and allowed Coverage Areas (Village/District).
     - Call `createSubContractor(input: {name, phone, coverageAreas})`.
     - Data-grid listing all currently assigned Subcontractors using the new `getMySubcontractors` query.
     - **Metadata Tracking:** The API now returns `subcontractorDetails` (containing `mappedAt`, `assignedAreas`, and `isSuspendedPendingPlan`) for full visibility.
     - Call `removeSubContractor(subContractorId)` to detach a Subcontractor.
     - Call `toggleSubContractorActive(subContractorId, isActive)` to temporarily pause a crew member.
   - **Enforcement:** Ensure that the UI restricts adding Subcontractors if `activeSubcontractors >= maxSubcontractors` defined in the Parent's plan.

2. **Subcontractor Job View:**
   - For Subcontractors viewing their incoming jobs, nothing materially changes—they accept/reject/complete jobs normally. 
   - However, they need a UI banner indicating they are operating as a Subcontractor for `<Parent Name>` and that commissions/payouts are handled centrally.
   - **Data Fetching:** The `marketplaceContractorProfile` query now exposes a `parentContractor` object containing the agency's Name and Phone so the Mobile App can automatically render this context banner.

3. **Availability Toggle:**
   - Implement the new "Availability Toggle" on the Contractor Dashboard. Contractors must manually toggle `isAvailableForJobs` to `false` when they are booked up to stop appearing in Consumer searches.

### Consumer App Changes

1. **New Job Acceptance / Payment Flow:**
   - **OLD Flow:** Consumer searches -> Books -> Pays Upfront -> Job enters Request state -> Contractor accepts.
   - **NEW Flow:** Consumer searches -> Requests Job -> Job enters `REQUESTED` State -> Contractor App gets notification -> Contractor Accepts -> Job enters `PAYMENT_PENDING` -> Consumer gets Notification to Pay -> Consumer accesses "Pay Now" screen in app -> Job transitions to `PAID` / `STARTED`.
   - **Action Required:** Build the "Pay Now" Screen in the Consumer app explicitly handling `PAYMENT_PENDING` state jobs and executing `initiateMarketplacePayment`.

2. **Search Results Filtering:**
   - Update the Consumer search views. The `searchContractors` query already includes Blacklisted and Suspend logic.
   - If `ContractorMarketplaceProfile.isBlacklisted == true` or `isAvailableForJobs == false`, gracefully hide them or disable their "Book" button with an "Unavailable" badge.
   - Ensure the UI conveys when a Contractor is a Subcontractor representing a premium Agency (if desired for transparency).

3. **GPS Nearest Village Lookup:**
   - Provide a "Use My Location" toggle in the search widget.
   - Execute the `nearestVillage(lat, lng)` query to automatically map the Consumer's GPS to the exact `villageId` required to view accurate Pricing and Services availability.
