# Stage 3: operating a single-host detector

Optional kernel blocking and its separate controls are covered by the
[prevention guide](prevention.md). The original stage 3 measurements below are
historical; [stage 4 validation](stage-4-validation.json) records successful live
container tests of native collection, prevention and collector integration.

The operational implementation is present. Userspace, synthetic collector transport, persistence/recovery, and package checks can run without root. **Live BPF and installed systemd-service validation remain a release gate**, not a claim made by these tests. See [stage 1](stage-1.md) for the privileged suite.

## Deployment and privilege separation

The release archive contains a static Linux amd64 binary, the compiled CO-RE sensor object, systemd units, a system user definition, configuration examples, and an installer. Linux amd64 remains the only supported build target. The service capability set targets modern kernels with `CAP_BPF` and `CAP_PERFMON`; it deliberately does not fall back to `CAP_SYS_ADMIN`.

```sh
make package
(cd dist && sha256sum -c rinlock-0.3.0-dev-linux-amd64.tar.gz.sha256)
# From inside the unpacked archive:
sudo ./scripts/install.sh
```

Verify the checksum from the directory containing the archive, for example `(cd dist && sha256sum -c rinlock-0.3.0-dev-linux-amd64.tar.gz.sha256)`. The checksum detects accidental corruption; it is not a release signature. The installer preserves an existing `/etc/rinlock/rules.json`, installs files, creates the `rinlock` system user through systemd-sysusers, and reloads systemd. **It does not enable or start monitoring.** `DESTDIR=/tmp/package-root ./scripts/install.sh` stages files without touching the host's users or services.

Two processes run from the same binary:

- **`rinlock-collector.service`** runs as root with a bounded capability set. It loads probes, watches host files/accounts, optionally follows the journal, and sends observations over `/run/rinlock-collector/events.sock`. It has no database, policy API, or webhook worker. Its allowed socket family is AF_UNIX; it does not need Internet access. Its host filesystem view is retained so private `/tmp` or hidden home directories cannot silently alter configured watches.
- **`rinlock.service`** runs as the `rinlock` user with no capabilities and `NoNewPrivileges`. It owns `/var/lib/rinlock`, evaluates policy, stores evidence, applies retention, exposes `/run/rinlock/rinlock.sock`, and sends configured webhooks. systemd makes the rest of its filesystem read-only, hides home directories, and restricts other privileged operations.

The collector verifies its consumer's kernel-reported UID. The daemon requires a root peer on the telemetry socket. Only observation and health frames cross that socket; there is no command interface for the daemon to ask the collector to open files or execute programs. The owner-only API socket remains the operator control boundary. Root or the `rinlock` account can change policy and incident state; there is no multi-user role system.

The collector sends health heartbeats every second. Frames are limited to 1 MiB, reads to 10 seconds, and writes to 5 seconds. A disconnect, stalled stream, invalid frame, or collector failure terminates the daemon with an error. systemd retries failed services with backoff and start-rate limits. Telemetry is not spooled while disconnected; restarted collectors begin fresh baselines. An interrupted connection therefore creates a monitoring gap, recorded as a critical `collector_gap` event when the daemon can still persist, and visible in service failures/restarts. A kernel administrator remains able to disable or evade the detector; this is not protection against a fully compromised kernel/root environment.

### First installation

```sh
sudo /usr/local/bin/rinlock check --config /etc/rinlock/rules.json
sudo /usr/local/bin/rinlock doctor --probe --bpf-object /usr/local/lib/rinlock/sensor.bpf.o
# Also run make test-live as root from a source checkout on each deployment kernel.
sudo systemctl enable --now rinlock.service
sudo -u rinlock /usr/local/bin/rinlock status --socket /run/rinlock/rinlock.sock
sudo -u rinlock /usr/local/bin/rinlock dashboard --socket /run/rinlock/rinlock.sock
```

Run the privileged suite on a disposable deployment-equivalent host before enabling production monitoring. Validate the actual installed units too: root-only probe tests do not prove the restricted collector's capability set works on that kernel/LSM configuration. Check `coverage`, process UIDs/capabilities, logs, collection under load, service restart behavior, and notification recovery.

Optional webhook configuration belongs in `/etc/rinlock/webhook.env`, root-owned mode 0600, using the supplied example. Only the unprivileged service receives those environment variables. Do not put credentials in the command line. The interactive `rinlock demo` still never reads webhook credentials.

