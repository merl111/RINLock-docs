# CLI, dashboard and API reference

`rinlock COMMAND --help` lists every flag. Examples below assume an installed
daemon socket at `/run/rinlock/rinlock.sock`; source/development commands default
to a user runtime/data location. Pass `--socket` explicitly for installed services.
The detection socket belongs to the daemon user. Prevention controls use the
separate root collector socket, `/run/rinlock-collector/prevention.sock`.

## Commands

| Command | Purpose |
| --- | --- |
| `version` | Build version and platform as JSON |
| `demo` | Temporary synthetic daemon/dashboard; no host monitoring or webhooks |
| `serve` | Collect or consume a collector, score, persist, expose API, notify |
| `collect` | Privileged telemetry collector; optional prevention |
| `dashboard` | Interactive client; quitting leaves standalone services running |
| `doctor [--probe]` | Sensor prerequisite checks; optionally attach real probes |
| `doctor --prevention [--probe]` | BPF LSM checks; optionally attach empty hooks |
| `init-config` | Print complete default detection JSON |
| `check --config FILE` | Validate detection configuration |
| `status`, `coverage` | Service health, losses and source limitations |
| `events --limit N --min-score N --search TEXT` | Recent events as NDJSON |
| `export --min-score N --search TEXT` | All matching events, newest first |
| `event --id N` | Full event/evidence record |
| `policy` | Active detection policy and revision |
| `policy --config FILE` | Activate a detection configuration |
| `policy --history`, `policy --rollback REVISION` | Detection versions and rollback |
| `replay --config FILE [--input FILE\|-] [--changed-only]` | Compare candidate detection scores without writes or notifications |
| `incidents` | Recent correlated incidents |
| `incident --id N [--state STATE --note TEXT] [--actions]` | Inspect/review incident and review history |
| `notifications --pause SECONDS` | Pause delivery; 0 resumes queued work |
| `deliveries --state pending\|failed\|delivered` | Delivery history and failures |
| `deliveries --retry ID` | Requeue a failed delivery with a fresh attempt budget |
| `compact --db SOURCE --output NEW` | Verified offline copy; source remains untouched |
| `vulnerabilities [--action list\|status\|scan\|sync\|refresh]` | Package findings and background scan controls |
| `vulnerabilities --action check --config FILE` | Validate scanner configuration |
| `prevention` | Active prevention rules, generation, bindings and counters |
| `prevention --check --config FILE` | Local prevention validation |
| `prevention --config FILE` | Root-only runtime activation |
| `prevention --rule ID --mode audit\|enforce\|disabled` | Root-only rule mode change |
| `prevention --history`, `prevention --rollback REVISION` | Runtime prevention versions and rollback |
| `prevention --disable` | Root-only emergency runtime disable |

Collection flags: `--config`, `--bpf-object`, `--socket`, and `--consumer-user`.
Optional prevention flags: `--prevention-config`, `--prevention-object`,
`--prevention-socket`. `serve --sensor-socket PATH` consumes the separate collector;
without that flag it uses combined collection. `serve --demo` uses synthetic data.
`serve --vulnerability-config FILE` enables the separate advisory scanning worker.
Storage uses `--db`; webhooks use environment variables, not command-line secrets.

Review states are `open`, `acknowledged`, `resolved`. New evidence can reopen a
resolved incident. Detection rollback is persisted in policy history, but the
daemon startup file remains authoritative on restart. Prevention hot changes and
history are runtime-only. These are independent policy namespaces.

## Dashboard

- `1` Events, `2` Rules, `3` Health, `4` Incidents, `5` Prevention, `6` Vulnerabilities; Tab cycles views.
- Arrow keys or `j/k` move selection/scroll. Enter opens event details.
- `/` searches; `s` cycles severity thresholds; Space freezes the event list.
- `g/G` jump within the event list; Escape closes detail/search/help; `?` shows help.
- Health: `p` pauses notification delivery for 10 minutes or resumes it.
- Incidents: `a/r/o` acknowledge, resolve, or reopen the selected incident.
- Prevention: `a/e/d` sets the selected rule mode; `D` disables all rules. Root
  access to the separate control socket is required; ordinary viewers are read-only.
- Vulnerabilities: `r` scans, `u` syncs and scans, `n/b` next/first page, `/` search, Enter details.
- `q` or Ctrl+C exits the dashboard. Collection continues for standalone services.

The UI sanitizes control characters from external event text. Scores are priorities,
not probabilities. The demo labels itself synthetic and never enforces policies.

## Local HTTP API

The API is HTTP over a filesystem-protected Unix socket; it is not a network
listener or a multi-tenant remote service. Example:

```sh
curl --unix-socket /run/rinlock/rinlock.sock http://localhost/v1/status
```

Read endpoints include `/v1/status`, `/v1/coverage`, `/v1/rules`, `/v1/events`,
`/v1/events/{id}`, `/v1/policy`, `/v1/policies`, `/v1/incidents`,
`/v1/incidents/{id}`, `/v1/incidents/{id}/actions`, `/v1/deliveries`, and
`/v1/notifications`. Lists use bounded `limit` and optional `before` cursors.
Event queries also accept `min_score` and `search`.

Detection policy updates carry an expected revision to prevent overwriting another
operator's update. Incident review uses `expected_revision`, `state` and `note`.
The server obtains review actor identity from the Unix peer, not the JSON body.
See `internal/api` for the exact request/response structs; CLI clients implement
the same protocol.

Vulnerability endpoints are `/v1/vulnerabilities`, `/v1/vulnerabilities/status`,
and `PUT /v1/vulnerabilities/job`. See [the scanner reference](vulnerabilities.md)
for pagination, request bodies and asynchronous job semantics.

The prevention socket supports:

- `GET /v1/prevention`: current config/generation, counters and runtime semantics.
- `GET /v1/prevention/history`: up to 64 runtime snapshots.
- `POST /v1/prevention`: root-only change with `expected_generation` and exactly
  one of `config`, `rollback`, `disable: true`, or `rule` plus `mode`.

Missing/unauthorized peer identity returns 403; malformed JSON returns 400;
invalid/stale activation returns 409. The API has bounded requests and timeouts.

## Event and notification contracts

Event JSON version 2 includes time, source, kind, outcome, process/parent identity,
file/path/network context, score, findings, detection `policy_revision`, enqueue
decision and associated incident IDs. Optional fields depend on source coverage.
Prevention records add `prevention.rule`, `revision`, `operation`, `mode`, `action`
and optional `path_error`. Their revision identifies the kernel authorization
policy, independently of the detection policy that scored the observation.

Webhook bodies retain the top-level event fields and can include incident data.
Receivers should support additive fields and deduplicate the `Idempotency-Key`
header. Delivery authentication uses the optional bearer token. Delivery success
means the HTTP receiver accepted the request; it does not prove a human saw it.
