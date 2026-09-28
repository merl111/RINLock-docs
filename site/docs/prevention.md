# Stage 4: explicit runtime prevention

RINLock has opt-in BPF LSM authorization for **program execution, file opens,
and IPv4/IPv6 connect attempts**. The original sensor continues logging its
observations. Detection scores do not turn blocking on. Nothing is enabled by
installing the release: the shipped prevention configuration has no rules.

## Prerequisites and validation

Use a disposable Linux amd64 machine with cgroup v2, kernel BTF, BPF LSM compiled
in (`CONFIG_BPF_LSM=y`), and `bpf` in the active LSM list. Keep the distro's other
LSMs enabled. Modern kernels with the three hooks and `bpf_d_path` support are
required; the real verifier/attachment probe is the support gate, not a version
number. All three hooks must load and attach. A requested prevention collector
fails startup on unsupported hooks or permissions; it never silently falls back
to observation.

From the source checkout:

```sh
make build
./bin/rinlock doctor --prevention
sudo ./bin/rinlock doctor --prevention --probe
sudo env "PATH=$PATH" make test-prevention-live
```

The probe attaches **empty** policies and immediately detaches them. The live
test creates a temporary cgroup and uses harmless fixtures to verify audit,
enforcement, exceptions, path replacement, symlinks, IPv4, IPv6, mapped IPv4,
scope isolation, emergency disable, and blocking during audit-ring overflow.
When opted in, missing privileges or verifier/attachment failures fail the test;
they do not skip. Normal `make test` runs the actual C matching/decision routines
with mocked maps and audit transport, Go policy/control tests, and object ABI
checks. Those tests do not prove the kernel verifier accepts the programs.

