# SmartMeter - DLT SMS Template Registration Guide

**Last Updated:** 17 February 2026

---

## What is DLT?

**DLT (Distributed Ledger Technology)** is a mandatory registration system by TRAI (Telecom Regulatory Authority of India) for all commercial SMS in India. All SMS templates must be pre-registered and approved before they can be sent.

---

## Registration Process

### Step 1: Register as Principal Entity
1. Go to your telecom operator's DLT portal:
   - Jio: https://trueconnect.jio.com
   - Airtel: https://www.airtel.in/business/commercial-communication
   - Vi: https://www.aboris.in
   - BSNL: https://www.ucc-bsnl.co.in

2. Register as **Principal Entity** (business sending SMS)
3. Get your **Principal Entity ID (PEID)**

### Step 2: Register Sender ID (Header)
- Header: `SMRTMR` or `KLONEC` (6 characters, alphabetic)
- This appears as the sender name

### Step 3: Register Templates
Submit the templates below for approval (takes 1-7 days)

---

## SmartMeter SMS Templates (2 Total)

### 1. OTP Template ✅ ALREADY REGISTERED
**Template Name:** `SmartMeter_OTP`
**Category:** OTP
**Status:** Active

**Template ID:** `1107171707202035772`
**Fast2SMS Message ID:** `180913`

**Template Content:**
```
Your {#var#} OTP is {#var#} for {#var#}. Valid for {#var#} minutes. Do not share with anyone.
```

**Variables:**
| Variable | Position | Description | Example |
|----------|----------|-------------|---------|
| {#var#} | 1 | App/Brand name | SmartMeter |
| {#var#} | 2 | 6-digit OTP code | 123456 |
| {#var#} | 3 | Purpose/Reason | login, registration, password reset |
| {#var#} | 4 | Validity in minutes | 5 |

**Example SMS:**
```
Your SmartMeter OTP is 123456 for login. Valid for 5 minutes. Do not share with anyone.
```

---

### 2. Registration Welcome Template 🔄 TO BE REGISTERED
**Template Name:** `SmartMeter_Registration`
**Category:** Transactional
**Purpose:** Send to consumer/contractor when admin registers them. Both use OTP-based login.

**Template Content:**
```
Welcome to {#var#}! You are registered with mobile {#var#}. Download the app from Play Store to get started.
```

**Variables:**
| Variable | Position | Description | Example |
|----------|----------|-------------|---------|
| {#var#} | 1 | App/Brand name | SmartMeter |
| {#var#} | 2 | Mobile number | 9876543210 |

**Character Count:** ~105 characters ✓

**When to send:** After admin creates a new consumer or contractor account

---

## Summary Table

| # | Template | Status | Used For | Variables |
|---|----------|--------|----------|-----------|
| 1 | OTP | ✅ Active | All logins, verifications | appName, OTP, purpose, minutes |
| 2 | Registration | 🔄 Pending | Consumer & Contractor | appName, mobile |

---

## OTP Purpose Values

When sending OTP, use these standardized purpose strings:

| Purpose | Description |
|---------|-------------|
| `login` | User logging into the app |
| `registration` | New user registration verification |
| `password reset` | Password reset verification |
| `phone verification` | Verifying phone number |
| `transaction` | Transaction verification |

---

## Environment Configuration

Add to your `.env` file:

```env
# SMS Gateway
SMS_GATEWAY_ENABLED=true
FAST2SMS_API_KEY=your-fast2sms-api-key

# App Name (used in SMS templates)
APP_NAME=SmartMeter

# DLT Template IDs
DLT_TEMPLATE_OTP=1107171707202035772
DLT_TEMPLATE_REGISTRATION=  # Add after TRAI approval

# Fast2SMS Message IDs
FAST2SMS_MSG_ID_OTP=180913
FAST2SMS_MSG_ID_REGISTRATION=  # Add after Fast2SMS approval
```

---

## Testing Without DLT

Until templates are approved, use **DEV mode**:

```env
SMS_GATEWAY_ENABLED=false
OTP_MODE=DUMMY
OTP_DUMMY_CODE=123456
```

This logs all SMS to console instead of sending.

---

## Character Limits

| Category | Max Characters |
|----------|---------------|
| OTP | 160 |
| Transactional | 160 (single SMS) |

**Note:** Keep messages under 160 chars to avoid splitting into multiple SMS (extra cost).

---

## Approval Timeline

| Step | Duration |
|------|----------|
| Principal Entity Registration | 1-3 days |
| Header/Sender ID Approval | 1-3 days |
| Template Approval | 1-7 days |
| **Total** | **3-13 days** |

---

## Next Steps

1. ✅ OTP template already registered
2. 🔄 Register Registration Welcome template with DLT
3. Add template ID to Fast2SMS after TRAI approval
4. Update `.env` with new template ID
5. Enable `SMS_GATEWAY_ENABLED=true` for production

---

## Support

- Fast2SMS: https://fast2sms.com/support
- TRAI DLT Portal: Contact your telecom operator
