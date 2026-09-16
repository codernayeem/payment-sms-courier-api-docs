Based on the provided documentation for the **Upay Payment Gateway API V4.1.0**, here are the complete payloads and response formats for every endpoint mentioned.

---

## **1. Merchant Authorization**

This is the first step to obtain the `token` required for all other APIs.

* 
**URL:** `<BASE_URL>/payment/merchant-auth/` 


* 
**Method:** `POST` 


* **Request Payload:**

```json
{
  "merchant_id": "4578753234567",
  "merchant_key": "ADSE1234"
}

```



* **Success Response (200):**

```json
{
  "code": "MAS2001",
  "lang": "en",
  "message": "Merchant Auth Success",
  "data": {
    "merchant_id": "4578753234567",
    "token": "eyJ0eXAi0iJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJtZXJjaGFudF9pZCI6IjQ1Nzg3NTMyMzQ1N jciLCJtZXJjaGFudF9rZXkiOiJBRFNFMTIzNCJ9.fowXvY400LoC9fbzbc575ILszDioEEYK5Yu T4P_zyTM"
  }
}

```



* **Error Response (400 - Merchant Not Found):**

```json
{
  "code": "MNF4002",
  "message": "Merchant Not Found",
  "lang": "en"
}

```



---

## **2. Merchant Payment Initiation**

Used to generate the payment URL where the customer will enter their credentials.

* 
**URL:** `<BASE_URL>/payment/merchant-payment-init/` 


* 
**Method:** `POST` 


* 
**Header:** `Authorization: UPAY {token}` 


* **Request Payload:**

```json
{
  "date": "2020-12-08",
  "txn_id": "Upay987654321",
  "invoice_id": "Upay987654321",
  "amount": 2050.00,
  "merchant_id": "4578753234567",
  "merchant_name": "rokomari",
  "merchant_code": "1252",
  "merchant_country_code": "BD",
  "merchant_city": "Dhaka",
  "merchant_category_code": "1252",
  "merchant_mobile": "01756348921",
  "transaction_currency_code": "BDT",
  "redirect_url": "https://www.rokomari.com/checkout/redirect/",
  "additional_info": {
    "data": "example"
  },
  "is_cashback": false,
  "cashback_amount": 205.00,
  "cashback_wallet": "01756348921",
  "seat_count": "2"
}

```



* **Success Response (200):**

```json
{
  "code": "MPIS2002",
  "lang": "en",
  "message": "Merchant Payment Init Success",
  "data": {
    "session_id": "16075882404726658",
    "txn_id": "Upay987654321",
    "trx_id": "01G38BC4X4HX1W4YCE272AFCN2",
    "invoice_id": "Upay987654321",
    "merchant_id": "4578753234567",
    "gateway_url": "https://www.upay.merchant.pay.com/16075882404726658/234567543/"
  }
}

```



---

## **3. Single Payment Status**

Check the status of a specific transaction using the merchant's `txn_id`.

* 
**URL:** `<BASE_URL>/payment/single-payment-status/{txn_id}/` 


* 
**Method:** `GET` 


* 
**Header:** `Authorization: UPAY {token}` 


* **Success Response (200):**

```json
{
  "code": "PS2005",
  "lang": "en",
  "message": "Payment Status Found",
  "data": {
    "txn_id": "Upay987654321",
    "trx_id": "01G38BC4X4HX1W4YCE272AFCN2",
    "invoice_id": "Upay987654321",
    "session_id": "16074983655190547",
    "status": "success",
    "amount": 2050.0,
    "merchant_name": "rokomari",
    "customer_wallet": "0193XXXXX96",
    "date": "2023-01-22"
  }
}

```

---

## **Summary of Status Codes**

| Code | Meaning |
| --- | --- |
| **MAS2001** | Merchant Authorization Success 

 |
| **MPIS2002** | Merchant Payment Initiation Success 

 |
| **PS2005** | Payment Status Found 

 |
| **MPR 200** | Merchant Payment Refunded Successfully 

 |
| **DVE4001** | Data Validation Error (4XX) 

 |
| **MNF4002** | Merchant Not Found 

 |
| **DTP4003** | Duplicate Transaction ID 

 |
| **PNF4012** | Payment Not Found 

 |
| **MSE5001** | Middleware Server Error (5XX) 

 |