The combined `serve` mode remains available for development and compatibility. `serve --sensor-socket PATH` enables separation; it must use the same `watch_files`, `watch_users`, and `auth_journal` settings as the collector. Detection policy can hot reload; collection and storage settings require restart. On upgrade, stop both services, back up the database/config, install the new archive, validate config, then start both. Keep a copy of the old binary and database for rollback; older software is not certified against newly written databases. Older saved policy versions inherit the new operational defaults when applied and receive a current-format revision ID.

## Storage lifecycle

Defaults:

```json
{
  "storage": {
    "retention_days": 30,
    "max_events": 1000000,
    "max_database_bytes": 1073741824,
    "min_free_bytes": 268435456,
    "maintenance_seconds": 60
  }
}
```

Maintenance runs at startup and at the configured interval. It removes unprotected events older than the retention period and then the oldest unprotected IDs necessary to reach the event-count target. It protects pending delivery events and the first 100 evidence IDs of every retained incident. Open/acknowledged incidents do not expire automatically. Resolved incidents expire only when both their last observation and review are older than the retention period and no pending delivery references them. Their index entries and review actions are removed with them. Other events can expire even if they once contributed evidence beyond the incident's first 100 IDs.

If protected evidence alone exceeds the count target, maintenance preserves it and reports `over_event_limit` in health. Operators must review/resolve incidents or increase capacity. Delivery payloads contain their own event/incident snapshots; their retention is controlled separately. Policy versions remain for attribution/rollback and are included in database capacity use.

`max_database_bytes` is a **high-water stop**, not a filesystem quota: ingestion and policy activation check the database file before writing. bbolt allocation growth and other transactions can cross that threshold. `min_free_bytes` checks available filesystem space before ingestion/activation. Reaching either stops collection rather than silently discarding new events. Shared-disk writers can race the free-space check; use a dedicated volume and filesystem quota where a strict byte ceiling is required. An event not yet committed when capacity is exhausted is not recoverable from the sensor stream.

Exports and replay use live paginated reads; concurrent retention can remove unprotected history between pages. For a stable long-lived replay source, preserve an export before its retention deadline.

Health includes database bytes, latest maintenance counts, protected event count, and event-limit pressure. Cleanup frees reusable bbolt pages but does not shrink the file. Maintenance currently scans retained records in one write transaction; the event-count target is enforced at maintenance intervals and can be exceeded between runs. Measure latency at the intended retention size; this is not an unlimited-volume storage engine.

### Backup and compaction

Stop both services before working on the database:

```sh
sudo systemctl stop rinlock.service rinlock-collector.service
sudo cp -a /var/lib/rinlock/events.db /var/lib/rinlock/events.db.backup
sudo -u rinlock /usr/local/bin/rinlock compact \
  --db /var/lib/rinlock/events.db \
  --output /var/lib/rinlock/events.compact.db
# After reviewing the successful verification and retaining the backup:
sudo -u rinlock mv /var/lib/rinlock/events.compact.db /var/lib/rinlock/events.db
sudo systemctl start rinlock-collector.service rinlock.service
```

Compaction refuses an existing output and a database locked by a running daemon. It creates and checks a separate copy, preserves bucket sequences/outbox state, flushes it, and never replaces the input. You need space for the backup and compacted copy. Take backups before compaction or upgrades; do not copy a live bbolt file with ordinary `cp`.

## Bounded asynchronous delivery

New notification settings, hot reloadable through the policy workflow:

```json
{
  "notifications": {
    "max_attempts": 10,
    "max_pending": 10000,
    "rate_per_minute": 60,
    "history_days": 30
  }
}
```

- Attempts and the per-UTC-minute budget are reserved in the database before HTTP. The rate limit permits bursts within a fixed minute; it is not a rolling-window or smooth token-bucket limit. It survives restart and applies to retries too. The single worker processes at most 32 candidates per second and sends requests sequentially, so achieved throughput may be lower than the configured limit.
- An interrupted attempt consumes budget and gets a 30-second retry lease. The HTTP timeout remains 5 seconds. A receiver may have accepted a request before a crash: delivery remains at least once, and receivers must namespace/deduplicate database-local event IDs.
- Most HTTP 3xx/4xx responses become failed records immediately. HTTP 408, 429, 5xx and transport failures retry with bounded exponential delay. `Retry-After` seconds/HTTP dates are honored up to 24 hours. The attempt limit ends retries even after repeated crashes.
- A full pending queue records `notification: queue_full` on the event and creates an inspectable failed record with the payload. Evidence collection continues. The cooldown reservation still applies to repeated copies of that alert.
- Completed/failed records retain their latest payload, attempt count, finish time and failure reason for `history_days`. A manual requeue resets the attempt budget, increments `requeues`, and preserves the idempotency key. It is only allowed for a failed record and while queue capacity remains. History is latest-state history, not an immutable record of every HTTP attempt.

