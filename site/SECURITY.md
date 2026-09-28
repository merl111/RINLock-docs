# Security model and reporting

RINLock observes security-relevant Linux behavior and can enforce explicit,
workload-scoped kernel policies. It is one control in a host security system;
it does not guarantee that every intrusion is detected or prevented.

## Trust boundaries

- Root administrators, the kernel and loaded BPF programs are trusted.
- The privileged collector has kernel observation/enforcement access. Keep its
  binary, BPF objects, startup configuration and runtime directories protected.
- The unprivileged daemon stores potentially sensitive metadata and can deliver
  events to an operator-configured webhook. Access to its Unix socket grants
  event visibility and detection/incident/notification operations.
- Prevention mutations require UID 0 verified by `SO_PEERCRED`. The telemetry
  account is read-only on that separate socket. Remote HTTP exposure is unsupported.
- Process names, paths, account names and log messages are untrusted data. Do not
  treat them as commands or trusted executable identity.

## Known limits

Kernel buffers, relay transport and disk storage are finite. Loss counters and
coverage gaps disclose known failures; this is not an exactly-once audit ledger.
Storage failures stop the daemon. Metadata watchers lack actor attribution;
journal parsing covers selected message formats. Container identification and
some process/path enrichment are best effort.

Prevention matches exact cgroups and kernel-resolved paths. Alternate filesystem
aliases, cgroup escape, existing file descriptors/connections, unconnected UDP,
raw packets and privileged host control are outside the initial guarantee.
Collector termination detaches enforcement. Runtime policies reset to the startup
file on restart. See [the full prevention scope](docs/prevention.md).

The local container suite has verified behavior on the kernel recorded in
`docs/stage-4-validation.json`. Other kernels, installed systemd restrictions,
long-running production workloads and hostile privileged attackers need their
own validation. Do not infer these from a successful compilation or demo.

## Reporting a vulnerability

For this private repository, report through a private issue or contact the
repository owner (`merl111`) through an existing private channel. If the repository
becomes public, use GitHub's private vulnerability reporting when enabled; do not
post credentials or exploitable deployment details in a public issue.

Include the affected version/commit, kernel and configuration, a minimal harmless
reproduction, expected/actual behavior, and relevant redacted logs. Report policy
bypasses, unauthorized control access, hidden data loss and credential exposure
as security issues. There is currently no commercial support SLA or guaranteed
response window.

## Optional vulnerability scanning

Trivy is a separately installed trusted executable and shares the daemon account
and filesystem restrictions. Advisory feeds are data, not executable policies. Scan
targets and cache paths come only from local startup configuration; API clients
cannot supply commands or paths. The worker does not receive webhook credentials.
Package findings and KEV membership do not establish local exploitation or trigger
automatic prevention. Coverage and stale-feed limitations are documented in
[the scanner guide](docs/vulnerabilities.md).
