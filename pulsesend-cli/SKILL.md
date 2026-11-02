---
name: pulsesend-cli
description: Use the pulsesend command-line tool to send test or live emails, check an API key, and inspect sent emails and their delivery status. Use when asked to send an email from the terminal or a CI job with PulseSend, verify a PulseSend integration end to end, debug why an email bounced, or script PulseSend with --json.
---

# pulsesend CLI

A single static binary that talks to the PulseSend API. Docs: https://docs.pulse-send.jiordiviera.me/cli

## Install

```bash
curl -fsSL https://pulse-send.jiordiviera.me/install.sh | sh     # macOS, Linux → ~/.local/bin
brew install useburoo/tap/pulsesend                          # Homebrew
```

Windows: `pulsesend-<version>-x86_64-pc-windows-msvc.zip` from https://github.com/useburoo/pulsesend-cli/releases/latest.

Check with `pulsesend --version`.

## Authentication

Key resolution order: `--api-key` flag, then `PULSESEND_API_KEY`, then the key saved by `pulsesend login` (`~/.config/pulsesend/config.json`, mode 0600).

- **As an agent, never run `pulsesend login`**: it prompts for a hidden key on the terminal. Use `PULSESEND_API_KEY` from the environment instead, and never print the key.
- `pulsesend whoami` checks the key and shows test or live mode. Run it first.
- Prefer a `ps_test_` key unless the user explicitly wants a real delivery: test emails are recorded, never delivered, free.

## Commands

```bash
pulsesend whoami
pulsesend send --from "Acme <hello@acme.com>" --to jane@example.com \
  --subject "Your code is 482910" --text "Your code is 482910"
pulsesend emails list [--limit 50] [--cursor <next_cursor>]
pulsesend emails get em_01m5…
```

`send` options: `--to`, `--cc`, `--bcc` (repeat or comma-separate, 50 recipients max), `--html`, `--html-file`, `--text`, `--text-file` (files accept `-` for stdin; at least one body), `--reply-to`, `--tag` (≤10), `--header "Name: value"` (≤20), `--attach <file>` (≤20, sent as base64; the file name is kept, the path is not), `--idempotency-key`, `--wait`, `--wait-timeout <seconds>` (default 60).

`--wait` polls every second until the email is delivered; exits 1 if it bounces, is marked as spam, fails, or times out. With a test key it stops as soon as the email is accepted.

## Scripting

- Add `--json` to any command: on success stdout is exactly the API response and stderr stays silent. Parse it with `jq`.
- Exit code is `0` on success, `1` on any error (API error, network, `--wait` not delivered).
- In a pipe or CI the CLI prints plain lines; `NO_COLOR=1` removes colors.
- `PULSESEND_BASE_URL` points the CLI at another API (for example a local PulseSend).

```bash
id=$(pulsesend send --json --from hello@acme.com --to qa@acme.com \
  --subject "Smoke test" --text "ok" --idempotency-key "ci-$GITHUB_RUN_ID" | jq -r .id)
pulsesend emails get "$id" --json | jq -r .status

# Bounced emails on the last page
pulsesend emails list --limit 100 --json | jq -r '.data[] | select(.status == "bounced") | .id'
```

## Reading errors

Errors always go to stderr (even with `--json`, stdout stays empty) and show the API message, HTTP status, `code`, `field` (when one is invalid), the request id, a hint for common codes and a docs link. Check the exit code, then read stderr.

| `code`                                | Likely fix                                                      |
| ------------------------------------- | --------------------------------------------------------------- |
| `invalid_api_key` / `missing_api_key` | Set `PULSESEND_API_KEY` to a valid key                          |
| `domain_not_verified`                 | Live key: the `--from` domain must be verified in the dashboard |
| `insufficient_credits`                | Buy credits on the dashboard's billing page                     |
| `daily_limit_reached`                 | Free plan: 100 live recipients per UTC day                      |
| `team_paused`                         | Too many bounces or complaints; the user must contact support   |
| `rate_limited`                        | More than 10 requests/second; wait and retry                    |

## Debugging a delivery

1. `pulsesend emails get <id>`: status, recipients, mode and credits charged. The full event timeline is on the email's page in the dashboard (`/emails/<id>`).
2. `bounced` or `complained`: the address is now suppressed; future sends to it are dropped (`all_recipients_suppressed` if it was the only one).
3. Stuck in `queued` with a live key: check the dashboard's status banner (team paused?) before retrying.

Not available in the CLI yet: domains, API keys and webhooks. They are managed in the dashboard (https://pulse-send.jiordiviera.me).
