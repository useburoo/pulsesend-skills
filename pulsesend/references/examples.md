# PulseSend send examples

All read the key from `PULSESEND_API_KEY` and send an `Idempotency-Key`.

## Node.js (fetch)

```js
export async function sendEmail({ to, subject, html, text, idempotencyKey }) {
    const res = await fetch('https://pulse-send.jiordiviera.me/api/v1/emails', {
        method: 'POST',
        headers: {
            Authorization: `Bearer ${process.env.PULSESEND_API_KEY}`,
            'Content-Type': 'application/json',
            'Idempotency-Key': idempotencyKey,
        },
        body: JSON.stringify({
            from: 'Acme <hello@acme.com>',
            to,
            subject,
            html,
            text,
        }),
    });
    const body = await res.json();
    if (!res.ok) {
        const err = new Error(body.error?.message ?? `PulseSend ${res.status}`);
        err.code = body.error?.code;
        err.status = res.status;
        throw err;
    }
    return body; // { id: "em_…", status: "queued", test_mode, … }
}
```

## Laravel

`config/services.php`:

```php
'pulsesend' => [
    'key' => env('PULSESEND_API_KEY'),
    'webhook_secret' => env('PULSESEND_WEBHOOK_SECRET'),
],
```

```php
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Http\Client\RequestException;
use Illuminate\Support\Facades\Http;

$email = Http::withToken(config('services.pulsesend.key'))
    ->withHeaders(['Idempotency-Key' => "order-{$order->id}-receipt"])
    ->retry(3, 1000, fn ($e) => $e instanceof ConnectionException
        || ($e instanceof RequestException && in_array($e->response->status(), [429, 500, 502, 503], true)
            && $e->response->json('error.code') !== 'daily_limit_reached'))
    ->post('https://pulse-send.jiordiviera.me/api/v1/emails', [
        'from' => 'Acme <billing@acme.com>',
        'to' => $order->customer_email,
        'subject' => "Receipt for order #{$order->number}",
        'html' => view('emails.receipt', ['order' => $order])->render(),
    ])
    ->throw()
    ->json();
```

Send from a queued job, not inside the HTTP request.

## Python (requests)

```python
import os, requests

def send_email(to, subject, html, idempotency_key):
    r = requests.post(
        "https://pulse-send.jiordiviera.me/api/v1/emails",
        headers={
            "Authorization": f"Bearer {os.environ['PULSESEND_API_KEY']}",
            "Idempotency-Key": idempotency_key,
        },
        json={"from": "Acme <hello@acme.com>", "to": to, "subject": subject, "html": html},
        timeout=10,
    )
    if r.status_code >= 400:
        err = r.json().get("error", {})
        raise RuntimeError(f"{err.get('code')}: {err.get('message')}")
    return r.json()
```

## PHP (no framework)

```php
$ch = curl_init('https://pulse-send.jiordiviera.me/api/v1/emails');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_TIMEOUT => 10,
    CURLOPT_HTTPHEADER => [
        'Authorization: Bearer '.getenv('PULSESEND_API_KEY'),
        'Content-Type: application/json',
        'Idempotency-Key: signup-'.$userId,
    ],
    CURLOPT_POSTFIELDS => json_encode([
        'from' => 'Acme <hello@acme.com>',
        'to' => $email,
        'subject' => 'Welcome',
        'text' => 'Welcome to Acme.',
    ]),
]);
$body = json_decode(curl_exec($ch), true);
$status = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
```

## Go

```go
payload, _ := json.Marshal(map[string]any{
    "from":    "Acme <hello@acme.com>",
    "to":      "jane@example.com",
    "subject": "Your code",
    "text":    "Your code is 482913",
})
req, _ := http.NewRequest("POST", "https://pulse-send.jiordiviera.me/api/v1/emails", bytes.NewReader(payload))
req.Header.Set("Authorization", "Bearer "+os.Getenv("PULSESEND_API_KEY"))
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Idempotency-Key", "otp-"+userID)
res, err := (&http.Client{Timeout: 10 * time.Second}).Do(req)
```

## Shell / CI

Prefer the CLI (`pulsesend send … --json`) in scripts; see the `pulsesend-cli` skill.
