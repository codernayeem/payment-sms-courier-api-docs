# Payment, SMS & Courier API Documentation Archive

A centralized repository containing API specifications, technical integration flows, official documentation PDFs, email communication records, and sandbox testing results for Payment Gateways, SMS Providers, and Courier Services.

> [!NOTE]
> **Credential Sanitization:** All live credentials, API keys, passwords, private keys, and secrets across all files have been replaced with standard sanitized placeholders like `<<<PLACEHOLDER>>>`.

---

## Repository Directory Structure

```
payment-sms-courier-api-docs/
├── payment-gateways/
│   ├── bkash/                         # bKash Tokenized Checkout (V2)
│   ├── bkash-recurring/               # bKash Recurring Payment Gateway (RPP)
│   ├── nagad/                         # Nagad Online PGW (DFS API)
│   ├── ssl-commerz/                   # SSLCOMMERZ Standard Checkout (v4)
│   ├── ssl-commerz recurring/         # SSLCOMMERZ Recurring & Easycheckout
│   └── upay/                          # Upay (UCB Fintech) PGW (v4.1.0)
│
├── sms-gateways/
│   ├── bulksmsbd/                     # BulkSMSBD Single/Multi SMS & OTP API
│   └── ssl smsplus/                   # SSL Wireless SMSPlus v3.0.0 Push SMS
│
└── courier-services/
    └── steadfast/                     # SteadFast Courier API v1
```

---

## Service Summaries

### 1. Payment Gateways (`payment-gateways/`)

* **`bkash/`**
  * Tokenized Checkout V2 integration: error codes (`bkash_error_codes.txt`), API request traces (`bkash_testing_example.txt`), official specification (`PGW Tokenized Payment V2(Non-Beta) -- API Specification - V1.2.pdf`), email communication threads (`email1.txt` through `email4.txt`, `email3.0-reply.txt`).

* **`bkash-recurring/`**
  * bKash Recurring Payment Gateway (RPP) documentation: full architecture flow (`bkash-recurring-flow.md`), Mermaid sequence diagrams (`flow-diagram.md`), OpenAPI 3.0 Postman spec (`api-docs.json`), API payload examples (`api-example.txt`), official guides (`Recurring Payment Merchant Integration Guide v2.1.2.pdf` / `.txt`, `Recurring Payment Sample API Request.pdf`), email threads (`email1.txt`, `email1-reply.txt`, `email2.txt`, `email2-reply.txt`), and sandbox testing reports (`testing/` with `Sandbox Test Template.txt`, `bkash-RPP-sandbox-report.pdf` / `.tex`).

* **`nagad/`**
  * Nagad Online Payment PGW: official guide (`ilide.info-nagad-online-payment-api-integration-guide-v3-3.pdf` / `.txt`), RSA key templates (`merchantPrivateKey.txt`, `pgPublicKey.txt`), onboarding emails (`email1.txt`, `email1-task.txt`, `email2.txt`), and live merchant portal reference screenshots.

* **`ssl-commerz/`**
  * SSLCOMMERZ Standard Checkout v4.00 technical guide (`Documentation for SSLCOMMERZ.md`) and store credentials/endpoints reference (`creds.txt`).

* **`ssl-commerz recurring/`**
  * SSLCOMMERZ Recurring & Subscription flow (`sslcommerz-recurring-flow.md`), manual cURL execution guide (`ssl-recurring-manual-curl.md`), Postman collection (`ssl-recurring-subscription-manual.postman_collection.json`), Refer/Salt email note (`recurring-creds.txt`), and official Easycheckout guide (`SSLCOMMERZ_Recurring_Easycheckout_v2.02.pdf`).

* **`upay/`**
  * Upay PGW v4.1.0 official guide (`Upay payment gateway API doc for merchant V4.1.0.pdf` / `.md`), live credentials email (`Live_Credentials.txt`), and test credentials (`Test_Credetials_of_PGW 4.txt`).

---

### 2. SMS Gateways (`sms-gateways/`)

* **`bulksmsbd/`**
  * Single/Multi-recipient SMS, OTP integration, HTTP POST parameters, and response status codes table (`bulksmsbd api doc.md`).

* **`ssl smsplus/`**
  * SSL Wireless Push SMS API v3.0.0 integration guide (`Technical-Specification-of-Push-SMS-Connectivity-API-V3.0.0.pdf` / `smsplus api doc.md` - PDF & Markdown versions).

---

### 3. Logistics & Courier Services (`courier-services/`)

* **`steadfast/`**
  * Official SteadFast Courier API v1 guide (`steadfast api doc.pdf` / `steadfast api doc.md` - PDF & Markdown versions) and extended guide (`steadfast api combined from pdf, online repos.md`).
