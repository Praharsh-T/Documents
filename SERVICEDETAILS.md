
ELECTRICAL WORKS – SERVICE CATALOGUE & PRICE LIST

1. Energy Meter Services
Service Name | UOM | Urban (₹) | Semi-Urban (₹) | Rural (₹)
Single Phase Energy Meter Installation | Per Job | 600 | 500 | 400
Three Phase Energy Meter Installation | Per Job | 1,200 | 1,000 | 800
Single Phase Meter Replacement | Per Job | 500 | 450 | 400
Three Phase Meter Replacement | Per Job | 1,000 | 900 | 800
Meter Relocation (within premises) | Per Job | 800 | 700 | 600
Meter Inspection & Sealing | Per Job | 300 | 250 | 200

2. Wiring & Cabling
Service Name | UOM | Urban (₹) | Semi-Urban (₹) | Rural (₹)
Internal Wiring (per point) | Per Point | 350 | 300 | 250
External Service Cable Laying | Per Meter | 120 | 100 | 80
Conduit Pipe Wiring | Per Meter | 180 | 150 | 120
Cable Replacement (single run) | Per Job | 700 | 600 | 500

3. Distribution Board (DB) & Panel Works
Service Name | UOM | Urban (₹) | Semi-Urban (₹) | Rural (₹)
Distribution Board Installation (Up to 8 way) | Per Job | 1,200 | 1,000 | 900
MCB Installation / Replacement | Per Unit | 300 | 250 | 200
RCCB / ELCB Installation | Per Job | 800 | 700 | 600
Load Balancing & DB Rewiring | Per Job | 1,500 | 1,300 | 1,100

4. Earthing & Safety Works
Service Name | UOM | Urban (₹) | Semi-Urban (₹) | Rural (₹)
Pipe Earthing Installation | Per Job | 2,500 | 2,200 | 2,000
Plate Earthing Installation | Per Job | 3,000 | 2,700 | 2,400
Earthing Resistance Testing | Per Job | 600 | 500 | 400
Surge Protection Device Installation | Per Job | 900 | 800 | 700

5. Inspection, Testing & Certification
Service Name | UOM | Urban (₹) | Semi-Urban (₹) | Rural (₹)
Electrical Safety Inspection | Per Visit | 500 | 400 | 300
Load Assessment Inspection | Per Visit | 600 | 500 | 400
Electrical Fitness Certificate Issuance | Per Job | 1,000 | 900 | 800
Fault Diagnosis (without repair) | Per Visit | 400 | 350 | 300

6. Add-On & Special Services
Service Name | UOM | Urban (₹) | Semi-Urban (₹) | Rural (₹)
Emergency / Same-Day Service | Per Job | 500 | 400 | 300
Additional Visit | Per Visit | 300 | 250 | 200
Night-Time Service | Per Job | 400 | 350 | 300

7. Catalogue Rules (for Developers)
- Prices are service charges only (materials extra unless specified).
- Location pricing resolved via LGD + Urban/Semi-Urban/Rural tag.
- One service = one UOM.
- Final payable = Base Price + Add-ons + Taxes (if enabled).
- Applicable SLAs to be mapped per service.

------------------------------------------------------------

SERVICE LEVEL AGREEMENT (SLA) TEMPLATE

1. SLA Metadata
SLA ID: System-generated unique identifier
SLA Name: Human-readable SLA name
Applicable Services: One or more services from catalogue
Area Type: Urban / Semi-Urban / Rural / All
Effective From: Start date
Effective To: End date (optional)
Active Status: Active / Inactive

2. SLA Timelines

2.1 Request Acceptance SLA
Urban: 12 Hours
Semi-Urban: 24 Hours
Rural: 48 Hours

System Behaviour:
- Countdown starts at service request creation.
- If no action within SLA, status changes to Auto-Expired or Auto-Reassigned (configurable).

2.2 Job Start SLA
Urban: 24 Hours
Semi-Urban: 24 Hours
Rural: 48 Hours

System Behaviour:
- Job can only start after OTP verification.
- SLA clock starts after contractor acceptance.

2.3 Job Completion SLA
Urban: 72 Hours
Semi-Urban: 72 Hours
Rural: 72 Hours

System Behaviour:
- Job completion requires OTP verification.
- SLA stops on successful job closure.

3. OTP Enforcement Rules
Job Start: OTP Required
Job Completion: OTP Required
OTP validity: Configurable (default 10 minutes)
OTP retry attempts: Configurable (default 3)

4. SLA Breach Handling
Acceptance SLA Breach:
- Auto-cancel or reassign job
- Notification to contractor and admin
- Breach recorded in contractor profile

Job Start SLA Breach:
- Warning notification
- Admin escalation flag

Job Completion SLA Breach:
- Admin notified
- Consumer informed

5. Notifications & Alerts
- SLA nearing breach (80%): Contractor
- SLA breach: Contractor & Admin
- Job delayed: Consumer

6. SLA Exclusions (Admin Controlled)
- Consumer unavailable for OTP
- Force majeure
- Consumer-requested reschedule
(Admin must provide reason codes)

7. SLA Metrics & Reporting
- Acceptance SLA compliance %
- Job completion SLA compliance %
- Average service time
- Contractor-wise SLA breach count

Usage:
- Contractor performance rating
- Service eligibility
- Incentives / penalties

8. SLA Configuration Rules
- One SLA per Service + Area Type
- SLA versioning mandatory
- SLA applied based on booking timestamp
- SLA changes do not affect ongoing jobs

9. Sample SLA Configuration
Service: Single Phase Energy Meter Installation
Area Type: Urban
Acceptance SLA: 30 mins
Job Start SLA: 4 hours
Job Completion SLA: 24 hours
OTP mandatory: Yes
