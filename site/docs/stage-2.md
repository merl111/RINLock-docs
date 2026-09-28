# Stage 2: detection and investigation

> Historical implementation record. Pending-work statements below describe that
> stage. See the [current documentation](index.md) and [stage 4 validation](stage-4-validation.json)
> for subsequent features and successful live kernel tests.

Stage 2 adds policy revisions, explicit hot reload and rollback, read-only rule replay, persistent correlation, and notification controls. All collected events remain in the audit log, including events whose notifications are suppressed.

## Policy workflow

```sh
./bin/rinlock init-config > rules.json
./bin/rinlock check --config rules.json
./bin/rinlock policy --socket /path/to/rinlock.sock
./bin/rinlock replay --socket /path/to/rinlock.sock --config rules.json --changed-only
./bin/rinlock policy --socket /path/to/rinlock.sock --config rules.json
./bin/rinlock policy --socket /path/to/rinlock.sock --history
./bin/rinlock policy --socket /path/to/rinlock.sock --rollback FULL_REVISION_ID
```

Each version has a SHA-256 ID derived from its fully defaulted JSON configuration. Version records retain the configuration and latest activation time. Events record `policy_revision`; old records without a revision remain readable. Activation is serialized with ingestion and persists before replacing the engine. A validation or storage failure leaves the active engine intact. The CLI sends the revision it just read, so a concurrent change is rejected instead of overwritten.

`watch_files`, `watch_users`, and `auth_journal` cannot change through hot reload. Restart the daemon to change collection. Applying or rolling back detection settings starts a separate correlation context for that policy version. Returning to a previous version can reuse its still-valid correlation state. Known policy versions survive restart. **Startup still uses `serve --config FILE` (or defaults)**, so keep the desired configuration in that file for the next restart; an API update does not rewrite files or silently override startup arguments. Hot reload is an explicit command, not an automatic file watcher.

Configuration and update bodies accept one JSON object of at most 1 MiB and reject unknown fields. Custom rule IDs cannot collide with built-in or correlation rule IDs. Invalid notification mute IDs are rejected.

## Replay

```sh
./bin/rinlock export --socket /path/to/rinlock.sock > events.ndjson
./bin/rinlock replay --config candidate.json --input events.ndjson > comparison.ndjson
# --input - reads an export from stdin
```

Replay reports original/candidate scores, findings, revision IDs, and a `changed` flag for each event. `--changed-only` filters unchanged results. Direct daemon replay walks history newest first with an upper bound fixed by its first response. Output is streamed; malformed input or a failed page stops with an error, so earlier output may be partial.

Replay evaluates **individual-event rules**. It does not rerun incident correlation, predict notification suppression, overwrite historical scores, or enqueue notifications. It can run entirely offline on an exported file. Missing evidence in old events cannot be reconstructed.

## Correlation

Default configuration:

```json
{
  "correlation": {
    "window_seconds": 300,
    "login_failures": 5,
    "disabled": false
  }
}
```

Two detectors are implemented:

- **`web-shell-chain` (95):** a web-spawned shell subsequently triggers a credential-file or outbound-network finding. An observed descendant exec inherits the origin through stable parent/process identities. Joining requires a process identity and boot ID; numeric PID alone is insufficient. The window is anchored at the original shell observation.
- **`repeated-login-failure` (75):** at least five failed authentication observations for the same host, source, user, and remote IP in the configured window. The window is anchored at the first observation. Missing user, host, or valid IP prevents grouping. These are log observations, not proven distinct attempts: PAM and SSH can report the same attempt separately.

Incidents have their own scores, explanation, revision, time range, evidence IDs, and count. Event scores are unchanged by correlation. Origin and triggering observations are linked; intermediate descendant execs remain in the full event history. The first 100 evidence IDs are retained in the incident; the observation count continues growing. A new trigger stores its incident IDs in the event. Origin events remain immutable and do not receive retroactive links.

State and incidents persist across restart. Correlation uses event timestamps in ingestion order, does not join an observation older than the subject's latest signal, and does not reconstruct missed ancestry or reorder delayed streams. Fixed windows deliberately do not promise detection of every rolling-window burst. Different processes, boots, and policy versions remain separate. Correlation cannot compensate for collector loss or missing auth logs.

