---
name: Provision a LISNR application and its tokens
description: >-
  Stand up a LISNR application against the right SDK generation and transaction type, mint the API token
  that calls the Tones Service and the SDK token that initializes a device SDK, and find the SDK release to
  ship with. Covers the entitlement checks that gate every step and the credential-handling rules.
api: openapi/lisnr-portal-openapi-derived.json
operations:
- login
- listAccounts
- listApps
- createApp
- getApp
- listApiTokens
- createApiToken
- deleteApiToken
- listSdkTokens
- createSdkToken
- deleteSdkToken
- listSdkReleases
generated: '2026-07-19'
method: generated
source: openapi/lisnr-portal-openapi-derived.json
---

# Provision a LISNR application and its tokens

Everything in LISNR hangs off an **application**. The app fixes which SDK generation you build against,
which tone profiles you may generate, and which transaction type is stamped on every tone. Get the app
wrong and the tokens minted under it are wrong too.

> **Provenance.** LISNR publishes no specification for the Portal API. The operations referenced here were
> observed in the first-party LISNR Portal client (build 2025-12-16) and are described in
> `openapi/lisnr-portal-openapi-derived.json`. Paths, methods and the auth scheme are observed facts;
> request and response bodies are not documented by LISNR. Verify against LISNR before automating against
> this in production.

## Step 1 — authenticate

`login` (`POST /v2/auth/login`) exchanges a username and password for a JWT. Send that JWT on every
subsequent call as `Authorization: JWT <token>` — the literal prefix is `JWT `, not `Bearer`.

A 401 anywhere afterwards means the session is invalid; the Portal itself logs the user out on 401. A 403
carrying "Your user account has been disabled" means an administrator must re-enable the user.

## Step 2 — check what the account is licensed for

`listAccounts` (`GET /v2/accounts`) returns the account, including a `products` array. This is the
entitlement gate for the whole product:

| Product | What it unlocks |
| --- | --- |
| `legacy` | Legacy SDK. Profiles `standard`, `compression`. |
| `radius` | Radius SDK. Profiles `standard2`, `standard2_wideband`, `pkab2`, `pkab2_wideband`. Unidirectional single channel, plus bidirectional transceivers. |
| `radius3` | Radius 3 SDK. Profiles `zone66`, `zone266`, `point1000`, `point2000`. Up to three channels, many-to-many. |
| `point` | Point SDK. Short-range 1:1 proximity, full duplex over two frequency channels. |
| `sda` | SDA. Apps carry an expiry window. |

Do not attempt to create an app for a product that is not in this array — the app will be rejected or its
pages will be inaccessible.

## Step 3 — create the application

`createApp` (`POST /v2/apps`). The fields that matter:

- `sdk_type` — one of `legacy`, `radius`, `radius3`, `point`, `sda`. Must be in the account's `products`.
- `profiles` — the tone profiles to enable, valid only for the chosen `sdk_type` per the table above.
- `tone_transaction_type` — `payment`, `identification`, `confirmation` or `device_pairing`. **This is
  inherited by every tone created with this app's API tokens and cannot be overridden per request.** If the
  account has several transaction types available, this choice is the one that matters most.
- `sda_expiration_delta_seconds` — required only when `sdk_type` is `sda`. An integer between **1 and
  172800** (48 hours). Values outside that range are rejected.

Use `listApps` and `getApp` to confirm what was created. `updateApp` (`PUT /v2/apps/{id}`) edits it;
`deleteApp` removes it.

## Step 4 — mint the API token (server-side tone generation)

`createApiToken` (`POST /v2/apps/{id}/api-tokens`) returns the credential your backend presents to
`https://tones.lisnr.com/`. List existing ones with `listApiTokens` (`GET /v2/apps/{id}/api-tokens`, which
accepts a `limit` query parameter) and revoke with `deleteApiToken`.

**Handling rules, published by LISNR:** an API Token grants the right to make requests on behalf of the
application and is as sensitive as a password. Do not share or distribute it to untrusted parties. Do not
include it in calls made from a website or any other publicly available project. Keep it server-side, and
never echo the response of this operation into a log or an agent transcript.

Once you hold the token, hand off to `skills/lisnr-generate-tone.md`.

## Step 5 — mint the SDK token (on-device transmit/receive)

`createSdkToken` (`POST /v2/apps/{id}/sdk-tokens`) returns the credential that initializes the Radius object
inside the device SDK. `listSdkTokens` and `deleteSdkToken` manage the set.

The constraint that catches people out: to demodulate an **AES-256 encrypted** tone, the receiving
application must have initialized with an SDK token issued from the **same account** as the API token that
created the tone. Cross-account encrypted tones will not decrypt.

## Step 6 — get the SDK build

`listSdkReleases` (`GET /v2/sdk-releases`) returns the downloadable releases, filtered to what the account
is licensed for. It accepts `platform` and `limit` query parameters. Each release carries a `version`, an
`sdk` download URL, a `sample_app` download URL and a `documentation_url`.

Platforms: Android (7.0+), iOS (15.1+), React Native (Android 7.0+ / iOS 15.1+), Linux, Windows.

There is no public package to install — LISNR ships nothing to npm, PyPI, Maven Central, RubyGems or
crates.io. This endpoint and the Portal's per-platform Developer Resources pages are the only distribution.

## Conventions that apply throughout

- **Response envelope.** Every Portal API response wraps its payload on a top-level `result` property.
  Collections put an array there, single reads an object, errors an object with a `message` string.
- **Pagination.** Cursor style: `limit` for page size, `starting_after` set to the id of the last item on
  the previous page. Page until a short page comes back — there is no total-count or next-cursor field.
- **No idempotency.** No idempotency key exists on this surface. `createApiToken`, `createSdkToken`,
  `createApp` and `createUser` all mint new state on every accepted call. A blind retry after an ambiguous
  response will create a duplicate — list first, then reconcile.
- **No request id.** Nothing correlates a call to a server-side trace, so capture your own context before
  contacting support.
- **Versioning.** Current surface is `/v2/`. A legacy `/api/v1/` prefix is still configured in the Portal
  client but no observed operation uses it — do not build against it.