```sh
rinlock deliveries --socket /run/rinlock/rinlock.sock --state failed
rinlock deliveries --socket /run/rinlock/rinlock.sock --state pending
rinlock deliveries --socket /run/rinlock/rinlock.sock --state delivered
rinlock deliveries --socket /run/rinlock/rinlock.sock --retry 42
```

Use the root or `rinlock` account for these commands. `--before ID --limit N` pages delivery history. Existing cooldowns, mutes, and pause/resume from [stage 2](stage-2.md) still apply. `queued` on an immutable event is an enqueue decision, not its current delivery status.

## Incident review

```sh
rinlock incident --id 1 --state acknowledged --note 'Investigating deployment activity'
rinlock incident --id 1 --state resolved --note 'Confirmed authorized maintenance'
rinlock incident --id 1 --actions
```

Add the service socket path when using these examples. In dashboard tab **4**, arrows select an incident, **a** acknowledges, **r** resolves, and **o** reopens. Updates use the incident revision, so a concurrent review/new observation rejects a stale update. The review log records time, note and the API connection's kernel-reported UID. A newly correlated observation increments the revision and automatically reopens a resolved incident, preserving the earlier action. Reviews do not mute detections or alter historical event scores.

API additions: `GET/PUT /v1/incidents/ID`, `GET /v1/incidents/ID/actions`, `GET /v1/deliveries?state=failed&before=ID&limit=100`, and `PUT /v1/deliveries/ID/retry` with `{}`. Review updates contain `expected_revision`, `state`, and `note`. Action history returns the latest 100 actions for that incident.

## Troubleshooting and release checks

```sh
rinlock version
systemctl status rinlock.service rinlock-collector.service
journalctl -u rinlock.service -u rinlock-collector.service --since '10 minutes ago'
rinlock coverage --socket /run/rinlock/rinlock.sock
rinlock status --socket /run/rinlock/rinlock.sock
```

- **Peer UID/config mismatch:** check the installed service users, socket ownership, and matching collector settings; do not loosen the peer checks.
- **Capacity stop:** free space, resolve incident retention pressure, or stop/compact the database and adjust storage settings. Restart after the cause is fixed.
- **Failed webhooks:** inspect failure reason, receiver authentication and availability; then explicitly requeue. Do not repeatedly reset attempts against a broken receiver.
- **Collector restart/disconnect:** inspect verifier/attachment errors, kernel limits, filesystem coverage and service start-rate limits. A restart is a potential monitoring gap, not proof that no events occurred.
- **Stale socket in manual combined mode:** verify the owner process has stopped before removing it. The code refuses to unlink arbitrary existing sockets; systemd removes each service's runtime directory on stop.

The 30-second synthetic CLI soak passed with 33 events, four incidents, failed-delivery recovery and incident reopening; peak daemon RSS was 16,752 KiB in that run. See [the recorded result](stage-3-validation.json). From a source checkout, re-run it with `python3 scripts/smoke.py --seconds 30`; `python3 scripts/check-package.py` checks the built archive and staged service files. These scripts also run in CI; the measurements here describe the original stage 3 build.

Validation includes race tests, a 5,000-event workload with repeated retention, pending/evidence protection, durable delivery budgets, failed delivery recovery, compaction verification, incident review/reopen, and synthetic collector transport/disconnect tests. `make benchmark` measures userspace scoring + capacity checks + synchronous persistence only. The initial local run measured about 24.2 µs/event, 31,522 allocated bytes/event, and 110 allocations/event on this environment's temporary filesystem and an i9-12900K. This is not a native collection throughput guarantee or a sustained disk benchmark.

Before a production release, still run privileged tests and the actual service pair on each supported kernel/LSM combination, then a deployment-sized sustained workload. Record CPU/RSS, database growth, maintenance pauses, queue depth, kernel/collector loss, shutdown/recovery behavior, and webhook outage recovery. Longer soak, kernel saturation and hard power-loss tests have not been completed here. Validate package installation/upgrades in disposable target VMs before distribution. Fleet management, kernel/root tamper resistance, multiple delivery destinations, and cross-host correlation remain outside the single-host v1 scope.

Service settings were checked against the upstream [systemd execution documentation](https://github.com/systemd/systemd/blob/main/man/systemd.exec.xml) and [unit dependency documentation](https://github.com/systemd/systemd/blob/main/man/systemd.unit.xml); static validation does not replace exercising the installed units.
