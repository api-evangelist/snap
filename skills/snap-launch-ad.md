---
name: snap-launch-ad
description: Launch a Snap Ad end-to-end via the Snapchat Marketing API — authenticate, find the ad account, upload media, create a creative, campaign, ad squad, and ad.
api: Snapchat Marketing API
base_url: https://adsapi.snapchat.com/v1
auth: oauth2 (Authorization: Bearer {access_token})
operations:
  - GET /v1/me/organizations
  - POST /v1/adaccounts/{ad_account_id}/media
  - POST /v1/media/{media_id}/upload
  - POST /v1/adaccounts/{ad_account_id}/creatives
  - POST /v1/adaccounts/{ad_account_id}/campaigns
  - POST /v1/campaigns/{campaign_id}/adsquads
  - POST /v1/adsquads/{ad_squad_id}/ads
---

# Launch a Snap Ad

Operating instructions for launching a single Snap Ad through the Snapchat
Marketing API. All requests use base URL `https://adsapi.snapchat.com/v1` and an
OAuth 2.0 Bearer token (see `authentication/snap-authentication.yml`).

## Preconditions
- OAuth app authorized with scope `snapchat-marketing-api`.
- A valid access token (refresh on `401` / `The access token expired`).

## Steps

1. **Find your account context.**
   `GET /me/organizations?with_ad_accounts=true` — pick the `organization_id`
   and `ad_account_id`.

2. **Create the media record.**
   `POST /adaccounts/{ad_account_id}/media` with `{"media":[{"name":...,"type":"VIDEO","ad_account_id":...}]}`.
   Capture the returned `media_id`.

3. **Upload the media bytes.**
   `POST /media/{media_id}/upload` as `multipart/form-data` with `file=@video.mp4`.

4. **Create the creative.**
   `POST /adaccounts/{ad_account_id}/creatives` with `type:"SNAP_AD"`,
   `top_snap_media_id: {media_id}`, headline, and optional `profile_properties`.
   Capture `creative_id`.

5. **Create the campaign.**
   `POST /adaccounts/{ad_account_id}/campaigns` with `status:"ACTIVE"`,
   `buy_model:"AUCTION"`, and `objective_v2_properties`.
   Capture `campaign_id`.

6. **Create the ad squad.**
   `POST /campaigns/{campaign_id}/adsquads` with targeting, `billing_event`,
   `optimization_goal`, `bid_strategy`, budget (`daily_budget_micro`), and
   `start_time`. Capture `ad_squad_id`.

7. **Create the ad.**
   `POST /adsquads/{ad_squad_id}/ads` with `creative_id`, `type:"SNAP_AD"`,
   `status:"ACTIVE"`.

## Conventions & guardrails
- **Envelope:** every response carries `request_status`, `request_id`, and
  per-entity `sub_request_status`. Log `request_id` for support.
- **Batch limits:** ≤10 creatives/media, ≤30 campaigns/ad squads/ads per request
  (`conventions/snap-conventions.yml`).
- **No idempotency key:** the API does not dedupe retries — guard against
  duplicate creation client-side.
- **Eligibility:** validate optimization-goal / conversion-window eligibility
  before creation to avoid rejections (see `changelog/snap-changelog.yml`).
- **Errors:** JSON Patch updates can return `E1165`/`E1166`; immutable-field
  edits return `E2023` (`errors/snap-error-codes.yml`).
- **Pagination:** list calls (`GET /adaccounts/{id}/ads?limit=50`) are cursor
  paginated via `paging.next_link`.
