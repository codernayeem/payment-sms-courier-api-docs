Here is the clean, formatted Markdown representation of the SteadFast Courier API Documentation PDF:

---

# SteadFast Courier Limited

## API Documentation V1



---

### Table of Contents

1. [API Authentication Parameter](https://www.google.com/search?q=%231-api-authentication-parameter)

2. [Placing an Order](https://www.google.com/search?q=%232-placing-an-order)

3. [Bulk Order Create](https://www.google.com/search?q=%233-bulk-order-create)

4. [Checking Delivery Status](https://www.google.com/search?q=%234-checking-delivery-status)

5. [Checking Current Balance](https://www.google.com/search?q=%235-checking-current-balance)

6. [Fraud Check](https://www.google.com/search?q=%236-fraud-check)


---

## 1. API Authentication Parameter

Authentication parameters are required to be added in the header of each request.

* **Base URL:** `[https://portal.packzy.com/api/v1](https://portal.packzy.com/api/v1)`


| Header Name | Type | Description | Value |
| --- | --- | --- | --- |
| `Api-Key` | String | API Key provided by SteadFast Courier Ltd. | `***************` |
| `Secret-Key` | String | Secret Key provided by SteadFast Courier Ltd. | `***************` |
| `Content-Type` | String | Request Content Type | `application/json` |

---

## 2. Placing an Order

* **Path:** `/create_order`

* **Method:** `POST`


### Input Parameters:

| Name | Type | MOC | Description | Example |
| --- | --- | --- | --- | --- |
| `invoice` | String | required | Must be unique and can be alphanumeric including hyphens and underscores. | `12366`, `abc123`, `12abchd`, `Aa12-das4`, `asdfd-wq` |
| `recipient_name` | String | required | Within 100 Characters. | `John Smith` |
| `recipient_phone` | String | required | Must be an 11-digit phone number. | `01234567890` |
| `recipient_address` | String | required | Recipient's address within 250 characters. | `Flat#A1, House#17/1, Road#3/A, Dhanmondi, Dhaka-1209` |
| `cod_amount` | Numeric | required | Cash on delivery amount in BDT including all charges. Cannot be less than 0. | `1060` |
| `note` | String | optional | Delivery instructions or other notes (max 480 characters). | `Deliver within 3PM` |

### Sample Response:

```json
{
  "status": 200,
  "message": "Consignment has been created successfully.",
  "consignment": {
    "consignment_id": 1424107,
    "invoice": "Aa12-das4",
    "tracking_code": "15BAEB8A",
    "recipient_name": "John Smith",
    "recipient_phone": "01234567890",
    "recipient_address": "Flat#A1, House#17/1, Road#3/A, Dhanmondi, Dhaka-1209",
    "cod_amount": 1060,
    "status": "in_review",
    "note": "Deliver within 3PM",
    "created_at": "2021-03-21T07:05:31.000000Z",
    "updated_at": "2021-03-21T07:05:31.000000Z"
  }
}

```

---

## 3. Bulk Order Create

* **Path:** `/create_order/bulk-order`

* **Method:** `POST`


### Input Parameters:

| Name | Type | MOC | Description | Example |
| --- | --- | --- | --- | --- |
| `data` | JSON | required | Maximum 500 items are allowed. JSON-encoded array. | See example below |

### Example Code (PHP / Laravel):

```php
public function bulkCreate() {
    $orders = Order::with('address')->where('status', 1)->take(500)->get();
    $data = array();
    
    foreach ($orders as $order) {
        $data[] = [
            'invoice'           => $order->id,
            'recipient_name'    => $order->address ? $order->address->name : 'N/A',
            'recipient_address' => $order->address ? $order->address->address : 'N/A',
            'recipient_phone'   => $order->address ? $order->address->phone : '',
            'cod_amount'        => $order->due_amount,
            'note'              => $order->note,
        ];
    }
    
    $steadfast = new Steadfast();
    $result = $steadfast->bulkCreate(json_encode($data));
    return $result;
}

public function bulkCreate($data) {
    $api_key = '<<<STEADFAST_API_KEY>>>';
    $secret_key = '<<<STEADFAST_SECRET_KEY>>>';
    
    $response = Http::withHeaders([
        'Api-Key'      => $api_key,
        'Secret-Key'   => $secret_key,
        'Content-Type' => 'application/json'
    ])->post($this->base_url . '/create_order/bulk-order', [
        'data' => $data,
    ]);
    
    return json_decode($response->getBody()->getContents());
}

```

### Success Response:

```json
{
  "status": 200,
  "message": "We have a response for you.",
  "data": [
    {
      "invoice": "SFC-1234j56",
      "recipient_name": "John Smith",
      "recipient_phone": "+88012345678",
      "recipient_address": "Dhanmondi, Zigatola",
      "cod_amount": "3010",
      "status": "success",
      "note": "Hello World",
      "consignment_id": 106833902,
      "tracking_code": "65E27EE263A"
    },
    {
      "invoice": "SFC-12i3456",
      "recipient_name": "John Smith",
      "recipient_phone": "+88012345678",
      "recipient_address": "Dhanmondi, Zigatola",
      "cod_amount": "3010",
      "status": "success",
      "note": "Hello World",
      "consignment_id": 106833903,
      "tracking_code": "65E27EFFEC0"
    },
    {
      "invoice": "SFC-1234156",
      "recipient_name": "John Smith",
      "recipient_phone": "+88012345678",
      "recipient_address": "Dhanmondi, Zigatola",
      "cod_amount": "3010",
      "status": "success",
      "note": "Hello World",
      "consignment_id": 106833904,
      "tracking_code": "65E27F0A1D5"
    }
  ]
}

```

### Error Response Example:

If there is an error in the submitted data, you will receive a response like this:

```json
{
  "data": [
    {
      "invoice": "SFC-1234j56",
      "recipient_name": "John Smith",
      "recipient_phone": "+88012345678",
      "recipient_address": "Dhanmondi, Zigatola",
      "cod_amount": "3010",
      "note": null,
      "consignment_id": null,
      "tracking_code": null,
      "status": "error"
    }
  ]
}

```

---

## 4. Checking Delivery Status

### i) By Consignment ID

* **Path:** `/status_by_cid/{id}`

* **Method:** `GET`


### ii) By Invoice ID

* **Path:** `/status_by_invoice/{invoice}`

* **Method:** `GET`


### iii) By Tracking Code

* **Path:** `/status_by_trackingcode/{trackingCode}`

* **Method:** `GET`


### Sample Response:

```json
{
  "status": 200,
  "delivery_status": "in_review"
}

```

### Delivery Status Values:

| Status Name | Description |
| --- | --- |
| `in_review` | Order is placed and waiting to be reviewed. |
| `pending` | Consignment is not delivered or cancelled yet. |
| `delivered_approval_pending` | Consignment is delivered but waiting for admin approval. |
| `partial_delivered_approval_pending` | Consignment is delivered partially and waiting for admin approval. |
| `cancelled_approval_pending` | Consignment is cancelled and waiting for admin approval. |
| `unknown_approval_pending` | Unknown pending status. Need to contact support team. |
| `delivered` | Consignment is delivered and balance added. |
| `partial_delivered` | Consignment is partially delivered and balance added. |
| `cancelled` | Consignment is cancelled and balance updated. |
| `hold` | Consignment is on hold. |
| `unknown` | Unknown status. Need to contact support team. |

---

## 5. Checking Current Balance

* **Path:** `/get_balance`

* **Method:** `GET`


### Sample Response:

```json
{
  "status": 200,
  "current_balance": 0
}

```

---

## 6. Fraud Check

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