---
name: Generate a LISNR ultrasonic tone
description: >-
  Turn a data payload into an inaudible LISNR audio tone and download the resulting file. Covers picking a
  tone profile the account is licensed for, encoding and size-checking the payload, optional AES-256
  encryption and ToneLock pairing, and retrieving the signed URL before it expires.
api: openapi/lisnr-tones-openapi-original.json
operations:
- createTone
supporting_api: openapi/lisnr-portal-openapi-derived.json
supporting_operations:
- listAccounts
- getApp
- listApiTokens
generated: '2026-07-19'
method: generated
source: openapi/lisnr-tones-openapi-original.json
---

# Generate a LISNR ultrasonic tone

LISNR encodes data into inaudible high-frequency audio. This skill produces one tone file from one payload.

`createTone` is the only operation on the Tones Service API. It is `POST https://tones.lisnr.com/`, and it
returns a signed URL rather than audio bytes.

## Before you start

You need an **API Token** minted against a specific LISNR application. Get one from the Portal at
`https://portal.lisnr.com/apps/{app_id}/api-tokens`, or list existing ones with `listApiTokens`.

Two things are inherited from the application the token belongs to, and neither can be overridden on the
request:

- the **transaction type** stamped on the tone (`payment`, `identification`, `confirmation`, `device_pairing`)
- the **SDK generation**, which constrains the tone profiles you may use

If the account has more than one transaction type available, confirm you are using a token from the right
application before calling. Check with `listAccounts` (returns the licensed `products` array) and `getApp`.

Treat the API Token as a password. LISNR documents it as being as sensitive as one: never put it in a
browser, a mobile binary, or any public repository.

## Step 1 — pick a profile the app is licensed for

| SDK generation | Profiles | Byte limit |
| --- | --- | --- |
| `legacy` | `standard`, `compression` | 255 |
| `radius` | `standard2`, `standard2_wideband` | 255 |
| `radius` | `pkab2`, `pkab2_wideband` | 3000 |
| `radius3` | `zone66` | 255 |
| `radius3` | `zone266`, `point1000`, `point2000` | 3000 |

The Tones Service API accepts the eight non-legacy profiles: `zone66`, `zone266`, `point1000`, `point2000`,
`pkab2`, `pkab2_wideband`, `standard2`, `standard2_wideband`.

## Step 2 — encode and size-check the payload

`payload` is a **hexadecimal string representation of bytes**. A hex string has twice as many characters as
it has bytes, so `a1b2c3` is six characters but three bytes. Text must be ASCII before it is hex-encoded;
`hello world` becomes `68656c6c6f20776f726c64`.

Size-check against the profile limit in the table above, and if `encrypt` is `true`, subtract the encryption
overhead first: **2 bytes for `zone66`, 4 bytes for every other profile**. Exceeding the limit returns 400.

## Step 3 — decide the transport options

- `format` — `wav` (default) or `mp3`. All tones are 24-bit.
- `zip` — compress the result. Defaults to `true`.
- `prependSilence` — prepends 200ms of silence. Set this to `true` if the tone will be broadcast from an
  **iOS browser**; without it the start of the tone is clipped by an iOS limitation.
- `channel` — `0`, `1` or `2` for `zone66` and `point1000`. `zone266` and `point2000` always broadcast on
  channel 1. Channel does not apply to the `standard2` or `pkab2` families.

## Step 4 — decide pairing and encryption

- `tonelockType` — `account_based`, `custom_value` or `time_based`. A receiver cannot demodulate a
  ToneLock-enabled tone unless it holds a matching ToneLock value. With `custom_value` you must also send
  `tonelockValue` as an RFC 4122 UUID (36 characters, 32 hex digits, 4 hyphens).
- `encrypt` — AES-256 encrypts the payload. The receiving application must have initialized its Radius
  object with an **SDK token from the same account** as the API token used here, or it will not decrypt.

## Step 5 — call createTone

```
POST https://tones.lisnr.com/
Authorization: JWT <api-token>
Content-Type: application/json

{
  "profile": "point1000",
  "payload": "68656c6c6f20776f726c64",
  "format": "wav",
  "zip": true,
  "prependSilence": false,
  "encrypt": false
}
```

A 200 returns `{ "url": "..." }`. **The URL is valid for twenty-four hours** — download the file promptly
and store it yourself if you need it longer. There is no read, list or re-fetch operation; a lost URL means
generating the tone again.

## Retry and error handling

`createTone` is **not idempotent**. There is no idempotency key. Every accepted call generates a new tone
artifact and a new URL, and consumes account quota. Retry only after a transport failure where no response
was received — never after a response, however unhelpful.

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 | Invalid request body | Check required fields (`payload`, `profile`), hex encoding, the profile byte limit including encryption overhead, and the ToneLock UUID format. |
| 401 | Unauthorized | The `Authorization` header is missing or the token is invalid. Confirm the literal `JWT ` prefix — it is not `Bearer`. |
| 403 | Forbidden | Token is valid but the app is not entitled. Check `listAccounts` for the licensed `products` and confirm the profile matches the app's SDK generation. |
| 429 | Too many requests | Back off exponentially. LISNR publishes no limit values and sends no `RateLimit` or `Retry-After` headers. |
| 500 | Failed to generate tone | Retry once, then escalate to techsupport@lisnr.com. Remember a successful retry produces a *new* artifact. |

Errors are a flat `{ "message": "..." }` object — not RFC 9457 problem+json, and with no machine-readable
error code. Branch on HTTP status, not on the message string.
