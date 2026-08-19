---
name: snap-receive-lead-webhooks
description: >-
  Register a webhook against a Snap lead generation form, fire Snap's test lead
  to prove delivery end to end, and consume the lead payload safely.
api: Snapchat Marketing API (Ads API)
operations:
  - "POST /v1/lead_gen/integrations/public_webhook"
  - "GET /v1/lead_gen/integrations/{integration_id}/test"
  - "GET /v1/lead_gen/forms/{form_id}/integrations"
generated: '2026-08-13'
method: generated
source: >-
  Grounded in https://developers.snap.com/api/marketing-api/Ads-API/lead-generation-ads.
  Snap publishes no OpenAPI for the Ads API, so the operations above are the
  documented HTTP request lines verbatim, not invented operationIds.
---

# Receive Snapchat lead generation webhooks

When a Snapchatter submits a lead-gen form, Snap POSTs the lead to a URL you
register. This is Snap's only outbound event surface.

## Prerequisites

- **Organization Admin** access on the ad account that owns the form.
- An OAuth 2.0 access token with the `snapchat-marketing-api` scope
  (`Authorization: Bearer {access_token}`). See
  `authentication/snap-authentication.yml`.
- A public HTTPS endpoint that responds fast and does its work asynchronously.

## Step 1 — register the webhook

```
POST https://adsapi.snapchat.com/v1/lead_gen/integrations/public_webhook
Content-Type: application/json
Authorization: Bearer {access_token}

{"webhook_integrations":[{"form_id":"<form uuid>","webhook_url":"https://your.host/snap/leads"}]}
```

The response returns `integrationId` and — **read once, store securely** — an
`hmacSecret`.

**Only one webhook integration can exist per form.** Registering a second one
for the same form is not supported.

## Step 2 — prove delivery with Snap's test lead

```
GET https://adsapi.snapchat.com/v1/lead_gen/integrations/{integration_id}/test
Authorization: Bearer {access_token}
```

Snap fires a fixed dummy lead at your registered URL. The payload is the same
every time regardless of your form's real fields — `form_name` will read
"Snap Lead Generation Form", the email will be `johndoe@snapchat.com`. Use it to
confirm reachability and parsing, **not** to test your field mapping.

## Step 3 — consume the payload

Always present: `form_id`, `form_name`, `ad_account_id`, `campaign_id`,
`campaign_name`, `ad_id`, `ad_name`, `ad_squad_id`, `ad_squad_name`, `lead_id`,
`create_time` (millisecond UNIX timestamp, sent as a **string**).

Everything else is conditional on how the advertiser built the form — treat
`first_name`, `last_name`, `email`, `phone_number`, address fields, `birthday`,
`job_title`, `company_name`, `custom_field_1`–`custom_field_8`, `consent_1`,
`consent_2` and `lead_preferred_status` as **optional**. Do not assume presence.

Use `lead_id` as your dedupe key.

## Handle the PII correctly

The lead payload carries **plaintext PII** — name, email, phone, address,
birthday. This is the opposite of the Conversions API, which requires SHA-256
hashed identifiers. Do not log the raw body, and honour `consent_1` /
`consent_2` before any downstream use. Snap's Business Services and Business
Tools Terms bind retention and deletion.

## Verifying authenticity — read this before you rely on it

Snap says the `hmacSecret` "could be used to verify the authenticity of the
webhook events" but **publishes neither the signature header name nor the digest
algorithm**, so HMAC verification cannot be implemented from the public docs
alone. Until Snap documents it:

- Register a webhook URL containing a long unguessable path segment.
- Restrict by source where you can, and validate the payload shape.
- Reconcile received leads against
  `GET /v1/lead_gen/forms/{form_id}/integrations?partner_type=PUBLIC_WEBHOOK`
  and the lead read endpoints — do not treat the webhook as the sole system of
  record.

Snap publishes **no retry schedule, no delivery guarantee, no ordering guarantee
and no replay mechanism**. Assume a missed delivery is lost and reconcile by
polling.

## Related

- Webhook catalog: `asyncapi/snap-lead-gen-webhooks.yml`
- Conventions: `conventions/snap-conventions.yml`
- Sandbox: `sandbox/snap-sandbox.yml`
