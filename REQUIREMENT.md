MESCOM Smart Meter Installation System
System Architecture & End-to-End Process Flow

1. Purpose of the System
This system digitizes and controls the complete lifecycle of smart meter installation
for new connections under MESCOM by integrating consumer onboarding, site
readiness, inventory control, contractor execution, and billing readiness into a single
governed platform.

The design prioritizes:
- Field realism
- Inventory accountability
- Contractor fairness
- Audit and billing accuracy


2. Core Design Principles
- Inventory control remains with the Retail Outlet (RO) level
- One meter can be mapped to only one consumer at a time
- Meter–consumer mapping is provisional until installation
- Site readiness is verified before contractor assignment
- All actions are QR, geo, and time-stamped


3. User Roles & Responsibilities

3.1 Admin
- Upload consumer master data (Excel)
- Maintain master records:
  - Circle, Division, Subdivision
  - Retail Outlet (RO) with login ID generation
  - Contractor master
  - Meter master attributes
- Monitor dashboards


3.2 Consumer
- Log in using the mobile number
- Update/confirm:
  - Address
  - Contact details
- Upload mandatory geo-tagged site photo
- Track installation status


3.3 Retail Outlet (RO) – under the subdivision assigned
- Custodian of smart meter inventory
- Scan and maintain meter stock using QR
- Verify consumer site readiness based on:
  - Geo-tagged photo
  - Electrical readiness
  - Network availability
  - Meter requirement (phase, prepaid/postpaid, voltage)
- Verify meter installation
- Issue physical meters to contractors
- Assign meter to consumer via QR scan
- Verify installed meters


3.4 Contractor
- Collect meters from RO
- Install meters at consumer premises
- Capture installation evidence:
  - Geo-tagged photos
  - Initial meter reading
- Update installation or failure status


4. System Modules (Architecture View)
1. Consumer Management Module
2. Site Verification Module (RO-led)
3. Inventory & Meter Registry Module
4. Installation & Commissioning Module
5. Billing Integration Module

All modules share a central Meter Registry.


Meter Inventory Management (RO)
- RO scans QR of incoming meters
- RO uploads Excel file with meter serial number,
  non-RAPDRP or RAPDRP status, or updates
  non-RAPDRP/RAPDRP status during QR scan

Meter attributes stored:
- Serial number
- non-RAPDRP or RAPDRP
- Network compatibility
- Phase (1P/3P, AEH/Non-AEH)
- Prepaid/Postpaid

Meter status: Available at RO


5. End-to-End Process Flow

Step 1: Consumer Master Upload
- Admin uploads Excel with:
  - Consumer Number, Name, Mobile
  - Connection Type, Tariff
  - Supply Voltage, Meter Type
- Consumer mapped to:
  Circle → Division → Subdivision → RO


Step 2: Consumer KYC Verification by RO
- RO reviews consumer submission
- Verifies:
  - Site accessibility
  - Electrical readiness
  - Network availability (Airtel / Jio / BSNL / None)
  - Estimated Installation Date
  - Required meter specifications
- RO updates readiness status
- Sends welcome message via SMS and email with customer app link

Consumer actions:
- Logs in
- Uploads geo-tagged site photo (mandatory)
- Confirms network availability
- Confirms address and contact details

System status: Request pending


Step 3: Appointment Scheduling
- Appointment scheduled by Retail Outlet
- RO calls the consumer around the estimated installation date


Step 4: Contractor Collects the Meter
- Contractor visits RO
- Provides consumer ID or mobile number
- RO creates contractor login if not registered
- Contractor logs in using mobile number & OTP
- RO searches ready consumers using:
  - Mobile number
  - Name
  - Location filters
- RO scans meter QR against consumer ID

System validations:
- Meter availability
- Meter–consumer specification match
  (non-RAPDRP/RAPDRP, network, phase)

If valid:
- Meter status → Reserved
- Consumer status → Meter Assigned
- RO physically issues the meter


Step 5: Installation & Commissioning
- Contractor installs the meter
- Captures:
  - Geo-tagged installation photos
  - Seal verification
  - Initial meter reading using OCR
- System updates:
  - Meter → Installed
  - Consumer → Commissioned
- Billing system fetches meter details and reading via QR & OCR


Step 6: Verification (RO)
- RO verifies:
  - Geo-tagged photos
  - Seal
  - Communication LED
  - Initial meter reading


6. Meter Lifecycle States
- Available at RO → Reserved → Installed
- Reserved → Failure/Timeout → Available at RO


7. Conclusion
This architecture aligns with MESCOM’s operational reality while remaining scalable
for RDSS smart metering programs. It minimizes manual intervention, enforces
accountability, and ensures a smooth transition from connection approval to billing.
