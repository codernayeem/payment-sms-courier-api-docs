# bKash Recurring Payment - Complete Flow Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Database Models](#database-models)
3. [Complete API Flow](#complete-api-flow)
4. [Every Scenario & Edge Case](#every-scenario--edge-case)
5. [Webhook Handling](#webhook-handling)
6. [Deduplication & Idempotency](#deduplication--idempotency)
7. [Gateway Configuration & Credentials](#gateway-configuration--credentials)
8. [What You Must Provide to bKash](#what-you-must-provide-to-bkash)
9. [Pre-Go-Live Checklist](#pre-go-live-checklist)
10. [Known Limitations & Risks](#known-limitations--risks)

---

## Architecture Overview

```
Frontend (Next.js)                Backend (Fastify)                    bKash RPP
==================               =================                    =========
                                  
SubscriptionForm.tsx ----POST /public/recurring-bkash--->  Create:
  - customerName                                           - Customer (find/create)
  - customerPhone                                          - RecurringSubscription (status: initiated)
  - amount                                              - Order (status: pending)
  - frequency                                           - Call bKash createSubscription()
  - planId                                              - PaymentTransaction (audit)
                                                        - Return redirectURL
                                  
User <----------- redirect to bKash consent page ------  redirectURL from bKash
  - Enter wallet number
  - Enter OTP
  - Enter PIN
  - Consent given
                                  
    GET /callback?subscriptionRequestId=...&status=... <--- bKash redirects user
                                    |
                                    v
                                  Query bKash by subscriptionRequestId
                                  Update RecurringSubscription with actual bKash subscription ID
                                  Map bKash status to internal status
                                  Redirect to /payment/success or /payment/fail
                                    |
    POST /webhook (Type: PAYMENT) <--- bKash auto-debit fires webhook
                                    |
                                    v
                                  Verify HMAC-SHA256 signature
                                  Find RecurringSubscription
                                  Deduplicate by trxId
                                  Create/Update Order record
                                  Update stats (recurring, customer)
                                  Send email notification
```

### Key Files

| File | Purpose |
|------|---------|
| `backend/src/routes/api/v1/public/recurring-bkash/index.ts` | All public API routes (~900 lines) |
| `backend/src/plugins/app/payment-gateways/bkash-recurring-gateway.ts` | bKash RPP API client (~630 lines) |
| `backend/src/plugins/app/payment-gateways/bkash-errors.ts` | 100+ bKash error codes mapped |
| `backend/src/plugins/app/payment-service.ts` | Shared payment transaction service |
| `backend/src/plugins/app/payment-gateways/gateway-manager.ts` | Gateway factory/availability checks |
| `frontend/components/donate/SubscriptionForm.tsx` | Order form UI |
| `frontend/data/orderConfig.ts` | Recurring payment method configuration |
| `frontend/lib/services/recurring.ts` | API hooks for recurring operations |

---

## Database Models

### RecurringSubscription
```
id                  Int       PK
customerId             Int       FK -> Customer
amount              Decimal   The recurring charge amount
frequency           String    daily | weekly | monthly | yearly
paymentMethod       String    "bkash-recurring"
subscriptionId      String?   Initially: bKash subscriptionRequestId, then updated to actual bKash subscription ID (numeric) on callback
refer               String?   Not used for bKash (SSLCommerz only)
acctNo              String?   Payer wallet number (set on callback/webhook)
dayOfMonth          Int?      Not used for bKash (bKash manages scheduling internally)
weekDay             String?   Not used for bKash
status              String    initiated -> processing -> active -> paused/deactivated/cancelled/payment_failed
startDate           DateTime
endDate             DateTime? Set on expiry/cancellation
nextBillingDate     DateTime? Updated from bKash webhook nextPaymentDate
lastChargedAt       DateTime? Last successful charge timestamp
planId              Int       FK -> SubscriptionPlan
studentId           Int?      Not used for recurring (only regular fund)
totalCharged        Decimal   Sum of all successful charges
successCount        Int       Count of successful payments
failedCount         Int       Count of failed payment attempts
```

### Order (one per charge cycle)
```
id                    Int       PK
customerId               Int       FK -> Customer
recurringSubscriptionId   Int?      FK -> RecurringSubscription (links this charge to the subscription)
amount                Decimal
paymentMethod         String    "bkash-recurring"
paymentMethodDetail   String?   "bKash Recurring"
status                String    pending -> completed/failed/cancelled/refunded
transactionId         String?   bKash trxId (unique per payment cycle)
invoiceNumber         String?   Our generated invoice
isAnonymous           Boolean   Always false for recurring
userName              String?   Customer name
```

### PaymentTransaction (audit trail per charge)
```
id                    Int       PK
orderId            Int       FK -> Order
gateway               String    "bkash-recurring"
gatewayTransactionId  String?   bKash trxId
gatewayPaymentId      String?   bKash paymentId
status                String    initiated/pending/success/failed/cancelled
invoiceNumber         String?
executeResponse       JSON?     Full webhook/response payload
errorResponse         JSON?     Error details if failed
failureReason         String?
statusCode            String?
customerPhone         String?
customerEmail         String?
completedAt           DateTime?
failedAt              DateTime?
```

### Customer
```
id              Int
email           String?
phone           String?
name            String?
totalSpent    Decimal   Incremented on each successful recurring charge
orderCount   Int       Incremented on each successful recurring charge
lastPurchasedAt   DateTime? Updated on each successful charge
```

---

## Complete API Flow

### Step 1: User Submits Recurring Order Form

**Frontend (SubscriptionForm.tsx):**
- User selects "Recurring" order type
- Chooses frequency: Daily or Monthly
- Enters amount (preset chips: 10/20/50/100 for daily, 100/500/1000 for monthly)
- Enters name + phone (required for bKash)
- Selects bKash as payment method
- Anonymous option is DISABLED for recurring
- Confirm modal shown before submission

**API Call:** `POST /api/v1/public/recurring-bkash`
```json
{
  "customerName": "Rahim Ahmed",
  "customerPhone": "01712345678",
  "customerEmail": "rahim@example.com",  // optional
  "amount": 100,
  "frequency": "monthly",
  "planId": 1,
  "message": "Monthly contribution"   // optional
}
```

### Step 2: Backend Validation & Record Creation

**Validation checks (in order):**
1. bKash recurring enabled in site settings (`payment_methods_recurring` includes `bkash-recurring`)
2. bKash recurring gateway is available (`BKASH_RECURRING_API_KEY` and `BKASH_RECURRING_SERVICE_ID` env vars set)
3. At least one of email or phone is provided
4. Phone format validation (BD: +880XXXXXXXXXX, 880XXXXXXXXXX, 01XXXXXXXXX)
5. **Time constraint: Cannot create after 11:30 PM Bangladesh time** (bKash same-day restriction)
6. Fund exists, is active, and has slug `regular` (only regular fund supports recurring)

**Database creates (in order):**
1. **Customer**: Find by email OR phone, create if not found, update if found
2. **RecurringSubscription**: status=`initiated`, paymentMethod=`bkash-recurring`
3. **Order**: status=`pending`, linked to RecurringSubscription

### Step 3: bKash Subscription Creation

**Gateway call:** `bkashGateway.createSubscription()`

**bKash API request:** `POST /api/subscription`
```json
{
  "serviceId": "<BKASH_RECURRING_SERVICE_ID>",
  "startDate": "2026-03-10",
  "endDate": "2028-03-10",          // +2 years max
  "frequencyType": "MONTHLY",        // Mapped from our frequency
  "amount": 100.00,
  "currency": "BDT",
  "payer": "01712345678",           // Optional - pre-fill wallet
  "subscriptionType": "BASIC",
  "maxCapRequired": false,
  "merchantShortCode": "<SHORT_CODE>",
  "payerType": "CUSTOMER",
  "subscriptionRequestId": "<UUID>",
  "redirectURL": "https://api.example.com/api/v1/public/recurring-bkash/callback",
  "callbackURL": "https://api.example.com/api/v1/public/recurring-bkash/webhook"
}
```

**Headers:**
```
x-api-key: <BKASH_RECURRING_API_KEY>
version: v1.0
channelId: 2  (Merchant WEB)
Content-Type: application/json
```

**Response from bKash:**
```json
{
  "subscriptionRequestId": "uuid-xxx-xxx",
  "redirectURL": "https://recurring.pay.bka.sh/pay/xxx",
  "subscriptionStatus": "INITIALIZED"
}
```

### Step 4: Store Request ID & Create Audit Record

- Update RecurringSubscription: `subscriptionId = subscriptionRequestId`, `status = processing`
- Create PaymentTransaction: gateway=`bkash-recurring`, gatewayTransactionId=subscriptionRequestId

### Step 5: Redirect User to bKash

**Response to frontend:**
```json
{
  "success": true,
  "data": {
    "recurringSubscriptionId": 42,
    "orderId": 100,
    "paymentUrl": "https://recurring.pay.bka.sh/pay/xxx",
    "message": "Redirecting to bKash for subscription consent."
  }
}
```

Frontend redirects user to `paymentUrl` (bKash consent page).

### Step 6: User Completes bKash Consent

On bKash page, user:
1. Sees subscription terms (amount, frequency, dates)
2. Enters bKash wallet number
3. Enters OTP
4. Enters bKash PIN
5. Consent granted

### Step 7: Callback (User Redirect Back)

**bKash redirects to:** `GET /api/v1/public/recurring-bkash/callback?subscriptionRequestId=xxx&status=success`

**Backend processing:**
1. Find RecurringSubscription by `subscriptionRequestId` (stored in `subscriptionId` field)
2. **MUST query bKash API** to get actual subscription status (callback params are not reliable alone)
3. Call `bkashGateway.querySubscriptionByRequestId(subscriptionRequestId)`
4. Get back: actual bKash `subscriptionId` (numeric), `payer` (wallet number), `status`
5. Update RecurringSubscription:
   - `subscriptionId` = actual bKash subscription ID (replaces the request ID)
   - `acctNo` = payer wallet number
   - `status` = mapped from bKash status

**Status mapping (bKash -> Internal):**
| bKash Status | Internal Status | Action |
|-------------|----------------|--------|
| SUCCEEDED | active | Redirect to success page |
| VERIFIED | processing | Redirect to success page (pending state) |
| CANCELLED | cancelled | Cancel pending order, redirect to cancel page |
| FAILED | payment_failed | Fail pending order, redirect to fail page |
| INITIALIZED | initiated | Redirect to success with `pending=true` |

**Redirect to frontend:** `/payment/success?amount=100&invoice=INV-xxx&recurring=true&gateway=bkash`

### Step 8: bKash Auto-Debit (Scheduled Payments)

bKash handles all scheduling internally. On each cycle date:

1. bKash sends **PRE_NOTIFICATION webhook** (1-3 days before, except daily)
2. bKash auto-debits the payer's wallet
3. bKash sends **PAYMENT webhook** with result

**No bill-query needed** (unlike SSLCommerz). bKash debits directly.

### Step 9: Payment Webhook Processing

**Endpoint:** `POST /api/v1/public/recurring-bkash/webhook`
**Headers:** `X-Signature: <HMAC-SHA256>`, `Type: PAYMENT`

**Processing:**
1. Verify HMAC-SHA256 signature
2. Route to `handlePaymentWebhook()`
3. Find RecurringSubscription by subscriptionId or subscriptionRequestId
4. Check deduplication (by trxId)
5. Validate amount matches expected
6. For first payment: update existing pending Order
7. For subsequent payments: create new Order record
8. Create PaymentTransaction (audit)
9. Update RecurringSubscription stats (totalCharged, successCount, lastChargedAt, nextBillingDate)
10. Update Customer stats (totalSpent, orderCount, lastPurchasedAt)
11. Send email notification (async, non-blocking)

---

## Every Scenario & Edge Case

### Scenario 1: Happy Path - Successful Subscription + Payments
1. User submits form -> Records created -> Redirect to bKash
2. User completes consent -> Callback updates status to `active`
3. bKash auto-debits on schedule -> PAYMENT webhook -> Order created
4. Repeats every cycle until cancelled/expired

**DB State After First Payment:**
- RecurringSubscription: status=`active`, subscriptionId=bKash ID, successCount=1
- Order #1: status=`completed`, transactionId=trxId
- PaymentTransaction #1: status=`success`
- Customer: totalSpent += amount, orderCount += 1

### Scenario 2: User Closes Browser BEFORE Reaching bKash Page
- RecurringSubscription: status=`processing`, subscriptionId=requestId
- Order: status=`pending`
- **No callback, no webhook** will come
- **Impact:** Orphaned pending records sit in DB
- **Mitigation:** The cron job `reconcilePendingPayments()` will eventually expire stale pending PaymentTransactions. However, the RecurringSubscription stays in `processing` state.
- **RECOMMENDATION:** Add a cron job to auto-cancel RecurringSubscriptions stuck in `initiated` or `processing` for >1 hour

### Scenario 3: User Reaches bKash Page, Then Closes Browser (No Consent)
- Same as Scenario 2: bKash session expires silently
- **No callback, no webhook**
- RecurringSubscription stays in `processing`
- Order stays in `pending`
- **Mitigation:** Same as above - needs cleanup cron

### Scenario 4: User Completes Consent, But Browser Tab Closes Before Callback Redirect
- bKash has created the subscription successfully
- **Callback never hits our server** (user closed tab)
- bKash WILL send a **SUBSCRIPTION webhook** (Type: SUBSCRIPTION) with the subscription status
- **Current handling:** `handleSubscriptionWebhook()` updates RecurringSubscription status and stores subscription ID
- BUT: The first auto-debit PAYMENT webhook will arrive on schedule, which will:
  - Find RecurringSubscription by subscriptionId
  - Find the pending Order and update it to `completed`
  - Set status to `active`
- **Impact:** Subscription works correctly, but user sees the fail page (or nothing). On next visit, they'd see it's active.
- **Known gap:** The `subscriptionId` in our DB may still contain the requestId (not the actual bKash ID) until the SUBSCRIPTION webhook or first PAYMENT webhook arrives. The `findRecurringSubscription()` helper searches by both IDs to handle this.

### Scenario 5: User Cancels on bKash Consent Page
- bKash redirects to callback with `status=cancelled`
- Callback queries bKash API -> status=CANCELLED
- RecurringSubscription: status=`cancelled`
- Pending Order: status=`cancelled`
- **Clean state, no further action needed**

### Scenario 6: bKash Consent Fails (OTP/PIN Error)
- bKash redirects to callback with `status=failure`
- Callback queries bKash -> status=FAILED
- RecurringSubscription: status=`payment_failed`
- Pending Order: status=`failed`

### Scenario 7: Subscription Active, Scheduled Payment Succeeds
- PAYMENT webhook arrives with `paymentStatus: SUCCEEDED_PAYMENT`
- Deduplication check by trxId
- New Order created with status=`completed`
- Stats incremented
- Email sent

### Scenario 8: Subscription Active, Scheduled Payment Fails
- PAYMENT webhook arrives with `paymentStatus: FAILED_PAYMENT`
- Failed Order record created (for visibility)
- Failed PaymentTransaction created (for audit)
- RecurringSubscription: `failedCount += 1`, status=`payment_failed`
- **bKash retry policy:** For non-daily subscriptions, bKash retries 2 times every 3 days
- Retry webhook comes with `paymentStatus: RE_SUCCEEDED_PAYMENT` or `RE_FAILED_PAYMENT`

### Scenario 9: bKash Retries a Failed Payment Successfully
- PAYMENT webhook with `paymentStatus: RE_SUCCEEDED_PAYMENT`
- Treated same as `SUCCEEDED_PAYMENT`
- New Order created, stats incremented
- RecurringSubscription status set back to `active`

### Scenario 10: bKash Retries Exhaust, Payment Still Fails
- All retries fail (RE_FAILED_PAYMENT)
- RecurringSubscription: `failedCount` keeps incrementing
- bKash continues subscription from next cycle (does NOT auto-cancel)
- **The failed cycle amount is lost** - merchant must handle directly with customer
- Status remains `payment_failed` until next successful payment sets it back to `active`

### Scenario 11: User Cancels Subscription (From Our System)
- `POST /api/v1/public/recurring-bkash/:subscriptionId/cancel` (authenticated)
- Backend calls `bkashGateway.cancelSubscription(subscriptionId, reason)`
- RecurringSubscription: status=`cancelled`
- bKash ALSO sends a CANCEL webhook as confirmation

### Scenario 12: User Cancels from bKash App/Customer Service
- bKash sends CANCEL webhook
- `handleCancelWebhook()`: RecurringSubscription status=`cancelled`
- All pending orders set to `cancelled`

### Scenario 13: Subscription Expires (Reaches End Date)
- bKash sends EXPIRY webhook
- `handleExpiryWebhook()`: RecurringSubscription status=`deactivated`, endDate=now
- No further charges

### Scenario 14: Refund Initiated (From Our Admin)
- Admin calls refund via bKash API
- bKash sends REFUND webhook with `paymentStatus: SUCCEEDED_REFUND`
- Original Order: status=`refunded`
- Customer totalSpent decremented
- RecurringSubscription totalCharged decremented

### Scenario 15: Duplicate Webhook (bKash Sends Same Payment Twice)
- PAYMENT webhook arrives with same trxId
- Deduplication check: `prisma.order.findFirst({ where: { transactionId: trxId } })`
- If found: skip, return 200 "Transaction already processed"
- **Stats NOT double-incremented**

### Scenario 16: Webhook Signature Verification Fails
- HMAC-SHA256 check fails
- Return 400 "Invalid signature"
- No DB changes
- Logged as warning

### Scenario 17: Amount Mismatch in Webhook
- PAYMENT webhook has different amount than RecurringSubscription.amount
- Tolerance: 0.01 BDT
- Return 400 "Amount mismatch"
- Logged as error (possible tampering)
- No Order created

### Scenario 18: Subscription Not Found for Webhook
- Webhook arrives with unknown subscriptionId/subscriptionRequestId
- Return 404 "Subscription not found"
- Logged as warning

### Scenario 19: Gateway Unavailable When User Tries to Subscribe
- `isBkashRecurringGatewayAvailable()` returns false
- Return 503 "bKash recurring payments are not configured"
- No DB records created

### Scenario 20: After 11:30 PM Bangladesh Time
- bKash same-day start restriction
- Return 400 with message about trying tomorrow
- No DB records created

### Scenario 21: Network Error During bKash API Call (createSubscription)
- API call fails/times out
- Order: status=`failed`
- RecurringSubscription: status=`payment_failed`
- Return 503 with error message

### Scenario 22: Callback Query to bKash Fails (Network/API Error)
- `querySubscriptionByRequestId` throws
- Caught by try/catch
- RecurringSubscription: status=`payment_failed`
- Redirect to `/payment/fail?error=subscription_query_failed`
- **The subscription might actually be active on bKash side**
- **Mitigation:** SUBSCRIPTION webhook or PAYMENT webhook will eventually correct the state

### Scenario 23: PRE_NOTIFICATION Webhook
- bKash sends 1-3 days before payment date (except daily)
- Logged for monitoring
- Return 200 "Pre-notification received"
- No DB changes (informational only)

---

## Webhook Handling

### Webhook Types & Handlers

| Type | Header | Handler | Description |
|------|--------|---------|-------------|
| PAYMENT | `Type: PAYMENT` | `handlePaymentWebhook()` | Auto-debit result (success/fail) |
| SUBSCRIPTION | `Type: SUBSCRIPTION` | `handleSubscriptionWebhook()` | Subscription status change |
| REFUND | `Type: REFUND` | `handleRefundWebhook()` | Refund result |
| CANCEL | `Type: CANCEL` | `handleCancelWebhook()` | Subscription cancelled |
| EXPIRY | `Type: EXPIRY` | `handleExpiryWebhook()` | Subscription expired |
| PRE_NOTIFICATION | `Type: PRE_NOTIFICATION` | (logged only) | Payment reminder |

### Signature Verification

```
Algorithm: HMAC-SHA256
Key: BKASH_RECURRING_WEBHOOK_SECRET (from env)
Data: Raw request body (JSON string)
Header: X-Signature or Signature
```

All webhooks are verified before processing. Failed verification returns 400.

### Webhook Endpoint

```
POST https://api.yourdomain.com/api/v1/public/recurring-bkash/webhook
```

This single URL handles ALL webhook types. The `Type` header determines routing.

---

## Deduplication & Idempotency

### First Payment (Callback + Possible Webhook Race)
- Callback processes first: Updates pending Order to `completed`
- PAYMENT webhook arrives later: Checks by trxId -> Found existing -> Skips

### First Payment (Webhook Arrives First, Callback Later)
- PAYMENT webhook: Finds pending Order, updates to `completed`
- Callback later: `markPaymentSuccess()` is idempotent (skips if already completed)

### Subsequent Auto-Charges (Only Webhook)
- PAYMENT webhook: Checks by trxId -> Not found -> Creates new Order
- If bKash sends duplicate: trxId already exists -> Skips

### Key: All paths check `transactionId` (trxId) for deduplication

---

## Gateway Configuration & Credentials

### Environment Variables Required

| Variable | Description | Example |
|----------|-------------|---------|
| `BKASH_RECURRING_API_KEY` | bKash RPP API key | `abc123...` |
| `BKASH_RECURRING_SERVICE_ID` | Service ID from bKash | `SVC-xxx` |
| `BKASH_RECURRING_WEBHOOK_SECRET` | Secret for HMAC-SHA256 webhook verification | `secret-xxx` |
| `BKASH_RECURRING_MERCHANT_SHORT_CODE` | Merchant short code | `MERCHANT` |
| `BKASH_RECURRING_IS_PRODUCTION` | `true` for production | `false` |
| `CDN_URL` | Backend base URL (for callback/webhook URLs) | `https://api.yourdomain.com` |
| `FRONTEND_URL` | Frontend base URL (for redirect after payment) | `https://yourdomain.com` |

### Site Settings (Admin Panel)

- `payment_methods_recurring`: Must include `bkash-recurring` in the array

### bKash API Endpoints

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://recurring.sandbox.bka.sh` |
| Production | `https://recurring.pay.bka.sh` |

---

## What You Must Provide to bKash

### Before Integration Testing (Sandbox)

1. **Webhook URL:** `https://api.yourdomain.com/api/v1/public/recurring-bkash/webhook`
2. **Redirect URL:** `https://api.yourdomain.com/api/v1/public/recurring-bkash/callback`
   - Note: Both are set per-request in the createSubscription API call, so you don't need to configure them upfront with bKash

### For Go-Live

1. **Complete all 7 integration milestones** (from bKash email):
   - Milestone 1: Subscription creation
   - Milestone 2: Subscription Query
   - Milestone 3: Payment Query
   - Milestone 4: Payment refund
   - Milestone 5: Subscription cancellation
   - Milestone 6: Verify webhook notifications are handled
   - Milestone 7: End-to-end testing

2. **Production credentials from bKash:**
   - Production API Key
   - Production Service ID
   - Webhook Secret
   - Merchant Short Code

3. **SSL certificate** on your domain (HTTPS required)

4. **Publicly accessible** webhook URL (no IP whitelisting needed - signature verification is used)

5. **bKash will configure in their system:**
   - Your merchant service ID
   - Subscription parameters (frequencies you want to support)
   - Your channel ID

### bKash Recurring Constraints

| Constraint | Value |
|-----------|-------|
| Minimum amount | 10 BDT |
| Maximum amount | 10,000,000 BDT |
| Maximum subscription duration | 2 years from start date |
| Start time restriction | Cannot start after 11:30 PM BD time (same day) |
| Supported frequencies | Daily, 7 days, 15 days, 30 days, 90 days, 180 days, Monthly (calendar), Yearly (calendar) |
| Our supported frequencies | Daily, Monthly (mapped to appropriate bKash values) |
| Card types | N/A - bKash wallet only |
| Pre-notification | 1-3 days before charge (except daily) |
| Retry on failure | 2 retries every 3 days (except daily - no retry) |
| Guest subscriptions | NOT supported (customer name + phone required) |
| Fund restriction | Only "regular" fund |

---

## Pre-Go-Live Checklist

### Configuration
- [ ] Production `BKASH_RECURRING_API_KEY` set in environment
- [ ] Production `BKASH_RECURRING_SERVICE_ID` set in environment
- [ ] Production `BKASH_RECURRING_WEBHOOK_SECRET` set in environment
- [ ] Production `BKASH_RECURRING_MERCHANT_SHORT_CODE` set
- [ ] `BKASH_RECURRING_IS_PRODUCTION=true` in production environment
- [ ] `CDN_URL` points to production backend (HTTPS)
- [ ] `FRONTEND_URL` points to production frontend
- [ ] Site setting `payment_methods_recurring` includes `bkash-recurring`
- [ ] SSL certificate valid on backend domain

### Functional Testing (Sandbox First)
- [ ] Successfully create a subscription (daily, monthly)
- [ ] Verify callback redirect works (success, cancel, fail)
- [ ] Verify SUBSCRIPTION webhook is received and processed
- [ ] Verify PAYMENT webhook for first payment
- [ ] Verify PAYMENT webhook for subsequent auto-charges
- [ ] Verify failed payment creates failed Order entry
- [ ] Verify retry payment (RE_SUCCEEDED_PAYMENT) works
- [ ] Verify cancel subscription from our system works
- [ ] Verify CANCEL webhook from bKash works
- [ ] Verify EXPIRY webhook works
- [ ] Verify REFUND webhook works
- [ ] Verify duplicate webhook is deduplicated (trxId check)
- [ ] Verify amount mismatch is rejected
- [ ] Verify signature verification rejects tampered webhooks
- [ ] Verify 11:30 PM restriction works
- [ ] Verify phone validation works
- [ ] Verify email notifications are sent

### Edge Cases to Test
- [ ] Close browser before reaching bKash page
- [ ] Close browser during bKash consent (after OTP)
- [ ] Close browser after consent but before callback redirect
- [ ] Multiple rapid subscription attempts by same customer
- [ ] Webhook arrives before callback
- [ ] Invalid subscriptionRequestId in callback
- [ ] Gateway unavailable (env vars missing)

### Monitoring
- [ ] Error logging captures all failure scenarios
- [ ] Webhook processing failures are logged with full payload
- [ ] Payment amounts are validated and mismatches logged
- [ ] Risk transactions are flagged in logs

---

## Known Limitations & Risks

### 1. No Cron Job for bKash Recurring Status Sync
The `processRecurringPayments()` cron job only syncs SSLCommerz subscriptions. There is NO equivalent for bKash. This means:
- If a webhook is missed, the DB status may be stale
- **Recommendation:** Add a cron job that periodically queries active bKash subscriptions via `querySubscriptionById()` to sync status

### 2. Orphaned Processing Records
RecurringSubscriptions stuck in `initiated` or `processing` state (user abandoned before consent) are never cleaned up. The `reconcilePendingPayments()` cron handles PaymentTransactions but NOT RecurringSubscriptions.
- **Recommendation:** Add cleanup logic for RecurringSubscriptions in `initiated`/`processing` older than 1 hour

### 3. No Pause/Resume for bKash
Unlike SSLCommerz, bKash RPP does not support pause/resume via API. Only cancellation is supported. The `extendSubscription()` method exists in the gateway but is not exposed as a route.

### 4. Subscription ID Storage During Creation
During creation, the `subscriptionId` field stores the `subscriptionRequestId` (UUID). It is later updated to the actual bKash numeric subscription ID on callback or webhook. The `findRecurringSubscription()` helper handles this dual-ID lookup.

### 5. Daily Frequency Has No Retry
Per bKash docs, daily subscriptions have NO retry on failure. If a daily charge fails, that cycle's amount is lost.

### 6. First Payment Special Case
For BASIC subscription type, the first payment happens via auto-debit on the start date (same day). The pending Order is updated (not a new one created) when the first PAYMENT webhook arrives. This is correctly handled with the `isFirstPayment` flag and pending order lookup.
