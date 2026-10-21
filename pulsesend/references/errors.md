# PulseSend error codes

Shape:

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "invalid_parameter",
    "message": "The to field must be a valid email address.",
    "param": "to",
    "doc_url": "https://docs.pulse-send.jiordiviera.me/errors#invalid_parameter"
  }
}
```

Branch on `code`. `param` names the offending field when there is one. `doc_url` points to the explanation. Log the `X-Request-Id` response header: support needs it.

Statuses by `type`: `authentication_error` 401, `insufficient_credits` 402, `permission_error` 403 (`missing_scope`, `team_paused`), `not_found` 404, `idempotency_error` 409, `validation_error` 422 (including `domain_not_verified`, `missing_body`, `too_many_recipients`, `message_too_large`, `all_recipients_suppressed`), `rate_limit_error` 429 (`rate_limited`, `daily_limit_reached`), `api_error` 500.

| `code`                      | Meaning                                                    | Retry?                                   |
| --------------------------- | ---------------------------------------------------------- | ---------------------------------------- |
| `missing_api_key`           | No `Authorization: Bearer …` header                        | No, fix config                           |
| `invalid_api_key`           | Unknown or revoked key                                     | No, fix config                           |
| `missing_scope`             | The key lacks the scope for this route                     | No, create a key with the scope          |
| `insufficient_credits`      | Not enough credits for all recipients                      | No, buy credits                          |
| `team_paused`               | Sending paused (bounce rate > 4 % or complaints > 0.08 %)  | No, contact support after cleaning lists |
| `resource_missing`          | Unknown id (`GET /emails/{id}`)                            | No                                       |
| `idempotency_key_reused`    | Same `Idempotency-Key` with a different body               | No, use a new key for a new intent       |
| `idempotency_in_progress`   | Same key while the first request still runs                | Yes, after a short wait                  |
| `idempotency_key_too_long`  | Key over 255 characters                                    | No                                       |
| `invalid_parameter`         | A field is invalid; see `param`                            | No, fix the request                      |
| `missing_body`              | Neither `html` nor `text`                                  | No                                       |
| `too_many_recipients`       | More than 50 across `to`, `cc`, `bcc`                      | No, split the email                      |
| `message_too_large`         | Over 10 MB                                                 | No                                       |
| `domain_not_verified`       | Live key and `from` not on a verified domain               | No, verify the domain                    |
| `all_recipients_suppressed` | Every recipient bounced or complained before               | No                                       |
| `rate_limited`              | Over 10 requests/second for this key                       | Yes, after `Retry-After` seconds         |
| `daily_limit_reached`       | Free plan: 100 live recipients per UTC day                 | Tomorrow, or after buying credits        |
| `internal_error`            | Server error                                               | Yes, with backoff                        |

## Retry policy

Retry only `rate_limited`, `idempotency_in_progress`, `5xx` and network errors. Always resend the **same** `Idempotency-Key` so a request that actually succeeded is not sent twice. `5xx` responses are not stored against the idempotency key, so the retry runs again.

Example backoff: 1 s, 2 s, 4 s, 8 s, then give up and surface the error; honour `Retry-After` when present.
