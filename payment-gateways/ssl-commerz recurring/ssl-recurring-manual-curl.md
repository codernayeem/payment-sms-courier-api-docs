# SSLCommerz Recurring Manual Commands (cURL)

Use these commands to manually check and control recurring subscriptions from a server/VPS (recommended if local IP is blocked).

## 1) Set variables (required)

```bash

export SSL_BASE_URL="https://sandbox.sslcommerz.com"
export STORE_ID="testbox"
export STORE_PASSWD="qwerty"
export REFER="5B90BA91AA3F2"

export SUBSCRIPTION_ID="SUBTESTB69B8E03733E6D"
```

---

## 2) Check subscription status

```bash
curl -X POST "$SSL_BASE_URL/validator/api/v4/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "store_id=$STORE_ID" \
  --data-urlencode "store_passwd=$STORE_PASSWD" \
  --data-urlencode "refer=$REFER" \
  --data-urlencode "subscription_id=$SUBSCRIPTION_ID" \
  --data-urlencode "action=getSubscriptionStatus"
```

## 3) cancel
```bash
curl -X POST "$SSL_BASE_URL/validator/api/v4/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "store_id=$STORE_ID" \
  --data-urlencode "store_passwd=$STORE_PASSWD" \
  --data-urlencode "refer=$REFER" \
  --data-urlencode "subscription_id=$SUBSCRIPTION_ID" \
  --data-urlencode "action=cancelSubscription"
```


## 6) Verify after cancel

Run status check again and confirm:
- `APIConnect = DONE`
- `subscription_status = cancelled` (or `deactivated`)

Use command from section **2) Check subscription status**.

---

## Expected response fields to review

- `APIConnect`: should be `DONE`
- `subscription_status`: expected one of `active`, `paused`, `deactivated`, `cancelled`, etc.
- `failedreason`: if any error occurred

---

## Notes

- SSL recurring API does **not** provide a documented `deleteSubscription` action.
- Permanent stop is `cancelSubscription`.
- If local IP is blocked, execute from backend server/VPS with allowed egress IP.
