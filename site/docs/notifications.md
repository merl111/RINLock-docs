# Notification destinations

RINLock supports generic HTTP webhooks, Slack incoming webhooks, ntfy mobile push,
and SMTP email. Destinations are opt-in. Up to 16 destinations can receive the same
alert, each with independent attempts, backoff and a persistent delivery record.
A slow destination does not block another destination's worker or event ingestion.

## Quick setup

Copy [notifications-example.json](../configs/notifications-example.json), remove
unwanted destinations, and supply the referenced environment variables. URLs and
authentication secrets are read from the daemon's environment at startup; the
JSON stores their variable names. Do not commit live credentials or secret URLs.

```sh
rinlock destinations --config notifications.json
rinlock serve --config rules.json --notification-config notifications.json
```

The first command validates JSON and field constraints locally. It does not read
credential values, connect to providers, or require a running daemon. `serve`
resolves every configured variable and refuses invalid/missing values before
starting collection. Destination configuration and credentials require a daemon
restart; `rinlock policy` reloads detection policy only.

Send an explicit test to one destination, using the same environment as the daemon:

```sh
rinlock destinations --config notifications.json --action test --destination phone
```

**This sends a real message.** It is labelled `notification_test`, with score 90
and an explanation that no incident was detected. It bypasses routing, mutes,
pause, cooldown, the database and automatic retries. Other destinations' secrets
are not required for this test. A zero exit status means the provider accepted
the message, not that a device displayed it or someone read it.

## Common destination fields

| Field | Meaning |
| --- | --- |
| `id` | Stable unique ID; 1–64 letters, digits, underscores or hyphens, starting with a letter/digit |
| `type` | `webhook`, `slack`, `ntfy`, or `smtp` |
| `min_score` | Optional 0–100 minimum; defaults to 0 but cannot lower global `notify_score` |
| `rules` | Optional rule-ID allowlist, including correlation rule IDs |
| `kinds` | Optional event-kind allowlist, such as `exec`, `prevention`, `vulnerability` |
| `url_env` | HTTP types: environment variable containing the destination URL |
| `token_env` | Optional bearer token for webhook/ntfy; omit when unused |
| `topic` | Required ntfy topic, with the same character/length limits as `id` |
| `smtp` | Required SMTP settings object, described below |

Rule and kind filters allow at most 64 entries each. Lists use OR; different
filters combine with AND. At least one unmuted event finding or attached incident
must satisfy both the rule filter and `max(notify_score, min_score)`. An unrelated
higher-scoring finding does not satisfy a selected rule's threshold. `kinds`
filters the triggering event, even when eligibility comes from an incident.
Empty lists allow all values. No matching destination produces an `unrouted`
notification decision; evidence is still stored.

For example, restrict a destination to shell detection on execution events:

```json
{
  "id": "shell-alerts",
  "type": "ntfy",
  "url_env": "RINLOCK_NTFY_URL",
  "topic": "rinlock-shells",
  "min_score": 90,
  "rules": ["web-shell"],
  "kinds": ["exec"]
}
```

Use actual rule IDs from your policy/event findings. Rule filters accept IDs up to 256 bytes without NUL or line breaks; kind filters
use letters, digits, underscores or hyphens, up to 64 characters.

## Generic webhook

The adapter POSTs JSON to `url_env`, with `Content-Type: application/json`, a stable
`Idempotency-Key`, and optional `Authorization: Bearer …`. The payload retains the
existing top-level event fields and optional `incidents` array. It includes full
event details; choose recipients with appropriate access to that evidence.

HTTP and HTTPS are supported for local/self-hosted receivers; use HTTPS when
traffic crosses an untrusted network. URL user/password credentials and fragments
are rejected. Redirects are never followed, so secrets are not forwarded to a
redirect target. Response bodies and credential values are excluded from recorded
transport failures.

## Slack

Create a Slack app with Incoming Webhooks enabled, install it to the intended
workspace/channel, then place the generated HTTPS webhook URL in the environment
variable named by `url_env`. The URL is a secret. Omit `token_env`; channel and
identity come from the Slack webhook configuration.