```sh
./bin/rinlock incidents --socket /path/to/rinlock.sock --limit 100
./bin/rinlock event --socket /path/to/rinlock.sock --id 42
```

Dashboard **4** opens incidents; arrows/j/k scroll evidence and reasons. Events remain available in tab **1**. Rules refresh with the active revision in tab **2**.

## Notification controls

```json
{
  "notify_score": 70,
  "notifications": {
    "cooldown_seconds": 300,
    "muted_rules": ["watched-file-changed"],
    "mute_until": "2026-09-15T20:00:00Z"
  }
}
```

Omit `mute_until` for no time-based mute; timestamps are RFC 3339. Change these fields through the normal policy workflow.

- An event or associated incident must have an unmuted finding at or above `notify_score`. Muting one finding does not hide another eligible match.
- A cooldown groups the same subject and eligible rule set. Event subjects include host/boot, source, process identity/name, path, user, remote address and port; an associated incident groups updates across descendants. It is not a host-wide rate limit. Higher eligible scores bypass an existing cooldown.
- Cooldown reservations persist and are committed atomically with the event, incidents and outbox. They begin at enqueue time, not successful delivery. Retrying one delivery does not enqueue another copy.
- Each event records `notification`: `disabled`, `below_threshold`, `muted`, `cooldown`, or `queued`. These describe enqueue decisions; `queued` is not a delivery receipt.
- Rule/time mutes suppress **new enqueue operations**. They neither cancel existing jobs nor remove logged evidence.

```sh
./bin/rinlock notifications --socket /path/to/rinlock.sock
./bin/rinlock notifications --socket /path/to/rinlock.sock --pause 600
./bin/rinlock notifications --socket /path/to/rinlock.sock --pause 0
```

A delivery pause is separate from policy mutes: it retains and continues building the eligible backlog, survives restart, and resumes automatically at expiry. The maximum pause is 24 hours. Dashboard **3**, then **p**, pauses delivery for 10 minutes or resumes it. A request already in flight can finish; the worker checks the pause before each subsequent send.

Webhook JSON retains the event's existing top-level fields and adds an optional `incidents` array containing incident snapshots at enqueue time. The triggering event's score can be below the notification threshold when an incident qualifies. Delivery remains asynchronous and at least once, with durable retries and database-local event IDs as idempotency keys. Receivers should namespace these keys per host/database. No external webhooks are sent by the self-contained `demo` command or by the tests.

## Local API additions

The owner-only Unix socket now supports mutations. There is no TCP listener.

- `GET /v1/policy`, `GET /v1/policies`
- `PUT /v1/policy` with `{"config":{...},"expected_revision":"..."}`
- `GET /v1/incidents?limit=100&before=ID` (newest incident ID first, exclusive cursor)
- `GET /v1/events/ID`
- `GET /v1/notifications`
- `PUT /v1/notifications` with `{"seconds":600}`; `0` resumes

Existing `/v1/rules` returns the currently active config. `/v1/status` storage stats include the incident count. Anyone who can access the socket can modify policy and delivery controls; role-based access is future work.

## Validation and remaining release work

Tests cover concurrent activation/ingestion, invalid and stale updates, rollback, failed activation persistence, immutable replay, process/boot/window boundaries, descendant correlation across restart, login bursts, cooldown escalation, mutes, paused outbox restart/resume, API input validation, dashboard navigation, and a complete synthetic daemon workflow. The race-enabled full suite and `go vet` are the checks for this stage.

The privileged native validation gate from [stage 1](stage-1.md) remains pending. This stage does not certify live probes. Stage 3 still needs retention/compaction, bounded delivery/dead-letter policies, rate limits, packaging/service hardening, deployment performance measurements, and deeper operator workflows. Policy versions, events, incidents, and their indexes have no retention yet. Expired correlation subjects/cooldowns are periodically pruned during ingestion, with full scans every 256 events; high-cardinality latency needs measurement before production use. Multi-channel routing, cross-host incidents, configurable sequence languages, and incident replay are not implemented.
