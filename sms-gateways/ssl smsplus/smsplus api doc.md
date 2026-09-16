# SSL Wireless Push SMS API V3.0.0 Technical Integration Guide

[cite_start]This guide focuses strictly on the technical parameters and endpoints required for integration with the SSL Wireless Push SMS Gateway. [cite: 9, 10]

---

## 1. Status Codes & Error Handling

### 1.1 API Status Codes (Response Level)
[cite_start]These codes indicate the result of the HTTP request to the gateway. [cite: 76, 77]

| Code | Status | Meaning / Description |
| :--- | :--- | :--- |
| **200** | SUCCESS | [cite_start]Successful request. [cite: 79] |
| **4001** | FAILED | [cite_start]Unauthorized: Invalid API token. [cite: 79] |
| **4002** | FAILED | [cite_start]SID/Stakeholder is not permitted to send SMS. [cite: 79] |
| **4003** | FAILED | [cite_start]IP Blacklisted: Client IP is blocked in SSL gateway. [cite: 79] |
| **4005** | FAILED | [cite_start]Invalid request format: Not valid JSON or wrong content type. [cite: 79] |
| **4022** | FAILED | [cite_start]Required Parameter Missing: Mandatory field empty. [cite: 79] |
| **4023** | FAILED | [cite_start]Duplicate CSMS ID: Reference ID used already today. [cite: 79] |
| **4027** | FAILED | [cite_start]Message length exceeded. [cite: 79] |
| **4029** | FAILED | [cite_start]Too many requests: API Request limit exceeded. [cite: 79] |
| **4031** | FAILED | [cite_start]TPS Exceeded: Transaction Per Second limit reached. [cite: 79] |

### 1.2 SMS Status Codes (Message Level)
[cite_start]These codes appear within the `smsinfo` array for individual recipient status. [cite: 82]

| Message | Description |
| :--- | :--- |
| **SUCCESS** | [cite_start]Message sent successfully. [cite: 83] |
| **Invalid MSISDN** | [cite_start]The provided mobile number is invalid. [cite: 83] |
| **Duplicate MSISDN** | [cite_start]MSISDN is duplicate within the same request. [cite: 83] |
| **Duplicate CMS ID** | [cite_start]Reference ID (csms_id) is duplicate for the day. [cite: 83] |

---

## 2. API Endpoints

**Base Domain:** `https://smsplus.sslwireless.com`

### 2.1 Single SMS
[cite_start]Use this to send one message to one recipient. [cite: 95]

* [cite_start]**URL:** `https://smsplus.sslwireless.com/api/v3/send-sms` [cite: 97]
* [cite_start]**Method:** POST / GET [cite: 98]
* [cite_start]**Content-Type:** JSON [cite: 99]

**Request Body:**
| Parameter | Mandatory | Type | Length | Description |
| :--- | :--- | :--- | :--- | :--- |
| `api_token` | Yes | String | 50 | [cite_start]Your authentication token provided by SSL. [cite: 103] |
| `sid` | Yes | String | 20 | [cite_start]Your unique Masking/Sender ID. [cite: 103] |
| `msisdn` | Yes | Numeric | 16 | [cite_start]Recipient mobile number (e.g., 88017xxxxxxxx). [cite: 103] |
| `sms` | Yes | String | 1000 | [cite_start]The message content. [cite: 103] |
| `csms_id` | Yes | String | 20 | [cite_start]Your unique reference ID for this specific SMS. [cite: 103] |

---

### 2.2 Bulk SMS
[cite_start]Use this to send a **common** message to multiple recipients (up to 100). [cite: 142, 147]

* [cite_start]**URL:** `https://smsplus.sslwireless.com/api/v3/send-sms/bulk` [cite: 144]
* [cite_start]**Method:** POST [cite: 144]

**Request Body:**
| Parameter | Mandatory | Type | Description |
| :--- | :--- | :--- | :--- |
| `msisdn` | Yes | **Array** | [cite_start]List of recipient numbers (e.g., `["019xxx", "017xxx"]`). [cite: 150] |
| `batch_csms_id`| Yes | String | [cite_start]Unique reference ID for the entire batch. [cite: 150] |
| *Note* | - | - | [cite_start]Other parameters (`api_token`, `sid`, `sms`) same as Single SMS. [cite: 150] |

---

### 2.3 Dynamic SMS
[cite_start]Use this to send **unique** messages to different recipients in one call (up to 100). [cite: 212, 217]

* [cite_start]**URL:** `https://smsplus.sslwireless.com/api/v3/send-sms/dynamic` [cite: 214]
* [cite_start]**Method:** POST [cite: 214]

[cite_start]**Request Body Structure:** [cite: 225]
```json
{
  "api_token": "YOUR_TOKEN",
  "sid": "YOUR_SID",
  "sms": [
    {
      "msisdn": "88019XXXXXXXX",
      "text": "Unique Message 1",
      "csms_id": "REF_001"
    },
    {
      "msisdn": "88017XXXXXXXX",
      "text": "Unique Message 2",
      "csms_id": "REF_002"
    }
  ]
}
```

---

## 3. Response Structure (JSON)

[cite_start]Every API call returns a JSON object with the following key fields: [cite: 133, 134]

| Field | Description |
| :--- | :--- |
| `status` | [cite_start]Global request status (`SUCCESS` or `FAILED`). [cite: 135] |
| `status_code` | [cite_start]API level status code (e.g., 200). [cite: 135] |
| `smsinfo` | [cite_start]An array containing details for every MSISDN processed. [cite: 135, 208] |
| `smsinfo[].sms_status`| [cite_start]Status of that specific message. [cite: 135] |
| `smsinfo[].reference_id`| [cite_start]**SSL generated** unique ID for tracking/delivery reports. [cite: 135, 208] |
| `smsinfo[].sms_type` | [cite_start]`EN` for English/Standard, `BN` for Unicode/Bangla. [cite: 135] |

Would you like me to write a ready-to-use Python or Node.js function to handle these requests?