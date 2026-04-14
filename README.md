# Affury Test Partner

Minimal test client for verifying Affury S2S (server-to-server) integration. Use it to test the full flow: click tracking, conversion postback, and response handling.

## Quick Start

1. Open your Affury tracking link in a browser — it redirects here with `?click_id=clk_XXX`
2. Fill the fake registration form (name, email, age)
3. Click "Complete Registration" — a postback is sent to Affury API
4. See the result: success, duplicate, age_restricted, geo_restricted, or error

## How It Works

```
Affiliate Ad  →  Affury Tracker (/t/:offerId)  →  302 redirect with click_id
                                                          ↓
                                                  This test partner page
                                                  (extracts click_id from URL,
                                                   saves to cookie)
                                                          ↓
                                                  User "registers" (fake form)
                                                          ↓
                                                  POST /internal/postback
                                                  {click_id, event_type, age, geo}
                                                          ↓
                                                  Response: success / rejected / error
```

## Integration Guide (for your own product)

### Step 1: Extract click_id from URL

When a user arrives from an Affury tracking link, the URL contains `?click_id=clk_XXXXXXXXXXXXXXXXXXXXXXXX`.

```javascript
const clickId = new URLSearchParams(window.location.search).get('click_id');
```

### Step 2: Store click_id

Save it in a cookie or session so it persists through the registration flow.

```javascript
document.cookie = `affury_click_id=${clickId};max-age=${86400*30};path=/;SameSite=Lax`;
```

### Step 3: Send postback from your server

After a user successfully registers, send a server-to-server POST:

```bash
curl -X POST https://api.affury.com/internal/postback \
  -H "Content-Type: application/json" \
  -H "X-Internal-Key: YOUR_API_KEY" \
  -d '{
    "click_id": "clk_XXXXXXXXXXXXXXXXXXXXXXXX",
    "event_type": "REGISTRATION",
    "age": 45,
    "geo": "US"
  }'
```

**Important:** In production, call this from your backend server, never from the browser.

### Step 4: Handle the response

```json
{"success": true, "conversionId": "cnv_XXXXXXXXXXXXXXXXXXXXXXXX"}
```

## All Possible Responses

| HTTP | Response | Meaning |
|------|----------|---------|
| 200 | `{success: true, conversionId: "cnv_..."}` | Conversion created |
| 200 | `{success: true, duplicate: true}` | Already recorded (safe to retry) |
| 200 | `{success: false, reason: "age_restricted"}` | User age < 40, no payout |
| 200 | `{success: false, reason: "geo_restricted"}` | Non-US traffic, no payout |
| 200 | `{success: false, reason: "age_missing"}` | Age field is required |
| 200 | `{success: false, reason: "offer_not_active"}` | Offer paused or expired |
| 400 | `{error: {code: "BAD_REQUEST"}}` | Missing click_id or event_type |
| 401 | `{error: {code: "UNAUTHORIZED"}}` | Invalid API key |
| 404 | `{error: {code: "NOT_FOUND"}}` | Unknown click_id |
| 429 | `{error: {code: "TOO_MANY_REQUESTS"}}` | Rate limit (300/min per key) |

## Rules

- **Age 40+** required for payout (younger users are rejected)
- **US only** — non-US geo is rejected
- **Idempotent** — duplicate postbacks return `{success: true, duplicate: true}`
- **Event types:** `REGISTRATION` (required) and `DEPOSIT` (optional, for revenue tracking)
- **Postback should come from your server**, not the browser (API key exposure)

## Links

- [Affury Portal](https://portal.affury.com)
- [Integration Spec](https://github.com/yrkan/affury/blob/master/docs/letdate-integration-spec.md)
- [Affury Landing](https://affury.com)