Messages contain severity, host, process, event ID, path/connection when present,
and bounded finding/incident summaries. Plain-text blocks prevent event text from
invoking Slack mentions. Link/media previews are disabled. See
[Slack's incoming webhook setup](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/).

## ntfy, including self-hosting and iOS

Install an ntfy server or use an existing one. `url_env` must contain the **server
root URL**, such as `https://ntfy.example.org`, without a topic, path prefix or
query. Set `topic` separately. The adapter POSTs ntfy's JSON publishing format to
the server root. Give its token permission to publish to that topic and give
subscribers permission to read it. Omit `token_env` only if the server permits
unauthenticated publishing. Keep security topics private.

On the Android or iOS ntfy app, subscribe to the same server and topic. Scores
90–100 map to priority 5, 70–89 to 4, and lower eligible scores to 3. Messages are
plain text with a shield tag and are bounded to fit the default 4 KiB publish
request limit; inspect the stored event for full evidence. See [publishing](https://docs.ntfy.sh/publish/) and
[mobile subscriptions](https://docs.ntfy.sh/subscribe/phone/).

For prompt **iOS** delivery from a self-hosted server, configure the ntfy server:

```yaml
base-url: "https://ntfy.example.org"
upstream-base-url: "https://ntfy.sh"
```

These settings belong to **ntfy**, not RINLock. The upstream receives a message ID
and hashed topic URL to wake the app through Firebase/Apple push. The app then
fetches the actual message from your server, which must be reachable from the
phone. Without this upstream setting, iOS delivery can be significantly delayed.
See [ntfy's iOS delivery explanation](https://docs.ntfy.sh/config/#ios-instant-notifications).

## SMTP email

```json
{
  "destinations": [
    {
      "id": "email",
      "type": "smtp",
      "min_score": 90,
      "smtp": {
        "address": "smtp.example.org:587",
        "tls": "starttls",
        "from": "rinlock@example.org",
        "to": ["security@example.org", "oncall@example.org"],
        "username_env": "RINLOCK_SMTP_USERNAME",
        "password_env": "RINLOCK_SMTP_PASSWORD"
      }
    }
  ]
}
```

| SMTP field | Configuration |
| --- | --- |
| `address` | Required `host:port`; use `[IPv6]:port` for IPv6 literals |
| `tls` | Required `starttls` for an explicit upgrade, or `tls` for encryption immediately on connection |
| `from` | One plain sender email address, without a display name |
| `to` | 1–20 plain recipient email addresses |
| `username_env`, `password_env` | Set both for SMTP AUTH PLAIN, or omit both for a relay that permits unauthenticated submission over TLS |

For implicit TLS, use your provider's port (commonly 465) and `"tls": "tls"`.
For STARTTLS, use its submission port (commonly 587) and `"tls": "starttls"`.
Authentication occurs only after TLS. The adapter requires TLS 1.2 or later,
verifies the hostname and system certificate trust, and refuses plaintext
fallback. For a private CA, install its certificate in the daemon's trusted
system certificate store. There is no certificate-verification bypass.

Use a provider-supported SMTP password/app password. OAuth/XOAUTH2, attachments,
HTML templates, client certificates and per-recipient tracking are not supported.
The sender must be authorized by your relay; configure the domain's delivery
requirements with your mail provider. All recipients are visible in `To`; use
separate destinations when recipient isolation is needed.

Email includes a UTF-8 plain-text summary, severity/host subject and stable
`Message-ID`. The relay must accept every recipient before DATA is sent. SMTP 4xx
and connection/TLS failures retry within the attempt budget; 5xx and missing
STARTTLS fail immediately. The whole operation has a 10-second timeout. Once the
relay acknowledges DATA, a later QUIT failure does not trigger a resend. Relay
acceptance does not track eventual bounces or inbox delivery. A lost DATA reply
or daemon crash may still cause duplicate mail; `Message-ID` is not a guarantee
that a receiving mail system deduplicates it.

## Packaged systemd deployment

The installer ships configuration and environment examples without enabling them:

```sh
sudo cp /etc/rinlock/notifications-example.json /etc/rinlock/notifications.json
sudo install -m 0600 -o root -g root /etc/rinlock/notifications.env.example /etc/rinlock/notifications.env
# Edit both files locally; remove destinations you do not use.
rinlock destinations --config /etc/rinlock/notifications.json
sudo mkdir -p /etc/systemd/system/rinlock.service.d
sudo cp /etc/rinlock/notifications.conf.example /etc/systemd/system/rinlock.service.d/notifications.conf
sudo systemctl daemon-reload
sudo systemctl restart rinlock.service
```

The [drop-in](../packaging/notifications.conf.example) loads
`/etc/rinlock/notifications.env`, clears the legacy webhook environment source,
and passes `--notification-config`. Keep the environment file root-owned and
mode 0600; systemd reads it before switching to the service account.

If vulnerability scanning is also enabled, combine `--notification-config` and
`--vulnerability-config` in **one** `ExecStart` override. Two separate drop-ins
that each replace `ExecStart` do not combine their flags. Preserve your existing
custom environment files when adapting the example. Service environment values
are not automatically available to commands run from an interactive shell.

## Queue operations and migration

The event, incidents, cooldown and all matching destination records commit in one
transaction. Cooldown is shared per alert before fan-out; it is not per recipient.
Global `mute_until`/`muted_rules` affect new enqueue decisions. Existing queued
records retain their destination and payload when filters or policy change.

- Each destination has its own worker, attempt budget, backoff and persisted
  fixed-UTC-minute rate budget. The configured `rate_per_minute` applies to each
  destination; total traffic can be that rate times the number of destinations.
- Global `max_pending` counts delivery records across all destinations. If the
  whole fan-out will not fit, all matching records become failed with a queue-full
  reason. The event is retained and cooldown still applies.
- Attempts are reserved before network I/O with a 30-second crash lease. HTTP
  requests time out after 5 seconds. HTTP 408/429/5xx and transport failures retry;
  other 3xx/4xx fail permanently. `Retry-After` is capped at 24 hours.
- Delivery is at least once. Each new HTTP destination receives a namespaced,
  stable idempotency key; providers need not honor it. Manual retries preserve
  that key. Completed/failed history is bounded by `history_days`.

```sh
rinlock deliveries --socket /run/rinlock/rinlock.sock --state failed
rinlock deliveries --socket /run/rinlock/rinlock.sock --state pending --before 42
rinlock deliveries --socket /run/rinlock/rinlock.sock --retry 42
rinlock notifications --socket /run/rinlock/rinlock.sock --pause 600
rinlock notifications --socket /run/rinlock/rinlock.sock --pause 0
```

Use the top-level delivery `id` for `--retry`, `--before`, and the API retry path.
`event.id` identifies the underlying evidence; multiple delivery IDs can share it.
`destination` identifies the configured recipient. A failed record can be requeued
only when queue capacity remains; with active destinations configured, its
destination must also be present. Pause affects
all workers; it cannot recall a message already in flight.

Existing `RINLOCK_WEBHOOK_URL`/`RINLOCK_WEBHOOK_TOKEN` deployments continue working
without a notification config. Their old queued records remain readable, with
an empty/omitted destination and the legacy event-based HTTP idempotency key.
Do not combine that legacy URL with `--notification-config`; startup rejects it.

Drain the legacy backlog before switching to named destinations. When starting
with notifications enabled, queued records with IDs absent from the new config
become failed with `destination is not configured`, including legacy records
when switching to named destinations. Restore the original configuration to
retry them. Starting with no notification destinations leaves backlog suspended.

Do not reuse a destination ID for a different recipient: queued records resolve
that ID to its current endpoint and credentials after restart. Renaming an ID
means removing the old destination. Back up the database before upgrading;
older binaries assume one delivery per event and must not write a database after
multi-destination delivery has been used.
