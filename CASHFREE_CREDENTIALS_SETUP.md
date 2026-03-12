# Cashfree Credentials Setup Guide

## Issue: 401 Unauthorized Error

When calling `initiateSubscriptionUpgrade` or `initiateMarketplacePayment`, you're getting:
```
"message": "Request failed with status code 401"
```

**Root Cause**: The Cashfree API credentials (`CASHFREE_APP_ID` and `CASHFREE_SECRET`) in your `.env` file are either expired, invalid, or not activated.

---

## Solution: Generate New Cashfree Credentials

### Step 1: Access Cashfree Dashboard

1. Go to [Cashfree Merchant Dashboard](https://merchant.cashfree.com/)
2. Login with your account credentials
3. If you don't have an account, sign up at [Cashfree Signup](https://www.cashfree.com/payment-gateway/)

### Step 2: Navigate to Credentials Section

1. After login, go to **Developers** → **API Keys** in the dashboard
2. You'll see two environments:
   - **Sandbox** (for testing) - Use this for development
   - **Production** (for live) - Use this for production deployment

### Step 3: Generate Sandbox Credentials

1. Click on **Sandbox** environment
2. If credentials don't exist, click **Generate Credentials**
3. If credentials exist but are expired, click **Regenerate**
4. You'll get:
   - **App ID** (e.g., `TEST109944128e970adfb7a77d02c31a21449901`)
   - **Secret Key** (e.g., `cfsk_ma_test_9913b30e69050ef6072b6a1d3ad58137_720892f6`)

⚠️ **IMPORTANT**: Copy these immediately as the Secret Key is shown only once!

### Step 4: Update Your .env File

Open `/Users/mac/Projects/antigravity-workspace/user-service/.env` and update:

```env
# Cashfree Payments
CASHFREE_APP_ID=YOUR_NEW_APP_ID_HERE
CASHFREE_SECRET=YOUR_NEW_SECRET_KEY_HERE
```

### Step 5: Restart Backend

```bash
cd /Users/mac/Projects/antigravity-workspace/user-service
# Stop the current backend (Ctrl+C in the terminal)
pnpm run start:dev
```

You should see in the logs:
```
[CashfreeService] Cashfree initialized in SANDBOX mode
[CashfreeService] Using App ID: TEST109944128e9...
[CashfreeService] API Version: 2023-08-01
```

### Step 6: Test Again

Now try the subscription upgrade from your mobile app again. The 401 error should be resolved.

---

## For Production Deployment

When deploying to production:

1. Generate **Production** credentials from Cashfree dashboard
2. Set `NODE_ENV=production` in your production environment
3. Update production .env with production credentials:
   ```env
   NODE_ENV=production
   CASHFREE_APP_ID=YOUR_PRODUCTION_APP_ID
   CASHFREE_SECRET=YOUR_PRODUCTION_SECRET
   ```
4. The backend automatically switches to `CFEnvironment.PRODUCTION` mode

---

## Webhook Configuration

After getting valid credentials, you also need to configure webhooks in Cashfree dashboard:

1. Go to **Developers** → **Webhooks** in Cashfree dashboard
2. Set Webhook URL to: `https://your-domain.com/payments/cashfree/webhook`
3. Select events:
   - `PAYMENT_SUCCESS_WEBHOOK`
   - `PAYMENT_FAILED_WEBHOOK`
   - `REFUND_STATUS_WEBHOOK`
4. Save the configuration

---

## Troubleshooting

### Still Getting 401 After Updating Credentials?

1. **Check credential format**:
   - App ID should start with `TEST` for sandbox or have production format
   - Secret Key should start with `cfsk_ma_test` for sandbox

2. **Verify environment**:
   ```bash
   # Check what's in your .env
   cat /Users/mac/Projects/antigravity-workspace/user-service/.env | grep CASHFREE
   ```

3. **Check backend logs** for detailed error:
   - Look for `[CashfreeService]` logs
   - Check if initialization succeeded
   - Look for detailed error messages in createOrder failure logs

4. **Verify Cashfree account status**:
   - Ensure your Cashfree account is active
   - Check if Sandbox environment is enabled
   - Verify no IP restrictions are set (common issue)

5. **Test with Cashfree's test payment flow**:
   - Use test card details provided by Cashfree
   - Verify your account can process test payments

---

## Backend Code Changes Made

I've enhanced the error logging in `cashfree.service.ts` to provide better diagnostics:

- **Enhanced 401 error detection**: Logs specific message when authentication fails
- **Debug logging**: Shows order creation attempts and successes
- **Better error context**: Includes HTTP status, status text, and response data
- **Initialization logging**: Shows App ID (partially masked), API version, and environment

These logs will help diagnose any future issues quickly.

---

## Quick Test Command

Once credentials are updated, you can quickly test if Cashfree connection works by checking the backend logs during startup. You should see:

```
✅ [CashfreeService] Cashfree initialized in SANDBOX mode
✅ [CashfreeService] Using App ID: TEST109944128e9...
✅ [CashfreeService] API Version: 2023-08-01
```

If you see this, credentials are loaded. Then test the payment flow from mobile app.

---

## Security Notes

1. **Never commit credentials to git**: Ensure `.env` is in `.gitignore`
2. **Use environment variables in production**: Don't hardcode credentials
3. **Rotate credentials periodically**: Generate new keys every few months
4. **Separate sandbox and production**: Never use test credentials in production
5. **Webhook signature verification**: Always enabled (already implemented in the code)

---

## Need Help?

If issues persist after updating credentials:
1. Check Cashfree Dashboard for API call logs
2. Contact Cashfree support with your App ID (don't share Secret Key)
3. Review backend logs for detailed error messages
