# Vulnerability inventory and advisory scanning

RINLock can periodically check explicitly configured local targets against Trivy's
advisory database and enrich CVE matches with CISA's Known Exploited Vulnerabilities
(KEV) catalog. This runs in the userspace daemon, independently of eBPF collection.
It is **opt-in**: ordinary `demo`, `serve`, and installed services do not download
advisories or scan packages unless `--vulnerability-config FILE` is supplied.

A CVE match indicates potential exposure, not proof of exploitation. KEV means
exploitation has been observed somewhere in the wild, not necessarily on this host.
The scanner never executes exploit code or automatically installs prevention rules.

## Setup

Install a trusted Trivy binary separately. The adapter was tested with **Trivy
0.74.0**, JSON report schema 2 and vulnerability database schema 2. Download from
[Trivy's official releases](https://github.com/aquasecurity/trivy/releases) and
follow its [installation verification instructions](https://trivy.dev/docs/latest/getting-started/installation/).
RINLock neither bundles nor auto-upgrades the scanner executable. Keep the scanner
patched; it parses files and archives from scan targets.

Copy and edit [the example configuration](../configs/vulnerabilities-example.json):

```sh
cp configs/vulnerabilities-example.json /tmp/vulnerabilities.json
# Edit trivy_path, cache_dir, and targets for this machine and daemon user.
./bin/rinlock vulnerabilities --action check --config /tmp/vulnerabilities.json
./bin/rinlock serve --demo --vulnerability-config /tmp/vulnerabilities.json
# Another terminal, using the same daemon user's default socket:
./bin/rinlock dashboard
```

Here `--demo` only makes **runtime events** synthetic. Explicit vulnerability
configuration still performs real scans of the configured targets. For live
collection, use the normal `serve --sensor-socket PATH ...` flags instead.

The service starts a background sync followed by scans. Missing scanner binaries,
failed downloads and failed scans appear in status without stopping runtime
collection. Storage failures still follow the daemon's normal failure behavior.
The configuration is read at startup; changes require restarting the daemon.

## Configuration

| Field | Meaning |
| --- | --- |
| `version` | Required schema version, currently `1` |
| `trivy_path` | Absolute path to the scanner executable; default `/usr/local/bin/trivy` |
| `cache_dir` | Dedicated writable cache; default `/var/lib/rinlock/vulnerability-cache`; cannot be `/` |
| `sync_hours` | Advisory/KEV refresh interval, 1–168; default 24 |
| `scan_hours` | Cached scan interval, 1–168; default 6 |
| `timeout_seconds` | Timeout **per scanner subprocess**, 10–3600; default 900 |
| `java_db` | Also download Trivy's Java index during sync; default false |
| `targets` | 1–32 targets, each with unique `id`, `kind`, and absolute clean `path` |

Target IDs are ASCII letters, digits, dots, underscores or hyphens, 1–64 characters,
starting with a letter or digit. Target kinds:

- `rootfs`: an accessible Linux root filesystem, such as `/` or an exported rootfs.
- `filesystem`: an application directory containing supported dependency manifests
  or binaries. This does not claim complete OS package coverage.
- `image`: a **local container image archive** accepted by `trivy image --input`.
  For example, produce one with `docker save -o /path/image.tar IMAGE`.

There is no automatic container discovery, registry pull, Docker socket access,
remote target parameter, or arbitrary scanner-argument API. Image archives and
rootfs exports must be refreshed separately when workloads change. A mutable target
is not a point-in-time filesystem snapshot. Target paths are administrator-controlled.

Use a cache exclusively for this daemon. Scanner jobs are serialized within one
process; sharing the cache with other daemons or external writers is unsupported.
Budget disk for Trivy's databases: the main database can exceed 1 GiB and the
optional Java index adds more. This cache is separate from RINLock's event database
and its storage quota. Temporary scanner files also use the cache directory.

## Updates, offline scans and failure handling

- Startup and scheduled sync update the main Trivy database, optional Java index,
  and the [official CISA JSON feed](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json),
  then scan all configured targets. Manual `sync` and `refresh` both sync **and rescan**.
- `scan` uses the cached database and KEV catalog. Trivy runs with `--offline-scan`,
  `--skip-db-update`, and `--skip-java-db-update`; it does not pull images or resolve
  dependencies online. Initial sync needs network access. Missing caches fail visibly.
- KEV downloads have a 60-second HTTP timeout, no redirects, a 16 MiB bound, schema
  and count checks, duplicate-ID checks and a release-time rollback guard. A valid
  response replaces the cached catalog atomically; failure leaves the prior one.
- Trivy receives explicit flags, `/dev/null` config/ignore files and a restricted
  environment. Inherited `TRIVY_*`, webhook tokens, and home-directory credentials
  are not passed through. Proxy and TLS certificate environment settings are allowed.
- Database freshness reflects its published next-update time and a 48-hour maximum
  age. KEV freshness reflects the last successful download and configured interval,
  not just the catalog publication date. Stale cached scans carry warnings.
- Trivy stdout is bounded to 64 MiB and stderr to 4 KiB. More than 10,000 findings
  per report is rejected; split large targets. Timeouts kill the scanner process group.
- Failed scans preserve the last successful findings. Zero detected packages, or
  rootfs/image results without OS inventory, are marked `unknown`; absent previous
  findings are retained. Successful scans are labelled `partial`, never exhaustive.
- Successful reports may omit unsupported packages. Inspect package counts,
  warnings and `scanner_diagnostics`; a low count or zero findings does not establish
  that a system is safe. A distro-derived system such as Manjaro needs explicit
  coverage validation; it must not be assumed equivalent to Arch's advisories.
- Java artifact discovery may require the Java index. Enable `java_db` before
  scanning JARs; uncached offline analysis may fail. Other offline discovery limits
  depend on Trivy's supported formats.

Matching uses Trivy's distribution/vendor-aware advisories, including backported
fixes, rather than naive upstream version comparisons. See
[Trivy's vulnerability coverage](https://trivy.dev/docs/latest/guide/scanner/vulnerability/).
There is no separate NVD mirror maintained by RINLock.

## Findings, audit history and notifications

Latest findings are stored in the existing Bolt database per target. Identity
includes target, component, ecosystem, package path/name, installed version and
advisory ID. A scan transaction atomically commits its latest baseline, change
events and durable webhook outbox. A failed transaction cannot advance the baseline
while losing its notifications. Repeated scans and daemon restarts deduplicate.

Changes that create a `vulnerability` event:

- `new`: a newly detected package/advisory match;
- `changed`: severity, fixed version, or KEV membership changed;
- `no_longer_detected`: a previously detected match disappeared from a successful
  scan. This does **not** claim it was patched; package removal, changed advisory
  data, target content, or coverage can also cause disappearance.

All job completions/failures also produce `vulnerability_scan` audit events. Individual
scan summaries, timestamps, counts, warnings, diagnostics and errors are available
in status. Current baselines are retained independently of ordinary event retention;
change events use normal retention. Removed target IDs remain queryable as historical
baselines, and are not automatically reported as fixed. Reusing an ID with a changed
kind/path starts a new baseline. Prefer a new ID for a different system or artifact.

`vulnerable-package` priority scores: critical 90, high 75, medium 50, low 25,
unknown 20. `known-exploited-vulnerability` scores 95. These are policy priorities,
not CVSS scores or probabilities. `no_longer_detected` has no built-in alert score.
Normal custom rules, `notify_score`, mute settings, pause, retries, rate limits,
queue limits and manual failed-delivery requeue apply. Material changes have distinct
cooldown fingerprints; different CVEs do not suppress one another. Webhooks contain
a `vulnerability` object alongside the ordinary version-2 event fields.

This stage does **not** associate vulnerable packages with live processes, establish
internet exposure, prove a vulnerable function is reachable, distinguish installed
from still-loaded library versions, or enforce CVE-specific kernel mitigations.
Those require additional inventory and runtime evidence.

## Terminal dashboard and CLI

Press **6** for **Vulnerabilities**. `r` queues a cached scan; `u` queues sync plus
scan. Arrow keys select findings, Enter opens details, arrow keys scroll details,
`n` advances a page and `b` returns to the first page. `/` searches CVE, package,
component, severity or target text; Escape clears the filter. The tab shows feed freshness,
scan errors, partial/unknown coverage and fixed versions. Commands operate on the
same owner-only daemon Unix socket as the rest of the dashboard.

```sh
./bin/rinlock vulnerabilities --action status
./bin/rinlock vulnerabilities --action scan
./bin/rinlock vulnerabilities --action sync
./bin/rinlock vulnerabilities --kev
./bin/rinlock vulnerabilities --target host --search HIGH --limit 100
./bin/rinlock vulnerabilities --after NEXT_CURSOR
```

`--action init-config` prints defaults with an empty target list: add targets before
validation. Scan commands acknowledge a queued job immediately; poll `--action
status` for `busy=false`, `finished`, target errors and overall `error`. Only one job
can be queued or running; a second request receives HTTP 409. Existing findings
remain readable when scanning is disabled.

API:

- `GET /v1/vulnerabilities/status`: scanner job, feeds and target scan summaries.
- `GET /v1/vulnerabilities?target=...&search=...&kev=true&limit=100&after=...`:
  `{items, next}` sorted by stable finding ID; limit 1–1000. Pages reflect current
  state, not an immutable scan snapshot, so a concurrent rescan can change pagination.
- `PUT /v1/vulnerabilities/job` with `{"action":"scan"}` (or `sync`/`refresh`):
  HTTP 200 acknowledges scheduling, **not completion**. Requests cannot supply paths,
  executable names or arbitrary arguments. Configuration stays local and startup-only.
- `GET /v1/status` also contains the `vulnerabilities` status object.

## Installed service

The installer copies example config and an **inactive**
[vulnerability drop-in](../packaging/vulnerabilities.conf.example). It does not install
Trivy, enable scanning, change existing vulnerability configuration, or relax service
hardening. Copy/edit the config as `/etc/rinlock/vulnerabilities.json`, install the
scanner, then enable the reviewed drop-in and restart the daemon.

The scanner inherits `rinlock.service`'s unprivileged account and restricted
filesystem view. Hidden home directories, unreadable files, private `/tmp` and other
restrictions affect coverage. A scan of `/` is a scan of that service's visible root,
not guaranteed complete host coverage. Prefer explicit readable exports and application
paths; do not grant root or mount the Docker socket merely to hide coverage gaps.

## Validation

`make test` includes fixtures for parsing, KEV changes and invalid feeds, atomic
rollback, restart deduplication, per-finding cooldowns, asynchronous webhook delivery,
subprocess environment/argument isolation, process-group cancellation, API controls,
scanner-failure isolation from runtime collection, and terminal navigation/sanitization.
These tests require local sockets, but no internet or installed scanner.

For a real scanner test, sync a dedicated Trivy/KEV cache, then run:

```sh
RINLOCK_TRIVY_PATH=/absolute/path/to/trivy \
RINLOCK_TRIVY_CACHE=/absolute/path/to/cache make test-vulnerabilities-live
```

The opt-in test scans a disposable manifest for lodash 4.17.20, requires the known
CVE-2021-23337 match and fixed version, and checks repeated-scan deduplication. It
never installs/runs vulnerable application code and does not scan the host.

For end-to-end daemon/CLI validation and an optional actual terminal screenshot:

```sh
python3 scripts/vulnerability-smoke.py --trivy /absolute/path/to/trivy \
  --cache /absolute/path/to/cache --report /tmp/vulnerability-validation.json
# Add --screenshot with pexpect, pyte and Pillow installed to update the README image.
```

This smoke test performs a real startup sync, checks fresh feeds, scans a disposable
manifest, queues another scan over the CLI, checks deduplication, and shuts down its
temporary daemon. Runtime events are synthetic; vulnerability results are real.
