# PulseSend webhooks

Create endpoints in the dashboard (Webhooks). The URL must be HTTPS on a public address. Each endpoint has its own signing secret, shown on its page.

## Payload

```json
{
    "id": "evt_01m5…",
    "type": "email.delivered",
    "created_at": "2026-10-18T14:03:22Z",
    "data": {
        "email_id": "em_01m5…",
        "status": "delivered"
    }
}
```

`email.failed` also carries `data.reason`. `webhook.test` is sent by the dashboard's "Send test" button.

## Delivery

- Answer with any `2xx` within 10 seconds. Do the work in a background job; acknowledge first.
- Anything else is retried: 7 attempts in total over about 24 hours.
- An endpoint with no successful delivery for 3 days is disabled and its owner is emailed. Re-enable it in the dashboard.
- Deliveries can arrive more than once and out of order: deduplicate by `id`, and don't let an older status overwrite a newer one.

## Signature

Header: `X-PulseSend-Signature: t=1760796202,v1=5f2b…`

`v1` is the hex HMAC-SHA256 of the string `"{t}.{raw body}"` with the endpoint secret as key. To verify:

1. Read the **raw** body bytes, before any JSON parsing or re-serialisation.
2. Split the header on `,` and `=` to get `t` and `v1`.
3. Reject if `|now - t| > 300` seconds (replay protection).
4. Compute the HMAC and compare with a constant-time function.

### Node.js (Express)

```js
import crypto from 'node:crypto';
import express from 'express';

const app = express();

app.post(
    '/webhooks/pulsesend',
    express.raw({ type: 'application/json' }),
    (req, res) => {
        const header = req.get('X-PulseSend-Signature') ?? '';
        const parts = Object.fromEntries(
            header.split(',').map((p) => p.split('=')),
        );
        const t = Number(parts.t);
        if (!t || Math.abs(Date.now() / 1000 - t) > 300)
            return res.sendStatus(400);

        const expected = crypto
            .createHmac('sha256', process.env.PULSESEND_WEBHOOK_SECRET)
            .update(`${t}.${req.body}`)
            .digest('hex');
        const given = Buffer.from(parts.v1 ?? '', 'utf8');
        const wanted = Buffer.from(expected, 'utf8');
        if (
            given.length !== wanted.length ||
            !crypto.timingSafeEqual(given, wanted)
        ) {
            return res.sendStatus(400);
        }

        const event = JSON.parse(req.body);
        // queue(event) — dedupe on event.id
        res.sendStatus(200);
    },
);
```

### Laravel

```php
use Illuminate\Http\Request;

Route::post('/webhooks/pulsesend', function (Request $request) {
    parse_str(str_replace(',', '&', (string) $request->header('X-PulseSend-Signature')), $parts);
    $t = (int) ($parts['t'] ?? 0);
    abort_if(! $t || abs(time() - $t) > 300, 400);

    $expected = hash_hmac('sha256', $t.'.'.$request->getContent(), config('services.pulsesend.webhook_secret'));
    abort_unless(hash_equals($expected, (string) ($parts['v1'] ?? '')), 400);

    // dispatch(new HandlePulseSendEvent($request->json()->all())); — dedupe on id
    return response()->noContent();
})->withoutMiddleware(\Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class);
```

### Python (Flask)

```python
import hashlib, hmac, os, time
from flask import Flask, abort, request

app = Flask(__name__)

@app.post("/webhooks/pulsesend")
def pulsesend_webhook():
    parts = dict(p.split("=", 1) for p in request.headers.get("X-PulseSend-Signature", "").split(",") if "=" in p)
    t = int(parts.get("t", 0))
    if not t or abs(time.time() - t) > 300:
        abort(400)
    body = request.get_data()  # raw bytes
    expected = hmac.new(os.environ["PULSESEND_WEBHOOK_SECRET"].encode(), f"{t}.".encode() + body, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, parts.get("v1", "")):
        abort(400)
    event = request.get_json()
    # enqueue(event) — dedupe on event["id"]
    return "", 200
```

### Go

```go
func verify(r *http.Request, body []byte, secret string) bool {
    parts := map[string]string{}
    for _, p := range strings.Split(r.Header.Get("X-PulseSend-Signature"), ",") {
        if k, v, ok := strings.Cut(p, "="); ok {
            parts[k] = v
        }
    }
    t, err := strconv.ParseInt(parts["t"], 10, 64)
    if err != nil || math.Abs(float64(time.Now().Unix()-t)) > 300 {
        return false
    }
    mac := hmac.New(sha256.New, []byte(secret))
    fmt.Fprintf(mac, "%d.", t)
    mac.Write(body)
    return hmac.Equal([]byte(hex.EncodeToString(mac.Sum(nil))), []byte(parts["v1"]))
}
```

## Common mistakes

- Verifying a re-serialised body (`JSON.stringify(req.body)`): key order and spacing change, the signature never matches. Use the raw bytes.
- CSRF middleware rejecting the POST (Laravel, Rails, Django): exempt the route.
- Doing slow work before answering: past 10 seconds the delivery counts as failed and is retried.