A rootful Docker alternative is `scripts/test-container.sh`; see
[container validation](releasing.md#privileged-kernel-tests). It also verifies
collector-to-daemon persistence and enforcement across daemon disconnect.

## Configure a workload

Edit a copy of `configs/prevention-example.json`. It demonstrates a web service
with shell/file deny rules and a database connection exception. Replace the
example workload, executable paths and network ranges with real values.

Rules are evaluated in order; **the first matching enabled rule wins**. Put
allow exceptions before broader deny rules. No match allows the operation,
subject to other Linux security policies. Existing LSM denials are preserved.

Each rule contains:

- `id`: unique, 1–63 ASCII letters, digits, underscores, dots or hyphens.
- `mode`: `audit` (default), `enforce`, or `disabled`.
- `action`: `deny` (default) or `allow`.
- `operation`: `exec`, `file_open`, or `connect`.
- `cgroup`: exact existing directory below `/sys/fs/cgroup`.
- Optional `uid`: real UID in the host's initial user namespace, including 0.
- For files/exec: `path`, a literal canonical absolute pathname, at most 255 bytes.
- For connect: `cidr` and optional `port` (omitted/0 means every port).

An audit deny emits `would_block` and permits the operation. An enforce deny
returns `EACCES`. An allow rule records `allowed` or `would_allow`; it cannot
override another security module's denial. Audit rules still participate in
first-match ordering, so an audit exception before an enforce deny permits that
matching operation. Policy maximum: 32 rules.

```sh
./bin/rinlock prevention --check --config my-prevention.json
sudo install -m 0644 my-prevention.json /etc/rinlock/prevention.json
```

Local validation checks syntax and semantics. Activation additionally resolves
the cgroups. The root cgroup, symlink aliases, and the collector's own exact
cgroup are refused. Active bindings hold open cgroup descriptors and report their
IDs in status. Recreated cgroups need policy reapplication. Child cgroups are
**not** included automatically; ordinary child processes remain covered while
they stay in the selected cgroup. Use separate rules for child cgroups.

The collector and recovery shell must run outside protected workloads. A workload
able to move itself to an unprotected cgroup or administer BPF can bypass these
policies; this is not protection against a fully privileged host administrator.

## Start the collector and daemon

The collector startup file must be a root-owned regular file, with no group/other
write permission. Direct startup (supply matching detection settings to both):

```sh
sudo ./bin/rinlock collect --config configs/default.json \
  --consumer-user rinlock \
  --prevention-config /etc/rinlock/prevention.json \
  --prevention-object bpf/prevention.bpf.o
```

The unprivileged daemon connects with its existing `--sensor-socket` option.
Prevention is supported in the separate collector deployment; `demo` and combined
`serve` do not load prevention. When running manually, ensure the collector's
runtime directory/socket group permits the telemetry user to connect. Packaged
systemd units already use root:rinlock with a 0750 runtime directory.

For an installed deployment, the installer provides an **inactive** drop-in
example at `/etc/rinlock/prevention.conf.example`. After validation and policy
review, copy it to
`/etc/systemd/system/rinlock-collector.service.d/prevention.conf`, reload systemd,
and restart the collector and daemon. That restart creates an enforcement gap.
No service activation or boot-setting changes are performed by this feature.

## Operate and recover

```sh
rinlock prevention
sudo rinlock prevention --rule web-no-shadow --mode enforce
sudo rinlock prevention --rule web-no-shadow --mode audit
sudo rinlock prevention --config my-prevention.json
rinlock prevention --history
sudo rinlock prevention --rollback REVISION
sudo rinlock prevention --disable
```

`--socket` selects the prevention control socket (default
`/run/rinlock-collector/prevention.sock`). Reads allow root and the configured
telemetry user; mutations require a kernel-authenticated UID 0 peer. Clients
also verify that the server is root. The unprivileged daemon cannot activate
policies through its detection API. `--expected GENERATION` supports scripted
compare-and-swap; otherwise the CLI fetches the generation before applying.

Dashboard tab **5 Prevention** shows ordered rules, bindings, counters, and active
revision. Arrow keys select a rule; `a`, `e`, `d` set audit/enforce/disabled;
`D` disables every rule. Mutations require a root dashboard. Use
`dashboard --socket DAEMON_SOCKET --prevention-socket CONTROL_SOCKET` for custom
locations. CLI history and rollback are available for recovery.

Policy maps are built and frozen before a single outer-map update publishes the
complete ruleset. Failed preparation leaves the active rules unchanged. Each
decision includes the matching rule and revision; control events include the
full normalized policy, generation and acting UID. The last 64 snapshots are
available for runtime rollback. **Hot changes are runtime-only**: restart reloads
the root-owned startup file. Save an intended permanent policy there explicitly;
changing the file alone does not reload running hooks. After emergency disable,
also disable/update the startup file before any restart.

## Failure behavior and audit delivery

- Kernel decisions never wait for the daemon, database, network or webhook.
- A daemon disconnect leaves prevention attached in the collector. The collector
  waits for a new telemetry consumer and reports session failures in its log.
- Collector exit/crash detaches prevention. Policies/links are not pinned across
  collector restarts or reboots. There is no fail-closed guarantee across a
  collector failure. Service health monitoring and workload supervision remain
  necessary for deployments requiring continuous prevention.
- Matching decisions enter a bounded 4 MiB kernel ring. Audit overflow increments
  loss counters; a deny remains a deny. On reconnection, the daemon receives a
  `collector_gap` event. Control changes have a bounded 128-event audit queue;
  ordinary updates stop when full. Emergency disable still works and counts any
  lost control record. Delivery/storage failures can lose audit data; this is not
  an exactly-once or zero-loss kernel audit trail.
- A file path resolution failure applies a scoped enforce-deny rule conservatively
  and records the failure. It never grants a path-based allow exception. With
  audit-only rules, it records `would_block` without denying. An unresolved path
  may therefore deny an unrelated long pathname in the same protected workload.
- Deny decisions receive suspicion priority 90 and use the existing persisted
  event stream and asynchronous notification outbox. Notification threshold,
  mute/cooldown and delivery limits still apply. Allow decisions are logged with
  priority 0 unless a detection rule scores them. Counters measure hook decisions;
  multiple hooks can run for one application-level action.

## Scope limits

Paths use the kernel-resolved pathname in the calling task's filesystem view.
Symlink spellings resolve to their targets; replacing a file at the same resolved
path remains covered. Alternate hard links, bind-mount paths, changed filesystem
roots, and renamed executable copies can have different paths. This is **path
policy**, not a guarantee against every alias or executable of equivalent content.
Exec policies check each binary/interpreter seen by `bprm_check_security`; they do
not prevent an existing interpreter from evaluating commands internally.

`file_open` prevents new opens, not reads through descriptors already open,
inherited or received from another process, nor existing mappings. It does not
implement a general file-mutation/metadata protection policy. `connect` covers
IPv4/IPv6 connection attempts, including connected UDP. It does not filter
unconnected UDP `sendto`, existing connections, raw packets, Unix sockets, DNS
names or application payloads. IPv4-mapped IPv6 destinations use IPv4 rules;
native IPv6 needs its own rules. Network allowlists require both families and
appropriate send/packet hooks for complete egress confinement.

Account creation, login prevention, privilege-transition enforcement, persistent
kernel links, recursive/orchestrator workload identity, comprehensive file access
control and packet-level quarantine are further extensions. They are not implied
by enabling these three operation types.

## Technical references

- [Linux BPF LSM](https://docs.kernel.org/bpf/prog_lsm.html)
- [Linux security hook interfaces](https://docs.kernel.org/security/lsm-development.html)
- [Kernel d_path helper implementation and LSM eligibility](https://github.com/torvalds/linux/blob/master/kernel/trace/bpf_trace.c)
- [Tetragon enforcement semantics](https://tetragon.io/docs/concepts/enforcement/)
