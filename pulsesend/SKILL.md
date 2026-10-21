---
name: pulsesend
description: Integrate the PulseSend transactional email API (OTP codes, receipts, notifications) into an application. Use when code sends email through PulseSend, calls pulse-send.jiordiviera.me/api/v1, handles PulseSend webhooks or their X-PulseSend-Signature header, or deals with ps_test_/ps_live_ keys and PulseSend error codes.
---

# PulseSend API

PulseSend sends transactional email over a small REST API. Base URL: `https://pulse-send.jiordiviera.me/api/v1`. Docs: https://docs.pulse-send.jiordiviera.me.

## Rules that matter most

- **Keys stay on the server.** Read the key from the environment (`PULSESEND_API_KEY`), never hardcode it, never ship it in a browser or mobile bundle.
- **Use a `ps_test_` key in development and CI.** Test emails are recorded but never delivered, cost nothing, skip domain verification and never count toward limits. `ps_live_` keys deliver and need the `from` address on a verified domain.
- **Send an `Idempotency-Key` on every send** that could be retried (job queues, HTTP retries). Use an identifier of the intent (order id, user id + purpose), not of the attempt.
- **`202` means accepted, not delivered.** The email starts `queued`; follow it with webhooks or `GET /emails/{id}`.
- **Verify webhook signatures** on the raw request body before parsing JSON (see `references/webhooks.md`).

## Send an email

```bash
curl https://pulse-send.jiordiviera.me/api/v1/emails \
  -H "Authorization: Bearer $PULSESEND_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: signup-otp-8f3a2c" \
  -d '{
    "from": "Acme <hello@acme.com>",
    "to": "jane@example.com",
    "subject": "Your verification code",
    "html": "<p>Your code is <strong>482913</strong></p>",
    "text": "Your code is 482913"
  }'
```

Fields: `from` (bare address or `Name <address>`), `to` (string or list), `cc`, `bcc` (lists), `reply_to`, `subject`, `html` and/or `text` (at least one), `tags` (up to 10), `headers` (object, up to 20). Up to 50 recipients in total; each recipient costs one credit (live only).

The response is the email object: `id` (`em_…`), `status` (`queued`), `test_mode`, and the fields sent.

Framework snippets (Node, Laravel, Python, PHP, Go): `references/examples.md`.

## Read emails

- `GET /emails?limit=20&cursor=…` returns `{ "object": "list", "data": [...], "has_more": bool, "next_cursor": "…" }`. `limit` is 1–100. Pass `next_cursor` as `cursor` for the next page.
- `GET /emails/{id}` returns one email with its status. After 30 days `html`, `text` and `headers` come back `null`; subject, recipients, status and events stay.

Statuses: `queued` → `sent` → `delivered`; also `delivery_delayed`, `bounced`, `complained`, `failed` (credits refunded), `canceled`.

## Errors

Every error is `{ "error": { "type", "code", "message", "param", "doc_url" } }`. Branch on `code`, never on `message`. Each response carries an `X-Request-Id` header worth logging. Full table and handling advice: `references/errors.md`.

The ones to handle explicitly:

| Status | `code`                                 | What to do                                                      |
| ------ | -------------------------------------- | --------------------------------------------------------------- |
| 401    | `invalid_api_key`, `missing_api_key`   | Configuration problem, don't retry                              |
| 422    | `domain_not_verified`                  | Verify the sender domain in the dashboard (`param`: `from`)     |
| 403    | `team_paused`                          | Sending paused for bounces/complaints, don't retry              |
| 402    | `insufficient_credits`                 | Buy credits; don't retry in a loop                              |
| 422    | `invalid_parameter`, `missing_body`, … | Fix the request; `param` names the field                        |
| 429    | `rate_limited`                         | Wait `Retry-After` seconds, retry with the same idempotency key |
| 429    | `daily_limit_reached`                  | Free plan cap (100/day UTC); retry tomorrow or buy credits      |
| 5xx    | `internal_error`                       | Retry with backoff and the same idempotency key                 |

## Limits

10 requests/second per key, 50 recipients per email, message up to 10 MB, idempotency keys kept 24 h. Free plan (no purchased credits, no Pro): 100 live recipients per day and 1 sending domain. Details: `references/limits.md`.

## Webhooks

PulseSend POSTs `email.sent`, `email.delivered`, `email.delivery_delayed`, `email.bounced`, `email.complained`, `email.failed`, `email.canceled` (and `webhook.test`) to your HTTPS endpoint, signed with `X-PulseSend-Signature: t=<unix>,v1=<hex>`. Verification code in several languages: `references/webhooks.md`.

## Testing an integration

1. Create a `ps_test_` key in the dashboard (API keys).
2. Send with any `from` address; expect `202` and `test_mode: true`.
3. Use the CLI to inspect: `pulsesend emails list` (see the `pulsesend-cli` skill).
4. In unit tests, mock the HTTP call; don't hit the API.
