# Configuration reference

Generate a complete detection configuration with `rinlock init-config`; validate
with `rinlock check --config FILE`. Loading starts from the built-in defaults,
then overlays supplied JSON fields. A supplied array replaces the default array.
Unknown fields, oversized input and trailing JSON documents are rejected.

`rinlock policy --config FILE` activates detection changes with a new revision.
Storage and collection-source settings require restart. The startup file is
authoritative on daemon restart; runtime activation does not overwrite it.
Prevention has a [separate format and root control path](prevention.md).

## Detection and collection

| Field | Default / meaning |
| --- | --- |
| `web_processes` | Web-server command names used for shell-parent detection |
| `shells` | Shell command names used for shell execution/egress detection |
| `credential_paths` | Sensitive path patterns; see the shipped example |
| `credential_readers` | Exact executable-path allowlist for credential detection |
| `allowed_cidrs` | Empty; matching destinations skip built-in network findings |
| `suspicious_ports` | 4444, 5555, 6667, 9001 |
| `notify_score` | 70; threshold for notification eligibility |
| `watch_files` | `/etc/passwd`, `/etc/group`, `/etc/shadow`, `/etc/sudoers` |
| `watch_users` | true; diff local passwd records for account changes |
| `auth_journal` | false; opt in to SSH/PAM journal observations |
| `rules` | Custom event matching rules |

Custom rules use `id`, `description`, `kinds`, `score`, and optional `processes`,
`parents`, `paths`, `users`, `outcomes`, `changes`, `container_only`, `disabled`.
Fields combine with AND; entries within a list combine with OR. Empty selectors
are unrestricted. At least one configured `changes` field must be present in the
event. Kinds and scores are required; built-in rule IDs are reserved.

Custom `paths` use Go `path.Match`: `*` cannot cross `/`, and `**` is not recursive.
Built-in `credential_paths` additionally support a leading `*/` across directory
prefixes. `processes`/`parents` match spoofable Linux `comm` values, not cryptographic
identity. `users` matches account/login usernames, not a universal UID-to-name map.

Kinds: `exec`, `file_open`, `connect`, `credential`, `file_change`, `user_created`,
`user_deleted`, `user_changed`, `login`, `collector_gap`, `prevention`,
`prevention_policy`. Outcomes: `success`, `failed`, `in_progress`, `observed`,
`blocked`, `would_block`, `allowed`, `would_allow`, `activated`.

Change fields: `exists`, `mode`, `size`, `uid`, `gid`, `inode`, `device`, `links`,
`mtime`, `ctime`, `symlink_target`, `home`, `shell`. An open event does not prove
file contents were read; a connect attempt does not prove data was transmitted.

## Correlation

| `correlation` field | Default | Meaning |
| --- | --- | --- |
| `window_seconds` | 300 | Fixed correlation window |
| `login_failures` | 5 | Repeated-failure incident threshold |
| `disabled` | false | Disable incident correlation |

Web-shell chains require observed stable process identity/ancestry. Unknown
identity does not create guessed relationships. Policy revisions form separate
correlation boundaries. Incident evidence retains the first 100 event IDs; the
observation count can continue increasing.

## Notifications

Configure webhook, Slack, ntfy and SMTP destinations with the separate startup
file `serve --notification-config FILE`. See [notification setup](notifications.md)
for routing, SMTP TLS/authentication, mobile apps and systemd examples. Existing
`RINLOCK_WEBHOOK_URL` and optional `RINLOCK_WEBHOOK_TOKEN` deployments remain
supported without this file. Credentials belong in the environment, not rules.

| `notifications` field | Default | Meaning |
| --- | --- | --- |
| `cooldown_seconds` | 300 | Group repeated eligible findings by subject |
| `muted_rules` | [] | Findings excluded from enqueue eligibility |
| `mute_until` | unset | Do not enqueue new notifications before this RFC3339 time |
| `max_attempts` | 10 | Maximum reserved attempts per queued delivery |
| `max_pending` | 10000 | Maximum pending deliveries across all destinations |
| `rate_per_minute` | 60 | Reservations per destination per fixed UTC minute |
| `history_days` | 30 | Completed/failed delivery history retention |

Higher eligible scores can bypass a lower-score cooldown. Muting affects new
enqueue decisions; `rinlock notifications --pause SECONDS` pauses existing backlog
delivery, and `--pause 0` resumes it. Retryable responses include 408, 429, 5xx and
transport errors. SMTP 4xx/transport failures retry; 5xx and missing STARTTLS
fail immediately. Redirects are not followed; other 3xx/4xx are terminal failures.
Retry-After is bounded to 24 hours. The durable outbox provides retry/deduplication
support, not exactly-once delivery.

## Storage

| `storage` field | Default | Meaning |
| --- | --- | --- |
| `retention_days` | 30 | Age limit for unprotected events |
| `max_events` | 1000000 | Target maximum retained event count |
| `max_database_bytes` | 1073741824 | Database high-water admission threshold |
| `min_free_bytes` | 268435456 | Required available filesystem space |
| `maintenance_seconds` | 60 | Cleanup interval |

Pending notification and retained incident evidence can keep events past these
retention targets. Open/acknowledged incidents do not expire automatically.
Capacity checks are admission checks, not an OS-enforced disk quota; transactions
can grow the file past a threshold. Monitor health and use filesystem limits where
a hard quota is required. Cleanup does not shrink the database file; use offline
compaction and verified replacement as described in [operations](stage-3.md).

## Vulnerability scanner configuration

The optional `serve --vulnerability-config FILE` is a separate startup-only format.
See [vulnerabilities.md](vulnerabilities.md) for every field, coverage limitations,
KEV updates and notification scores. It is not part of hot-reloaded detection policy.
