# bKash Recurring Payment Gateway (RPP) Documentation

## Directory Files Summary

| File | Description |
| :--- | :--- |
| `bkash-recurring-flow.md` | Complete end-to-end flow documentation, database models, lifecycle scenarios, HMAC-SHA256 webhook handling, and idempotency. |
| `flow-diagram.md` | Mermaid sequence diagram of subscription creation, consent, callback, and auto-debit webhooks. |
| `api-docs.json` | OpenAPI 3.0.1 specification for bKash recurring gateway endpoints (can be imported to Postman/Swagger). |
| `api-example.txt` | API request and response payload examples for create, webhook, and verification. |
| `Recurring Payment Merchant Integration Guide v2.1.2.pdf` | Official bKash Recurring Payment Merchant Integration Guide v2.1.2. |
| `Recurring Payment Merchant Integration Guide v2.1.2.txt` | Text extraction of the official bKash recurring payment integration guide. |
| `Recurring Payment Sample API Request.pdf` | Official sample API requests document. |
| `email1.txt` | Initial recurring payment onboarding email with milestones and requirements. |
| `email1-reply.txt` | Merchant reply providing Display Name, Redirect URL, and Webhook URL (sanitized). |
| `email2.txt` | bKash follow-up regarding sandbox test completion and production sign-off. |
| `email2-reply.txt` | Merchant confirmation of readiness for sandbox validation. |
| `testing/` | Sandbox testing directory containing test cases template (`Sandbox Test Template.txt`), compiled PDF report (`bkash-RPP-sandbox-report.pdf`), and LaTeX source (`bkash-RPP-sandbox-report.tex`). |
