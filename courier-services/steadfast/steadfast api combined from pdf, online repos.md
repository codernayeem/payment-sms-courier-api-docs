# SteadFast Courier Limited - Complete API Documentation

Comprehensive reference for SteadFast Courier API V1, compiled from official documentation, the SteadFast Laravel package, and the Python SDK.

---

### Table of Contents

1. [API Authentication](#1-api-authentication)
2. [Placing an Order (Single Order)](#2-placing-an-order-single-order)
3. [Bulk Order Create](#3-bulk-order-create)
4. [Checking Delivery Status & Status Types](#4-checking-delivery-status--status-types)
5. [Consignment Cancellation & Return Requests](#5-consignment-cancellation--return-requests)
6. [Webhook Integration](#6-webhook-integration)
7. [Checking Current Balance](#7-checking-current-balance)
8. [Payment & Settlement Statements](#8-payment--settlement-statements)
9. [Coverage & Police Stations](#9-coverage--police-stations)
10. [Fraud Check](#10-fraud-check)

---

## 1. API Authentication

Authentication parameters must be passed in the headers of each HTTP request.

* **Base URL:** `https://portal.packzy.com/api/v1` *(also accessible via `https://portal.steadfast.com.bd/api/v1`)*

| Header Name | Type | Description | Example |
| --- | --- | --- | --- |
| `Api-Key` | String | API Key provided by SteadFast Courier Ltd. | `<<<STEADFAST_API_KEY>>>` |
| `Secret-Key` | String | Secret Key provided by SteadFast Courier Ltd. | `<<<STEADFAST_SECRET_KEY>>>` |
| `Content-Type` | String | Request Content Type | `application/json` |

---

## 2. Placing an Order (Single Order)

* **Path:** `/create_order`
* **Method:** `POST`

### Request Payload Parameters:

| Name | Type | Requirement | Description | Example |
| --- | --- | --- | --- | --- |
| `invoice` | String | **Required** | Unique identifier (alphanumeric, hyphens, underscores). | `ORD-2026-001`, `ECO26091619` |
| `recipient_name` | String | **Required** | Recipient's full name (max 100 characters). | `John Smith` |
| `recipient_phone` | String | **Required** | 11-digit phone number. | `01711111111` |
| `recipient_address` | String | **Required** | Full address (max 250 characters). | `Flat#A1, House#17/1, Road#3/A, Dhanmondi, Dhaka` |
| `cod_amount` | Numeric | **Required** | Cash on delivery amount in BDT (>= 0). | `1060` |
| `note` | String | Optional | Delivery instructions (max 480 chars). | `Call before delivery` |
| `delivery_type` | Integer | Optional | `0` = Home Delivery (default), `1` = Point Delivery. | `0` |
| `alternative_phone` | String | Optional | Secondary 11-digit phone number. | `01811111111` |
| `recipient_email` | String | Optional | Recipient's email address. | `customer@example.com` |
| `item_description` | String | Optional | Description of goods inside the parcel. | `Clothes / Electronics` |
| `total_lot` | Integer | Optional | Total lot / piece count (>= 1). | `2` |

### Success Response:

```json
{
  "status": 200,
  "message": "Consignment has been created successfully.",
  "consignment": {
    "consignment_id": 1424107,
    "invoice": "ECO26091619",
    "tracking_code": "15BAEB8A",
    "recipient_name": "John Smith",
    "recipient_phone": "01711111111",
    "recipient_address": "Flat#A1, House#17/1, Road#3/A, Dhanmondi, Dhaka",
    "cod_amount": 1060,
    "status": "in_review",
    "note": "Call before delivery",
    "created_at": "2026-09-16T07:05:31.000000Z",
    "updated_at": "2026-09-16T07:05:31.000000Z"
  }
}
```

> [!IMPORTANT]
> Both `consignment_id` and `tracking_code` are generated and returned **only upon initial order creation**. The status check endpoints only return `delivery_status`, so always store `consignment_id` and `tracking_code` immediately in your database.

---

## 3. Bulk Order Create

* **Path:** `/create_order/bulk-order` (or `/create_bulk_order`)
* **Method:** `POST`

### Request Payload:
Accepts a JSON array with up to **500 items**.

```json
{
  "data": [
    {
      "invoice": "INV-001",
      "recipient_name": "John Doe",
      "recipient_phone": "01711111111",
      "recipient_address": "House 44, Road 2/A, Dhanmondi, Dhaka",
      "cod_amount": 1000,
      "note": "Handle with care"
    },
    {
      "invoice": "INV-002",
      "recipient_name": "Jane Smith",
      "recipient_phone": "01822222222",
      "recipient_address": "456 Elm St, Chittagong",
      "cod_amount": 1500,
      "note": "Fragile"
    }
  ]
}
```

### Success Response:

```json
{
  "status": 200,
  "message": "We have a response for you.",
  "data": [
    {
      "invoice": "INV-001",
      "recipient_name": "John Doe",
      "recipient_phone": "01711111111",
      "recipient_address": "House 44, Road 2/A, Dhanmondi, Dhaka",
      "cod_amount": "1000.00",
      "note": "Handle with care",
      "consignment_id": 11543968,
      "tracking_code": "B025A038",
      "status": "success"
    },
    {
      "invoice": "INV-002",
      "recipient_name": "Jane Smith",
      "recipient_phone": "01822222222",
      "recipient_address": "456 Elm St, Chittagong",
      "cod_amount": "1500.00",
      "note": "Fragile",
      "consignment_id": 11543969,
      "tracking_code": "B025A1DC",
      "status": "success"
    }
  ]
}
```

---

## 4. Checking Delivery Status & Status Types

Steadfast provides 3 endpoints to check the live delivery status of an order:

1. **By Consignment ID:** `GET /status_by_cid/{id}`
2. **By Invoice:** `GET /status_by_invoice/{invoice}`
3. **By Tracking Code:** `GET /status_by_trackingcode/{trackingCode}`

### Sample Response:
```json
{
  "status": 200,
  "delivery_status": "in_review"
}
```

### All Delivery Status Types & Meanings:

| Status Code | Meaning / Description |
| --- | --- |
| `in_review` | Order has been placed and is waiting for review. |
| `pending` | Consignment is approved/queued, waiting for rider pickup. |
| `in_transit` | Consignment is picked up and in transit between hubs. |
| `picked` | Consignment has been picked up by the delivery agent. |
| `delivered_approval_pending` | Consignment delivered, awaiting admin approval. |
| `partial_delivered_approval_pending` | Consignment partially delivered, awaiting approval. |
| `cancelled_approval_pending` | Consignment cancelled, awaiting admin approval. |
| `unknown_approval_pending` | Unknown pending state. Contact Steadfast support. |
| `delivered` | Successfully delivered to recipient and balance credited. |
| `partial_delivered` | Partially delivered and balance adjusted. |
| `cancelled` | Consignment cancelled and balance updated. |
| `hold` | Consignment is on hold (e.g. customer requested delayed delivery). |
| `unknown` | Unknown status. Contact support. |

---

## 5. Consignment Cancellation & Return Requests

> [!NOTE]
> **Is there a direct Delete / Cancel Consignment endpoint?**
> Steadfast does **not** provide a simple `DELETE` or `POST /cancel_order` endpoint for already created consignments.
> To cancel or request a return for a consignment via API, Steadfast uses the **Return Request API**.
> (If a parcel is still in `in_review` or not yet picked up, it can also be cancelled via the Steadfast Merchant Portal UI or by contacting support).

### 5.1 Create Return Request (Cancellation / Return)

* **Path:** `/return-request/store`
* **Method:** `POST`

#### Request Payload:

| Name | Type | Requirement | Description |
| --- | --- | --- | --- |
| `identifier` | String / Numeric | **Required** | The ID or Code identifying the consignment. |
| `identifier_type` | String | **Required** | Type of identifier: `"consignment_id"`, `"invoice"`, or `"tracking_code"`. Default: `"consignment_id"`. |
| `reason` | String | Optional | Reason for cancellation or return request (e.g., `"Customer cancelled order"`, `"Item damaged"`). |

```json
{
  "identifier": 1424107,
  "identifier_type": "consignment_id",
  "reason": "Customer cancelled order before dispatch"
}
```

#### Success Response:
```json
{
  "status": 200,
  "id": 12,
  "user_id": 450,
  "consignment_id": 1424107,
  "reason": "Customer cancelled order before dispatch",
  "status": "pending",
  "created_at": "2026-09-16T08:00:00.000000Z",
  "updated_at": "2026-09-16T08:00:00.000000Z"
}
```

### 5.2 Get Return Request Details

* **Path:** `/return-request/{id}`
* **Method:** `GET`

### 5.3 List Return Requests

* **Path:** `/return-request/list`
* **Method:** `GET`

#### Return Request Statuses:
* `pending` — Request submitted, awaiting processing.
* `approved` — Return request approved.
* `processing` — Return parcel is in transit back to merchant.
* `completed` — Return completed and received.
* `cancelled` — Return request was rejected or cancelled.

---

## 6. Webhook Integration

Instead of polling status endpoints continuously, Steadfast can notify your server in real-time when the status of a parcel changes.

### Configuration
In the Steadfast Merchant Portal:
1. Set **Callback URL**: `https://your-domain.com/api/webhooks/steadfast`
2. Set **Auth Token (Bearer)**: A secure random token shared between Steadfast and your application.

### Incoming Webhook Request Format
* **Method:** `POST`
* **Headers:**
  ```http
  Authorization: Bearer <your-configured-bearer-token>
  Content-Type: application/json
  ```
* **Payload:**
  ```json
  {
    "consignment_id": 1424107,
    "invoice": "ECO26091619",
    "status": "delivered",
    "cod_amount": 1060,
    "updated_at": "2026-09-16T12:00:00.000000Z"
  }
  ```

### Handling Webhook in Node.js / Next.js:

```typescript
// Example: Next.js API Route / App Router handler
export async function POST(req: Request) {
  const authHeader = req.headers.get("Authorization");
  const expectedToken = `Bearer ${process.env.STEADFAST_WEBHOOK_BEARER_TOKEN}`;

  if (authHeader !== expectedToken) {
    return new Response(JSON.stringify({ error: "Unauthorized" }), { status: 401 });
  }

  const payload = await req.json();
  const { consignment_id, invoice, status, cod_amount, updated_at } = payload;

  // Process & sync status in your database
  // e.g. update delivery tracking status for invoice

  return new Response(JSON.stringify({ status: "success" }), { status: 200 });
}
```

---

## 7. Checking Current Balance

* **Path:** `/get_balance`
* **Method:** `GET`

### Sample Response:
```json
{
  "status": 200,
  "current_balance": 15420.50
}
```

---

## 8. Payment & Settlement Statements

APIs to view payout transactions and settlements made by Steadfast to your bank account / merchant wallet.

### 8.1 List Payments / Settlements
* **Path:** `/payment/list`
* **Method:** `GET`

#### Sample Response:
```json
{
  "status": 200,
  "data": [
    {
      "id": 101,
      "amount": 25400.00,
      "created_at": "2026-09-10T10:00:00.000000Z",
      "updated_at": "2026-09-10T10:00:00.000000Z"
    }
  ]
}
```

### 8.2 Get Payment Details
* **Path:** `/payment/{payment_id}`
* **Method:** `GET`

#### Sample Response:
```json
{
  "status": 200,
  "id": 101,
  "amount": 25400.00,
  "consignments": [
    {
      "consignment_id": 1424107,
      "invoice": "ECO26091619",
      "cod_amount": 1060,
      "delivery_charge": 60,
      "cod_charge": 10.60
    }
  ],
  "created_at": "2026-09-10T10:00:00.000000Z",
  "updated_at": "2026-09-10T10:00:00.000000Z"
}
```

---

## 9. Coverage & Police Stations

Retrieve list of supported police stations / thanas for coverage validation and return hubs.

* **Path:** `/location/police-stations`
* **Method:** `GET`

#### Sample Response:
```json
{
  "status": 200,
  "data": [
    {
      "id": 1,
      "name": "Dhanmondi",
      "location": "Dhaka"
    },
    {
      "id": 2,
      "name": "Mirpur",
      "location": "Dhaka"
    }
  ]
}
```

---

## 10. Fraud Check

Check customer delivery history and risk profile based on their phone number.

* **Path:** `/fraud_check/{phone}`
* **Method:** `GET`

### Sample Response:
```json
{
  "total_parcels": 17,
  "total_delivered": 7,
  "total_cancelled": 10,
  "total_fraud_reports": 0
}
```