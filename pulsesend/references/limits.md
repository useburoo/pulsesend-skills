# PulseSend limits

| Limit                    | Value                                    |
| ------------------------ | ---------------------------------------- |
| Requests                 | 10 per second per API key                |
| Recipients per email     | 50 across `to`, `cc` and `bcc`           |
| Tags per email           | 10, each up to 256 characters            |
| Custom headers per email | 20                                       |
| Subject                  | 998 characters                           |
| Message size             | 10 MB (subject, body, headers)           |
| Active API keys          | 20 per team                              |
| Sending domains          | 10 per team (1 on the free plan)         |
| Idempotency keys         | Kept 24 hours                            |
| Email content            | Kept 30 days (`html`, `text`, `headers`) |
| Webhook delivery log     | Kept 30 days                             |

## Free plan

A team with no purchased credits left and no Pro subscription is on the free plan: 100 live recipients per day (reset 00:00 UTC, `429 daily_limit_reached` beyond) and 1 sending domain. Test emails never count.

## Credits

Every team gets 3,000 free credits a month, used first. One credit = one live recipient. Failed sends are refunded. Suppressed recipients are dropped and not charged.

## Reputation

Bounces must stay under 4 % and complaints under 0.08 % (live emails, last 30 days, from 100 known outcomes). Over either, the whole team is paused (`403 team_paused`). Send only to people who asked, and stop sending to addresses that bounce.

## Batching

There is no batch endpoint. For many emails, send one request per email (or per group of up to 50 recipients who should see each other), stay under 10 requests/second per key, and give each an idempotency key.
