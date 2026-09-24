# SSLCommerz Recurring Payment - Complete Flow Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Database Models](#database-models)
3. [Complete API Flow](#complete-api-flow)
4. [The Schedule Parameter & Encryption](#the-schedule-parameter--encryption)
5. [Bill Query API (Our Endpoint, Called by SSLCommerz)](#bill-query-api)
6. [IPN Listener (Our Endpoint, Called by SSLCommerz)](#ipn-listener)
7. [Every Scenario & Edge Case](#every-scenario--edge-case)
8. [Deduplication & Idempotency](#deduplication--idempotency)
9. [Signature Verification](#signature-verification)
10. [Gateway Configuration & Credentials](#gateway-configuration--credentials)
11. [What You Must Configure in SSLCommerz Admin Panel](#what-you-must-configure-in-sslcommerz-admin-panel)
12. [Pre-Go-Live Checklist](#pre-go-live-checklist)
13. [Known Limitations & Risks](#known-limitations--risks)

---

## Architecture Overview

SSLCommerz recurring is fundamentally different from bKash recurring. SSLCommerz uses a **Bill Query + IPN** model, while bKash uses webhooks only.

```
Frontend (Next.js)                Backend (Fastify)                     SSLCommerz
==================               =================                     ==========

SubscriptionForm.tsx ----POST /public/recurring-ssl----->  Create:
  - customerName                                           - Customer (find/create)
  - customerEmail/Phone                                    - RecurringSubscription (status: initiated)
  - amount                                              - Order (status: pending)
  - frequency                                           - Build encrypted schedule param
  - planId                                              - Call SSLCommerz Session API
                                                        - PaymentTransaction (audit)
                                                        - Return GatewayPageURL
                                  
User <----------- redirect to SSLCommerz page ---------  GatewayPageURL
  - Visa/Mastercard only
  - Must login & save card (login_req=1)
  - First payment charged immediately
                                  
 [BROWSER CALLBACK]
    GET/POST /recurring-ssl/callback <--- SSLCommerz redirects user
                                  |       (with val_id, tran_id, status, etc.)
                                  v
                                Verify with SSLCommerz Validation API
                                Update Order (pending -> completed)
                                Update RecurringSubscription (-> active)
                                Store subscriptionId
                                Redirect to /payment/success
                                  
 [SERVER-TO-SERVER IPN]  (CONCURRENT with callback, or sole notification for auto-charges)
    POST /recurring-ssl/ipn <--- SSLCommerz sends IPN
                                  |
                                  v
                                Verify signature (MD5)
                                Verify with Validation API (val_id)
                                Validate amount
                                Deduplicate (by tran_id/bank_tran_id)
                                Create/Update Order
                                Update stats
                                  
 [SCHEDULED AUTO-CHARGES]  (SSLCommerz initiates for subsequent payments)
                                  
    POST /recurring-ssl/bill-query <--- SSLCommerz asks "should I charge?"
                                  |
                                  v
                                Find RecurringSubscription by subscription_id
                                Check status is active
                                Check end date not passed
                                Return: { status: "success", total_amount: "100.00" }
                                  
    POST /recurring-ssl/ipn <--- SSLCommerz notifies charge result
                                  |
                                  v
                                (Same IPN handler as above)
                                Create new Order record
                                Update stats
                                  
 [ADMIN MANAGEMENT]
    POST /recurring-subscriptions/:id/pause ----> gateway.disableSubscription()
    POST /recurring-subscriptions/:id/resume ---> gateway.enableSubscription()
    POST /recurring-subscriptions/:id/cancel ---> gateway.cancelSubscription()
    
 [CRON: processRecurringPayments()]
    Periodically queries SSLCommerz for subscription status
    Syncs local status with gateway status
```

### Key Files

| File | Purpose |
|------|---------|
| `backend/src/routes/api/v1/public/recurring-ssl/index.ts` | Public routes: create, callback, bill-query, IPN, status, pause/resume/cancel (~1500 lines) |
| `backend/src/plugins/app/payment-gateways/sslcommerz-recurring-gateway.ts` | SSLCommerz recurring API client (~736 lines) |
| `backend/src/utils/sslcommerz-encryption.ts` | AES-256-CBC encryption for schedule parameter (~97 lines) |
| `backend/src/plugins/app/cron-jobs.ts` | Cron: recurring subscription status sync (~575 lines) |
| `backend/src/plugins/app/payment-service.ts` | Shared payment transaction service |
| `backend/src/plugins/app/payment-gateways/gateway-manager.ts` | Gateway factory/availability |
| `backend/src/routes/api/v1/recurring-subscriptions/index.ts` | Admin recurring Order management (~337 lines) |
| `frontend/components/donate/SubscriptionForm.tsx` | Order form UI |
| `frontend/data/OrderConfig.ts` | Payment method configuration |

---

## Database Models

### RecurringSubscription
```
id                  Int       PK
customerId             Int       FK -> Customer
amount              Decimal   The recurring charge amount
frequency           String    daily | weekly | monthly | yearly
paymentMethod       String    "sslcommerz"
subscriptionId      String?   Unique. SSLCommerz subscription_id (from session response/callback/IPN)
refer               String?   QR Refer ID (from env SSLCOMMERZ_RECURRING_REFER)
acctNo              String?   Customer identifier (phone or email, used in schedule encryption & bill query)
dayOfMonth          Int?      1-30 for monthly/yearly schedules
weekDay             String?   sun-sat for weekly schedules
status              String    initiated -> processing -> active -> paused/deactivated/cancelled/payment_failed
startDate           DateTime
endDate             DateTime? Set on cancel/deactivate
nextBillingDate     DateTime? Calculated after each charge
lastChargedAt       DateTime? Last successful charge
planId              Int       FK -> SubscriptionPlan  
totalCharged        Decimal   Sum of all successful charges
successCount        Int       Count of successful payments
failedCount         Int       Count of failed payments
```

### Order (one per charge cycle)
```
id                    Int       PK
customerId               Int       FK -> Customer
RecurringSubscriptionId   Int?      FK -> RecurringSubscription
amount                Decimal
paymentMethod         String    "sslcommerz"
paymentMethodDetail   String?   SSLCommerz card_type (e.g., "VISA-Dutch Bangla", "Mastercard")
status                String    pending -> completed/failed/cancelled/refunded
transactionId         String?   SSLCommerz bank_tran_id or tran_id
invoiceNumber         String?   Our generated invoice number
isAnonymous           Boolean   Always false for recurring  
userName              String?   Customer name
```

### PaymentTransaction (audit trail)
```
id                    Int       PK
orderId            Int       FK -> Order
gateway               String    "sslcommerz"
gatewayTransactionId  String?   tran_id (our transaction ID sent to SSLCommerz, format: TXN{id}T{timestamp})
gatewayPaymentId      String?   SSLCommerz session key or val_id
bankTransactionId     String?   SSLCommerz bank_tran_id
status                String    initiated/pending/success/failed/cancelled
invoiceNumber         String?
createResponse        JSON?     Session API response
executeResponse       JSON?     Callback/IPN data
errorResponse         JSON?     Error details
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
lastPurchasedAt   DateTime?
```

---

## Complete API Flow

### Step 1: User Submits Recurring Order Form

**Frontend (SubscriptionForm.tsx):**
- User selects "Recurring" Order type
- Chooses frequency: Daily or Monthly
- Enters amount (preset chips or custom)
- Enters name + email or phone (required)
- Selects "Visa/Mastercard" as payment method (maps to sslcommerz)
- Anonymous option is DISABLED for recurring
- Confirm modal shown

**API Call:** `POST /api/v1/public/recurring-ssl`
```json
{
  "customerName": "Rahim Ahmed",
  "customerEmail": "rahim@example.com",
  "customerPhone": "01712345678",
  "amount": 500,
  "frequency": "monthly",
  "dayOfMonth": 15,           // optional, defaults to today's date
  "weekDay": "sun",           // optional, for weekly only
  "planId": 1,
  "message": "Monthly Order"
}
```

### Step 2: Backend Validation & Record Creation

**Validation checks (in order):**
1. Recurring SSLCommerz enabled in site settings (`payment_methods_recurring` includes `sslcommerz`)
2. Recurring gateway available (`SSLCOMMERZ_SALT_KEY` and `SSLCOMMERZ_RECURRING_REFER` env vars set)
3. At least email or phone provided
4. Phone format validation (BD format if provided)
5. Subscription plan exists and is active

**Database creates (in order):**
1. **Customer:** Find by email OR phone, create if not found
2. **RecurringSubscription:** status=`initiated`, paymentMethod=`sslcommerz`, dayOfMonth/weekDay set
3. **Order:** status=`pending`, linked to RecurringSubscription

### Step 3: Build Encrypted Schedule Parameter

This is SSLCommerz's unique requirement. The schedule is a JSON object encrypted with AES-256-CBC.

**Plain JSON (before encryption):**
```json
{
  "refer": "5B90BA91AA3F2",        // From SSLCOMMERZ_RECURRING_REFER env var
  "acct_no": "01712345678",        // Customer identifier (phone/email)
  "type": "monthly",               // Frequency
  "dayofmonth": 15                  // Day to charge (for monthly)
}
```

**Encryption process (matches SSLCommerz PHP library):**
1. Generate random 16-byte IV (AES-256-CBC standard)
2. Encrypt JSON string with AES-256-CBC using SALT_KEY
3. Combine: `base64(iv + '|||' + ciphertext)`

**Key handling:** PHP's openssl_encrypt pads keys < 32 bytes with NUL bytes, truncates if >= 32. Our implementation matches this exactly.

### Step 4: SSLCommerz Session API

**API Call:** `POST {baseUrl}/gwprocess/v4/api.php`

**Request body (form URL-encoded):**
```
store_id=<STORE_ID>
store_passwd=<STORE_PASSWORD>
total_amount=500.00
currency=BDT
tran_id=DON100T1710000000          // Our generated transaction ID
success_url=https://api.example.com/api/v1/public/recurring-ssl/callback
fail_url=https://api.example.com/api/v1/public/recurring-ssl/callback
cancel_url=https://api.example.com/api/v1/public/recurring-ssl/callback
ipn_url=https://api.example.com/api/v1/public/recurring-ssl/ipn
cus_name=Rahim Ahmed
cus_email=rahim@example.com
cus_phone=01712345678
cus_add1=Dhaka
cus_city=Dhaka
cus_country=Bangladesh
shipping_method=NO
product_name=Recurring Order
product_category=Order
product_profile=general
schedule=<ENCRYPTED_SCHEDULE>       // AES-256-CBC encrypted JSON
multi_card_name=visacard,mastercard // Only Visa & Mastercard support recurring
login_req=1                         // REQUIRED - forces card save for recurring
value_a=100                         // orderId
value_b=recurring                   // type marker
value_c=42                          // RecurringSubscriptionId
value_d=INV-xxx                     // invoiceNumber
```

**Note:** `success_url`, `fail_url`, `cancel_url` ALL point to the same callback URL. SSLCommerz includes status in the redirect data, and we route based on that.

**Response from SSLCommerz:**
```json
{
  "status": "SUCCESS",
  "sessionkey": "ABC123...",
  "GatewayPageURL": "https://sandbox.sslcommerz.com/gwprocess/v4/gw.php?Q=PAY&SESSKEY=ABC123...",
  "subscription_status": "initiated",
  "subscription_id": "SUVB0000011110003"
}
```

### Step 5: Store & Create Audit Records

- If `subscription_id` returned: Update RecurringSubscription with subscriptionId, status=`processing`
- Create PaymentTransaction with session API response stored in `createResponse`

**Note:** `subscription_id` may NOT be returned in Session API response if there's an issue with schedule params. In that case, the transaction proceeds as normal (one-time), and the subscription_id comes later via IPN. Our code handles both cases.

### Step 6: Redirect User to SSLCommerz

**Response to frontend:**
```json
{
  "success": true,
  "data": {
    "RecurringSubscriptionId": 42,
    "orderId": 100,
    "paymentTransactionId": 5,
    "transactionId": "DON100T1710000000",
    "amount": 500,
    "frequency": "monthly",
    "status": "processing",
    "paymentUrl": "https://sandbox.sslcommerz.com/gwprocess/v4/gw.php?Q=PAY&SESSKEY=ABC123...",
    "message": "Redirecting to payment gateway. Only Visa/Mastercard cards supported for recurring payments."
  }
}
```

Frontend redirects user to `paymentUrl`.

### Step 7: User Completes Payment on SSLCommerz

On SSLCommerz page:
1. User sees ONLY Visa and Mastercard options (multi_card_name filter)
2. User MUST log in (login_req=1) - this saves the card for future recurring charges
3. User enters card details
4. 3D Secure verification (if required by issuer)
5. First payment charged immediately

### Step 8: Callback (Browser Redirect)

SSLCommerz redirects user to our callback with payment data (GET or POST).

**Endpoint:** `GET/POST /api/v1/public/recurring-ssl/callback`

**Data includes:** `tran_id`, `val_id`, `status` (VALID/FAILED/CANCELLED), `amount`, `card_type`, `bank_tran_id`, `subscription_id`, etc.

**Backend processing:**
1. Parse callback data (GET query or POST body)
2. Find PaymentTransaction by:
   - `tran_id` (our TXN{id}T{timestamp} format) -- primary lookup
   - `value_a` (orderId) -- fallback
   - Legacy `REC{id}T` format -- fallback for old transactions
3. Get the linked Order and RecurringSubscription
4. **If status=SUCCESS:**
   - Verify with SSLCommerz Validation API using `val_id`
   - Call `queryPayment({ transactionId: val_id })` -> hits `/validator/api/validationserverAPI.php`
   - If validation passes: `markPaymentSuccess()` (idempotent)
   - Update RecurringSubscription: status=`active`, store subscriptionId
   - Redirect to `/payment/success?tx=...&amount=...&invoice=...&recurring=true`
5. **If status=CANCELLED:**
   - Mark payment cancelled, Order cancelled
   - RecurringSubscription: status=`cancelled`
   - Redirect to `/payment/cancel`
6. **If status=FAILED:**
   - Mark payment failed, Order failed
   - RecurringSubscription: status=`payment_failed`, failedCount++
   - Redirect to `/payment/fail`

### Step 9: IPN (Server-to-Server, Concurrent with Callback)

SSLCommerz sends IPN to our backend independently of the browser callback.

**Endpoint:** `POST /api/v1/public/recurring-ssl/ipn`

**IPN data includes:** `tran_id`, `val_id`, `status`, `amount`, `card_type`, `bank_tran_id`, `subscription_id`, `verify_sign`, `verify_key`, `risk_level`, `risk_title`, etc.

**Processing:**
1. **Verify signature** (MD5 hash using verify_key + store_passwd)
2. Find RecurringSubscription by:
   - `subscription_id` -- primary
   - Legacy `REC{id}T` from tran_id -- fallback
   - `value_a` (orderId) -- fallback
   - `acctNo` -- last resort
3. **Save subscription_id if missing** (IPN is most reliable source per SSL doc)
4. **If status=VALID/VALIDATED:**
   - Verify with SSLCommerz Validation API (val_id)
   - Validate amount matches RecurringSubscription.amount (tolerance: 0.01 BDT)
   - Log risk level if >= 1 (HIGH RISK)
   - **Deduplication:** Check for existing Order with same tran_id or bank_tran_id
   - **First payment:** Find pending Order -> update to `completed`
   - **Subsequent charges:** Create new Order record
   - Create PaymentTransaction (audit)
   - Update RecurringSubscription stats (totalCharged, successCount, lastChargedAt, nextBillingDate)
   - Update Customer stats (totalSpent, orderCount, lastPurchasedAt)
   - Send email notification (async)
5. **If status=FAILED:**
   - Create failed Order entry (every charge attempt is visible)
   - Create failed PaymentTransaction
   - Update RecurringSubscription: failedCount++, status=`payment_failed`

### Step 10: Subsequent Scheduled Charges

On each billing date, SSLCommerz initiates the charge cycle:

**10a. Bill Query (SSLCommerz calls our endpoint)**

**Endpoint:** `POST /api/v1/public/recurring-ssl/bill-query`

SSLCommerz sends:
```
refer=5B90BA91AA3F2
subscription_id=SUVB0000011110003
acct_no=01712345678
tran_id=<SSL-generated>
amount=500.00
currency=BDT
```

Our response:
```json
{
  "status": "success",
  "failedreason": "Information okay",
  "tran_id": "<echoed>",
  "refer": "5B90BA91AA3F2",
  "subscription_id": "SUVB0000011110003",
  "acct_no": "01712345678",
  "total_amount": "500.00",
  "currency": "BDT",
  "ipn_url": "https://api.example.com/api/v1/public/recurring-ssl/ipn",
  "cus_name": "Rahim Ahmed",
  "cus_email": "rahim@example.com",
  "cus_phone": "01712345678"
}
```

**Bill Query checks:**
- RecurringSubscription found by subscription_id or acctNo
- Status is `active`
- End date not passed (auto-deactivates if expired)
- If any check fails: returns `status: "failed"` with reason, SSLCommerz skips the charge

**10b. SSLCommerz Charges the Card**

If bill-query returns success, SSLCommerz auto-charges the saved card.

**10c. IPN Notification**

SSLCommerz sends IPN with charge result -> Same IPN handler as Step 9.
Creates new Order record for this billing cycle.

### Step 11: Subscription Management

**Admin-only endpoints (from admin panel):**

| Action | Endpoint | SSLCommerz API | Effect |
|--------|----------|---------------|--------|
| Pause | `POST /recurring-subscriptions/:id/pause` | `disableSubscription` | Temporary - can resume later |
| Resume | `POST /recurring-subscriptions/:id/resume` | `enableSubscription` | Re-enable from paused state |
| Cancel | `POST /recurring-subscriptions/:id/cancel` | `cancelSubscription` | Permanent - cannot undo |

**Public endpoints (OTP-authenticated):**

| Action | Endpoint | Effect |
|--------|----------|--------|
| Pause | `POST /public/recurring-ssl/:subscriptionId/pause` | Requires `fastify.authenticate` |
| Resume | `POST /public/recurring-ssl/:subscriptionId/resume` | Requires `fastify.authenticate` |
| Cancel | `POST /public/recurring-ssl/:subscriptionId/cancel` | Requires `fastify.authenticate` |

### Step 12: Cron Job - Status Sync

`processRecurringPayments()` runs periodically:
1. Queries all `active` RecurringSubscriptions with paymentMethod=`sslcommerz`
2. For each: calls `gateway.getSubscriptionStatus(subscriptionId)`
3. Maps SSLCommerz status to internal status
4. Updates if different

---

## The Schedule Parameter & Encryption

### Encryption Implementation

File: `backend/src/utils/sslcommerz-encryption.ts`

```
Input:  JSON string (schedule)
Key:    SSLCOMMERZ_SALT_KEY (from env)
Algorithm: AES-256-CBC
Output: base64(iv + '|||' + ciphertext)
```

**Key handling matches PHP exactly:**
- Key < 32 bytes: padded with NUL bytes
- Key >= 32 bytes: truncated to first 32
- IV: 16 random bytes (openssl_cipher_iv_length for aes-256-cbc)

### Schedule JSON Examples

**Daily:**
```json
{"refer":"5B90BA91AA3F2","acct_no":"01712345678","type":"daily"}
```

**Weekly (Sunday):**
```json
{"refer":"5B90BA91AA3F2","acct_no":"01712345678","type":"weekly","week":"sun"}
```

**Monthly (15th):**
```json
{"refer":"5B90BA91AA3F2","acct_no":"01712345678","type":"monthly","dayofmonth":15}
```

**Yearly (August 24th):**
```json
{"refer":"5B90BA91AA3F2","acct_no":"01712345678","type":"yearly","month":"8","dayofmonth":"24"}
```

---

## Bill Query API

This is YOUR endpoint that SSLCommerz calls before each auto-charge.

**Endpoint:** `POST /api/v1/public/recurring-ssl/bill-query`

**Purpose:** SSLCommerz asks "Should I charge this subscriber? How much?"

### Request (from SSLCommerz)

```
refer=<QR Refer ID>
subscription_id=<Subscription ID>
acct_no=<Account Number>
tran_id=<SSL-generated transaction ID>
amount=500.00
currency=BDT
cus_name=Rahim Ahmed
cus_email=rahim@example.com
cus_phone=01712345678
```

### Response (from our system)

**Success (charge should proceed):**
```json
{
  "status": "success",
  "failedreason": "Information okay",
  "tran_id": "<echoed from request>",
  "refer": "<Refer ID>",
  "subscription_id": "<Subscription ID>",
  "acct_no": "<Account Number>",
  "total_amount": "500.00",
  "currency": "BDT",
  "ipn_url": "https://api.example.com/api/v1/public/recurring-ssl/ipn",
  "cus_name": "Rahim Ahmed",
  "cus_email": "rahim@example.com",
  "cus_phone": "01712345678"
}
```

**Failure (skip this charge):**
```json
{
  "status": "failed",
  "failedreason": "Subscription is paused",
  "tran_id": "<echoed>",
  "refer": "<Refer ID>",
  "subscription_id": "<Subscription ID>",
  "acct_no": "<Account Number>",
  "total_amount": "0.00",
  "currency": "BDT"
}
```

### Bill Query Decision Logic

1. Find RecurringSubscription by `subscription_id` or `acctNo`
2. Not found? Return `failed` ("Subscription not found")
3. Status != `active`? Return `failed` ("Subscription is {status}")
4. End date passed? Auto-deactivate, return `failed` ("Subscription has ended")
5. All good? Return `success` with amount from RecurringSubscription.amount

**Dynamic amount:** The bill-query response `total_amount` determines the charge amount. If you want to change the amount mid-subscription, update RecurringSubscription.amount in the DB and the next bill-query will return the new amount.

---

## IPN Listener

This is YOUR endpoint that SSLCommerz calls after each transaction (both first payment and subsequent auto-charges).

**Endpoint:** `POST /api/v1/public/recurring-ssl/ipn`

### IPN Processing Flow

```
IPN received
  |
  v
Verify MD5 signature (verify_sign + verify_key)
  |
  v  (fail -> 400 "Invalid signature")
  |
Find RecurringSubscription
  (by subscription_id -> tran_id REC format -> value_a -> acctNo)
  |
  v  (not found -> 404)
  |
Save subscription_id if missing
  |
  v
Status = VALID/VALIDATED?
  |--- YES ---> Verify with Validation API (val_id)
  |               |--- fail ---> 400 "Payment validation failed" (continue if API down)
  |               |--- pass --->
  |             Validate amount matches (tolerance 0.01)
  |               |--- mismatch ---> 400 "Amount mismatch" (logged as potential tampering)
  |             Log risk_level if >= 1
  |             Deduplicate (check tran_id AND bank_tran_id)
  |               |--- exists ---> 200 "Transaction already processed" (skip)
  |             Check for pending Order (first payment path)
  |               |--- found ---> Update pending to completed
  |               |--- not found ---> Create new Order (subsequent charge)
  |             Create PaymentTransaction (audit)
  |             Update RecurringSubscription stats
  |             Update Customer stats
  |             Send email (async, non-blocking)
  |             Return 200
  |
  |--- NO (FAILED) ---> Create failed Order entry
  |                       Create failed PaymentTransaction
  |                       Increment failedCount
  |                       Return 200
  |
  |--- UNKNOWN ---> Log and return 200
```

---

## Every Scenario & Edge Case

### Scenario 1: Happy Path - First Payment + Recurring
1. User submits -> DB records created -> Redirect to SSLCommerz
2. User pays with Visa/Mastercard, saves card
3. **Callback fires:** Validates with SSLCommerz, marks completed, sets active
4. **IPN fires (concurrent):** Deduplicates (Order already completed), skips
5. On next billing date: Bill-query -> IPN -> New Order
6. Repeats every cycle

### Scenario 2: User Closes Browser BEFORE SSLCommerz Page
- RecurringSubscription: status=`processing`
- Order: status=`pending`
- PaymentTransaction: status=`initiated`
- **No callback, no IPN** will come
- **Mitigation:** `reconcilePendingPayments()` cron job expires stale PaymentTransactions (marks as failed after timeout). However, RecurringSubscription stays in `processing`.
- **RECOMMENDATION:** Add cleanup for RecurringSubscriptions stuck in `initiated`/`processing` older than 1 hour

### Scenario 3: User Reaches SSLCommerz Page, Waits Indefinitely
- SSLCommerz session has a built-in timeout (typically 20 minutes)
- After timeout, session expires on SSLCommerz side
- No callback or IPN fires
- Same orphaned state as Scenario 2
- **Mitigation:** Same cron job cleanup

### Scenario 4: User Completes Payment, Browser Tab Closed Before Callback
- Payment is successful on SSLCommerz side
- **Callback never reaches our server**
- **IPN DOES fire (server-to-server, independent of browser)**
- IPN handler: Finds pending Order, updates to completed, activates subscription
- **Impact:** User doesnt see success page, but subscription IS active
- **This is WHY IPN exists** - it's the safety net for lost callbacks

### Scenario 5: User Cancels on SSLCommerz Page
- SSLCommerz redirects to callback with status=CANCELLED
- Callback: marks Order cancelled, RecurringSubscription cancelled
- No subscription created on SSLCommerz side

### Scenario 6: Card Declined / Payment Fails
- SSLCommerz redirects to callback with status=FAILED
- Callback: marks Order/payment failed
- RecurringSubscription: status=`payment_failed`, failedCount++
- No subscription activated

### Scenario 7: Callback Arrives Before IPN
- Callback processes first: validates, marks completed, activates subscription
- IPN arrives later: finds Order with matching tran_id, skips ("Transaction already processed")
- **Both paths are safe, no double-counting**

### Scenario 8: IPN Arrives Before Callback
- IPN processes first: validates, updates pending Order to completed, activates
- Callback arrives later: `markPaymentSuccess()` is idempotent (checks if already completed)
- **Both paths are safe, no double-counting**

### Scenario 9: Bill Query - Subscription Active, Charge Proceeds
- SSLCommerz calls bill-query -> returns `status: "success"` with amount
- SSLCommerz charges saved card
- SSLCommerz sends IPN with result
- IPN creates new Order + PaymentTransaction
- Stats updated

### Scenario 10: Bill Query - Subscription Paused
- SSLCommerz calls bill-query
- RecurringSubscription.status = `paused`
- Returns `status: "failed"`, `failedreason: "Subscription is paused"`
- SSLCommerz skips the charge for this cycle

### Scenario 11: Bill Query - Subscription Expired
- SSLCommerz calls bill-query
- EndDate has passed
- Auto-deactivates RecurringSubscription
- Returns `status: "failed"`, `failedreason: "Subscription has ended"`

### Scenario 12: Bill Query - Subscription Not Found
- SSLCommerz calls bill-query with unknown subscription_id
- Returns `status: "failed"`, `failedreason: "Subscription not found"`
- SSLCommerz skips charge

### Scenario 13: Scheduled Charge Succeeds (IPN with VALID)
- IPN arrives for auto-charge
- Amount validated against RecurringSubscription.amount
- Deduplication check (by tran_id, bank_tran_id)
- New Order created (NOT update - these are new charges)
- Stats incremented
- Email sent

### Scenario 14: Scheduled Charge Fails (IPN with FAILED)
- IPN arrives with status=FAILED
- Failed Order record created (for full charge history visibility)
- Failed PaymentTransaction created
- RecurringSubscription: failedCount++, status=`payment_failed`
- **SSLCommerz does NOT retry** (unlike bKash which retries 2 times)
- The failed cycle is lost
- **Bill-query for next cycle still runs** - subscription continues

### Scenario 15: Admin Pauses Subscription
- `POST /recurring-subscriptions/:id/pause`
- Backend calls `gateway.disableSubscription(subscriptionId)`
- SSLCommerz API response confirms
- Local status updated to `paused`
- Next bill-query returns `failed` - charge skipped

### Scenario 16: Admin Resumes Subscription
- `POST /recurring-subscriptions/:id/resume`
- Backend calls `gateway.enableSubscription(subscriptionId)`
- Local status updated to `active`
- Next bill-query returns `success` - charge proceeds

### Scenario 17: Admin Cancels Subscription (Permanent)
- `POST /recurring-subscriptions/:id/cancel`
- Backend calls `gateway.cancelSubscription(subscriptionId)`
- Local status: `cancelled`, endDate=now
- **Cannot be undone** - SSLCommerz permanently removes the schedule

### Scenario 18: IPN Signature Verification Fails
- MD5 hash mismatch
- Return 400 "Invalid signature"
- No DB changes
- Logged as warning with computed vs received hash

### Scenario 19: IPN Amount Mismatch
- IPN amount differs from RecurringSubscription.amount by > 0.01 BDT
- Return 400 "Amount mismatch"
- Logged as error ("possible tampering")
- No Order created

### Scenario 20: Duplicate IPN (Same tran_id)
- First IPN creates Order with transactionId=tran_id
- Duplicate IPN: finds existing Order with same tran_id or bank_tran_id
- Skips, returns 200 "Transaction already processed"
- Stats NOT double-incremented

### Scenario 21: High-Risk Transaction (risk_level >= 1)
- Logged as WARNING with risk_level and risk_title
- Payment still processed (per SSLCommerz recommendation: proceed but flag for review)
- Admin should review flagged transactions

### Scenario 22: subscription_id Missing from Session Response
Per SSLCommerz docs: "If this ID is not generated, there is an issue with the request parameters of the schedule. However, the transaction will still be processed successfully as a normal transaction."
- Our code handles this: subscription_id update is conditional
- The IPN for the first payment may include subscription_id
- If it does, we save it then (Section "SAVE subscription_id IF MISSING" in IPN handler)

### Scenario 23: Gateway Unavailable During Creation
- `isRecurringGatewayAvailable()` returns false (env vars missing)
- Return 503 "Recurring payments are not configured"
- No DB records created

### Scenario 24: SSLCommerz Session API Fails
- API returns status != SUCCESS or no GatewayPageURL
- Order: status=`failed`
- RecurringSubscription: status=`payment_failed`
- Return 503 with gateway error message

### Scenario 25: SSLCommerz Validation API Down During Callback
- `queryPayment()` throws
- Caught, payment marked failed
- RecurringSubscription: status=`payment_failed`
- Redirect to fail page
- **But:** IPN arrives independently and may succeed separately
- IPN validation uses val_id; if SSLCommerz validation API is also down from IPN, the code **continues processing** anyway (logged as error, but doesn't block)

### Scenario 26: Concurrent Bill-Query and IPN for Same Cycle
- Not an issue: bill-query happens BEFORE the charge, IPN happens AFTER
- They are sequential from SSLCommerz side, never concurrent

### Scenario 27: Cron Sync Detects Status Mismatch
- `processRecurringPayments()` queries SSLCommerz for active subscriptions
- If SSLCommerz shows `cancelled` but our DB shows `active`
- Updates our DB to match (maps SSLCommerz status to internal)

---

## Deduplication & Idempotency

### First Payment: Dual Path (Callback + IPN)

SSLCommerz sends BOTH a browser callback AND a server-to-server IPN for the first payment.

**Deduplication strategy:**
1. **Callback path:** Uses `markPaymentSuccess()` which is idempotent (checks if Order already completed)
2. **IPN path:** Checks for existing Order with same `tran_id` or `bank_tran_id`, or finds pending Order
3. **Race condition safe:** Either path can win, the second is deduplicated

### Subsequent Auto-Charges: IPN Only

Only IPN fires for auto-charges (no browser involved).

**Deduplication:** Each IPN checked against existing Orders by `tran_id` AND `bank_tran_id`.

### Transaction ID Lookup Chain

When finding PaymentTransaction in callback:
1. By `tran_id` (our TXN{id}T{timestamp} format)
2. By `value_a` (orderId)
3. By legacy `REC{id}T` format

When finding RecurringSubscription in IPN:
1. By `subscription_id`
2. By legacy `REC{id}T` from `tran_id`
3. By `value_a` (orderId)
4. By `acctNo`

---

## Signature Verification

### IPN Signature Algorithm (MD5)

Matches the official SSLCommerz PHP library:

1. Read `verify_key` (comma-separated list of parameter names)
2. For each key in `verify_key`: collect `key=value` from IPN data
3. Add `store_passwd=MD5(STORE_PASSWORD)` to the collection
4. Sort all keys alphabetically (PHP ksort)
5. Build hash string: all `key=value` pairs joined by `&`
6. MD5 hash the result
7. Compare with `verify_sign` from IPN data

```
verify_key = "amount,bank_tran_id,card_brand,..."
store_passwd_hash = MD5("your_store_password")

sorted_data = sort_alphabetically({ amount: "500", bank_tran_id: "xxx", ..., store_passwd: store_passwd_hash })
hash_string = "amount=500&bank_tran_id=xxx&...&store_passwd=MD5HASH"
computed_sign = MD5(hash_string)

valid = (computed_sign === verify_sign)
```

---

## Gateway Configuration & Credentials

### Environment Variables Required

| Variable | Description | Sandbox Example |
|----------|-------------|-----------------|
| `SSLCOMMERZ_STORE_ID` | Store ID | `testbox` (your sandbox store) |
| `SSLCOMMERZ_STORE_PASSWORD` | Store password | `qwerty` (your sandbox password) |
| `SSLCOMMERZ_SALT_KEY` | SALT Key for schedule encryption | `898c8b19e6cef9fe954b7a557458715e` (sandbox) |
| `SSLCOMMERZ_RECURRING_REFER` | QR Refer ID from SSLCommerz Admin Panel | `5B90BA91AA3F2` (sandbox) |
| `SSLCOMMERZ_IS_PRODUCTION` | `true` for production | `false` |
| `CDN_URL` | Backend base URL | `https://api.yourdomain.com` |
| `FRONTEND_URL` | Frontend base URL | `https://yourdomain.com` |

### API Endpoints

| Environment | Session API | Subscription Management | Validation |
|-------------|-------------|------------------------|------------|
| Sandbox | `https://sandbox.sslcommerz.com/gwprocess/v4/api.php` | `https://sandbox.sslcommerz.com/validator/api/v4/` | `https://sandbox.sslcommerz.com/validator/api/validationserverAPI.php` |
| Production | `https://securepay.sslcommerz.com/gwprocess/v4/api.php` | `https://securepay.sslcommerz.com/validator/api/v4/` | `https://securepay.sslcommerz.com/validator/api/validationserverAPI.php` |

### Site Settings (Admin Panel)

- `payment_methods_recurring`: Must include `sslcommerz` in the array

---

## What You Must Configure in SSLCommerz Admin Panel

### Step 1: Create Recurring QR Reference

1. **Login to SSLCommerz Merchant Panel:** https://merchant.sslcommerz.com/
2. Navigate to: **My Stores > Manage QR**
3. Click **"Add New QR"** (red button)
4. Configure:

| Field | Value |
|-------|-------|
| **Store Name** | (auto-displayed) |
| **Store ID** | (auto-displayed) |
| **Name of the Payment** | "Recurring Subscription" |
| **Alias of Transaction Reference** | "Subscription ID" |
| **Pay Button** | "Subscribe Now" |
| **Amount (BDT)** | Leave empty (dynamic amounts) |
| **QR Type** | **Recurring** |
| **Amount Type** | **Dynamic** (uses Bill Query API) |
| **Bill Query API** | `https://api.yourdomain.com/api/v1/public/recurring-ssl/bill-query` |
| **IPN URL** | `https://api.yourdomain.com/api/v1/public/recurring-ssl/ipn` |

5. Save. This generates the **Refer ID** (set as `SSLCOMMERZ_RECURRING_REFER` env var)

### Step 2: Get SALT Key

- The SALT Key is displayed when creating the QR reference, or contact SSLCommerz support
- Sandbox SALT Key: `898c8b19e6cef9fe954b7a557458715e`
- Production SALT Key: provided by SSLCommerz

### Step 3: Verify URLs Are Accessible

SSLCommerz needs to reach:
1. **Bill Query URL:** `https://api.yourdomain.com/api/v1/public/recurring-ssl/bill-query`
   - Must return JSON with `status`, `failedreason`, `total_amount`, etc.
2. **IPN URL:** `https://api.yourdomain.com/api/v1/public/recurring-ssl/ipn`
   - Must accept POST and return 200

### Step 4: Test with Sandbox Credentials

SSLCommerz sandbox credentials:
- **Refer:** `5B90BA91AA3F2`
- **SALT Key:** `898c8b19e6cef9fe954b7a557458715e`
- **Store ID/Password:** Register at https://developer.sslcommerz.com/registration/

Test card numbers (sandbox):
- **VISA:** 4111111111111111 (Exp: any future, CVV: 123)
- **Mastercard:** 5500000000000004

---

## Pre-Go-Live Checklist

### SSLCommerz Admin Panel Configuration
- [ ] Recurring QR Reference created in **production** merchant panel
- [ ] QR Type = **Recurring**, Amount Type = **Dynamic**
- [ ] Bill Query API URL set to production backend URL
- [ ] IPN URL set to production backend URL
- [ ] Production SALT Key obtained
- [ ] Production Refer ID noted

### Environment Variables
- [ ] `SSLCOMMERZ_STORE_ID` = production store ID
- [ ] `SSLCOMMERZ_STORE_PASSWORD` = production store password
- [ ] `SSLCOMMERZ_SALT_KEY` = production SALT Key
- [ ] `SSLCOMMERZ_RECURRING_REFER` = production Refer ID
- [ ] `SSLCOMMERZ_IS_PRODUCTION=true`
- [ ] `CDN_URL` = production backend HTTPS URL
- [ ] `FRONTEND_URL` = production frontend URL
- [ ] Site setting `payment_methods_recurring` includes `sslcommerz`

### Functional Testing (Sandbox)
- [ ] Create recurring subscription (daily, monthly)
- [ ] Verify SSLCommerz page shows only Visa/Mastercard
- [ ] Verify login_req forces card save
- [ ] First payment completes (callback + IPN both fire)
- [ ] Callback validates with SSLCommerz API (val_id verification)
- [ ] IPN signature verification works
- [ ] RecurringSubscription status transitions: initiated -> processing -> active
- [ ] Subscription ID stored in DB
- [ ] Bill-query endpoint returns correct amount
- [ ] Bill-query returns "failed" for paused subscription
- [ ] Bill-query returns "failed" for cancelled subscription
- [ ] IPN for auto-charge creates new Order
- [ ] Failed IPN creates failed Order entry
- [ ] Duplicate IPN is deduplicated
- [ ] Amount mismatch in IPN is rejected
- [ ] Pause subscription works (admin)
- [ ] Resume subscription works (admin)
- [ ] Cancel subscription works (admin, permanent)
- [ ] Cron syncs subscription status

### Edge Cases to Test
- [ ] Close browser before SSLCommerz page
- [ ] Close browser during payment (after card entry)
- [ ] Close browser after payment but before callback
- [ ] IPN arrives before callback (race condition)
- [ ] Callback arrives but IPN never does
- [ ] Gateway unavailable (env vars missing)
- [ ] Invalid SALT Key (encryption should fail gracefully)
- [ ] Bill-query with unknown subscription_id
- [ ] Subscription with end date in the past

### Security
- [ ] IPN signature verification rejects tampered data
- [ ] Amount validation catches mismatches
- [ ] High-risk transactions are logged
- [ ] Store password not exposed in logs (redacted in logApiCall)
- [ ] HTTPS on all endpoints
- [ ] Sensitive env vars not in version control

### Monitoring
- [ ] All IPN processing logged (success and failure)
- [ ] Bill-query calls logged
- [ ] Signature failures logged with details
- [ ] Amount mismatches logged as errors
- [ ] Gateway API errors logged with response payloads
- [ ] Cron job sync results logged

---

## Known Limitations & Risks

### 1. No SSLCommerz Retry on Failed Auto-Charge
Unlike bKash (which retries 2 times), SSLCommerz does NOT retry failed auto-charges. If a scheduled charge fails (expired card, insufficient funds), that cycle's payment is lost. SSLCommerz continues with the next scheduled date.
- **Impact:** Customer may not realize a charge failed
- **Recommendation:** Send notification email on failed charges (currently implemented in IPN handler)

### 2. Orphaned Records for Abandoned Flows
RecurringSubscriptions stuck in `initiated` or `processing` are not cleaned up by existing cron jobs. The `reconcilePendingPayments()` handles PaymentTransactions but not RecurringSubscriptions.
- **Recommendation:** Add cleanup for RecurringSubscriptions stuck > 1 hour in initiated/processing

### 3. subscription_id May Be Missing
If schedule parameter has issues, SSLCommerz processes the transaction as a one-time payment without creating a subscription. The `subscription_id` is not returned. Our code handles this by:
- Checking for subscription_id in IPN and saving if present
- Multiple fallback lookups (tran_id, value_a, acctNo)
- **Risk:** If subscription_id is never provided, recurring charges wont happen (its a one-time payment)
- **Recommendation:** Alert admin if subscription_id is missing after first successful payment

### 4. Dynamic Amount Changes
The bill-query response determines the charge amount. If you change RecurringSubscription.amount in the DB, the next charge uses the new amount. However:
- Old Order records still show the old amount
- Amount validation in IPN compares against current RecurringSubscription.amount
- **Risk:** If IPN arrives with old amount but amount was just changed, it may fail validation

### 5. Encryption Compatibility
The AES-256-CBC encryption must match PHP's `openssl_encrypt` exactly. Our implementation handles:
- Key padding (NUL bytes for short keys, truncation for long keys)
- IV generation (16 random bytes)
- Output format: `base64(iv + '|||' + ciphertext)`

This has been tested against the PHP example in SSLCommerz documentation. If SSLCommerz changes their encryption library, this would break.

### 6. Bill Query URL Must Be Configured in SSLCommerz Panel
The bill-query URL is configured BOTH in the QR reference setup AND in the ipn_url field of our bill-query response. If the panel configuration is wrong, auto-charges wont work.

### 7. Single QR Reference = Single Subscription Plan
Each QR Refer ID = one subscription plan in SSLCommerz. If you want multiple plans (different frequencies at different fixed amounts), you need multiple Refer IDs. Our current implementation uses a single Refer ID with dynamic amounts from bill-query.
