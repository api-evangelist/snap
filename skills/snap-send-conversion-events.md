---
name: snap-send-conversion-events
description: >-
  Send server-to-server conversion events to Snap with the Conversions API V3 —
  validate the payload first, send the batch, then read the validation logs and
  stats to confirm Snap accepted it.
api: openapi/snap-conversions-api-v3-openapi.yml
operations:
  - sendValidationEvent
  - getValidationLogs
  - getValidationStats
  - sendEvent
generated: '2026-08-13'
method: generated
source: >-
  Grounded in the operationIds of Snap's own OpenAPI (harvested from
  github.com/Snapchat/business-sdk-v3-java) plus
  developers.snap.com/marketing-api/Conversions-API. Every operationId below was
  verified present in openapi/snap-conversions-api-v3-openapi.yml.
---

# Send conversion events to Snap (Conversions API V3)

Use this to report web, app or offline conversions to Snap for campaign
optimization and measurement. This is an **ingestion** API — you POST events to
Snap; Snap never calls you.

## Before you start

- **Base URL:** `https://tr.snapchat.com`
- **Asset id** — every path is keyed by one:
  - web and offline events → **Pixel ID**
  - mobile app events → **Snap App ID**
- **Credential:** a static long-lived Conversions API token, generated in
  Ads Manager → Business Details → Conversions API Tokens. You must be an
  Organization Admin to generate one. It does not expire.
- **Credential placement:** the token goes in the `access_token` **query
  parameter**, not a header. Treat the whole URL as a secret — never log it,
  never put it in a referrer-visible context. See
  `authentication/snap-authentication.yml`.
- The token can only send events for Pixel IDs and Snap App IDs in the **same
  org** the token was generated in. Cross-org calls fail.

## Step 1 — validate before you send (`sendValidationEvent`)

```
POST /v3/{asset_id}/events/validate?access_token={token}
```

Send the exact batch you intend to send for real. Nothing is processed as a
conversion. Do this on every new integration and after any payload change.

## Step 2 — check what the validator found

```
GET /v3/{asset_id}/events/validate/logs?access_token={token}    # getValidationLogs
GET /v3/{asset_id}/events/validate/stats?access_token={token}   # getValidationStats
```

`getValidationLogs` returns recent validation errors; `getValidationStats`
returns counts. **Do not skip this** — a 200 from the validate endpoint is not
by itself proof the events were well formed (see "Reading the response" below).

## Step 3 — send the real batch (`sendEvent`)

```
POST /v3/{asset_id}/events?access_token={token}
Content-Type: application/json
```

Body shape (`sendEvent_request` → `data[]` of `CapiEvent`):

```json
{
  "data": [
    {
      "event_name": "PURCHASE",
      "event_time": 1705508777,
      "event_id": "<your unique id>",
      "action_source": "WEB",
      "event_source_url": "https://example.com/checkout",
      "user_data": { "em": ["<sha256 of lowercased email>"] },
      "custom_data": { "currency": "USD", "value": "10" }
    }
  ]
}
```

### Required on every event

`event_name`, `event_time`, `action_source`, `event_source_url`, and `user_data`
carrying **at least one** matching identifier: `em`, `ph`,
`client_ip_address` + `client_user_agent`, or `madid` (app events only).
App events additionally require `app_data` including `extinfo`.
If `event_name` is `PURCHASE`, `currency` and `value` are also required.

### Hash the identifiers

`em` and `ph` must be **SHA-256 hashed** before sending. Never send a raw email
or phone number.

## Batching and rate

- Up to **2,000 events per request** (`data[]`).
- **10 requests/second** per token by default; Snap recommends up to
  **1,000 QPS** for static long-lived tokens.
- On **HTTP 429** slow down. There is no `Retry-After` and no rate-limit
  header — back off on your own schedule.
  See `rate-limits/snap-rate-limits.yml`.

## Deduplication — this is not idempotency

Send a stable `event_id` on every event across every integration (CAPI, Snap
Pixel, MMP). Snap deduplicates on `event_id` + timestamp within a **48 hour**
window.

This is **reporting-level deduplication, not request idempotency.** Snap
publishes no idempotency key. A retried POST is not guaranteed to be a no-op at
the request level — only the resulting duplicate *events* get collapsed, and
only if you supplied `event_id`. Always set it.

## Reading the response

Failures can arrive inside a **200**. The response is an `EventResponse`
carrying a `status` and, on problems, `ErrorDetails`. A successful body reads:

```json
{ "status": "VALID", "reason": "Events have been processed successfully." }
```

Do not treat `200` as success — parse `status`. See
`errors/snap-error-codes.yml`.

## Consent

- **App events:** pass `advertiser_tracking_enabled` reflecting the user's ATT
  status.
- **Web events:** mark an opt-out by setting `data_use` to `["lmu"]`.

## Related

- Conventions: `conventions/snap-conventions.yml`
- Sandbox / validation surfaces: `sandbox/snap-sandbox.yml`
- Data model: `data-model/snap-data-model.yml`
- v2 (superseded): `openapi/snap-conversions-api-openapi.yml` —
  operations `sendData`, `sendTestData`, `conversionValidateLogs`,
  `conversionValidateStats`. Migrate to V3.
