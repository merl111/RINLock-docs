# Changelog

## 0.5.0 — unreleased

- Opt-in Trivy scanning for local rootfs, application directories and image archives.
- Cached advisory sync, validated atomic CISA KEV refresh and feed freshness reporting.
- Persisted scan baselines, transactional change events/outbox and restart deduplication.
- Async scan controls, CLI filtering/pagination and sixth terminal dashboard tab.
- Coverage/failure reporting, isolated scanner subprocesses and real offline scanner test.
- Public GitHub Pages guides, raw Markdown, agent workflows and versioned documentation manifests.
- Checked documentation builds and a docs-only publication workflow that keeps application source private.

## 0.4.0 — implementation milestone

- Optional BPF LSM prevention for execution, file opens and IPv4/IPv6 connect attempts.
- Ordered audit/enforce/disabled policies scoped to exact cgroup v2 workloads.
- Atomic rule publication, root-only control socket, runtime history and rollback.
- Prevention dashboard tab, decision audit records and async notification integration.
- Privileged container tests for actual kernel denial and collector/daemon integration.
- Documentation, real demo screenshots, GPL-2.0-only licensing and release automation.

## 0.3.0 — implementation milestone

- Separate privileged collector/unprivileged daemon, packaged systemd deployment.
- Retention, capacity checks, verified offline compaction and delivery operations.
- Incident review state/history with automatic reopening on new evidence.

## 0.2.0 — implementation milestone

- Versioned detection policies, runtime reload/rollback, replay and correlation.
- Notification cooldowns and mute/pause controls.

## 0.1.0 — implementation milestone

- Native eBPF collection, file/account/journal sources, event identities and loss reporting.
- Persistent events, detection rules, webhook outbox and terminal dashboard.

Milestones above describe implementation history; they do not assert that GitHub
releases or production-validation certifications were published.
