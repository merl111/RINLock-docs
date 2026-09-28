# RINLock documentation

Start with the repository [quick start](../getting-started.md), then choose a guide:

- [Architecture and data flow](architecture.md): kernel sensors, identity,
  daemon, storage, policy boundaries, and notification delivery.
- [Configuration reference](configuration.md): all detection, collector,
  correlation, storage and notification settings.
- [Notification destinations](notifications.md): webhook, Slack, ntfy with iOS,
  SMTP TLS/authentication, routing, delivery tests and migration.
- [CLI and API reference](reference.md): commands, sockets, endpoints and dashboard.
- [Deployment and operations](stage-3.md): installation, systemd, retention,
  compaction, incident review, retries and recovery.
- [Runtime prevention](prevention.md): rule semantics, workload scope, activation,
  audit/enforce modes, rollback, failure behavior and kernel limits.
- [Vulnerability scanning](vulnerabilities.md): Trivy setup, cached advisories, KEV,
  coverage, persisted findings, notifications and terminal controls.
- [Testing and releases](releasing.md): local checks, container kernel tests,
  screenshots, version tags, release assets and source distribution.
- [Security model](../SECURITY.md): assumptions, boundaries and reporting.
- [Contributing](../CONTRIBUTING.md): development and change validation.
- [Agent integration](agents.md): Markdown discovery, JSON commands and automation workflows.
- [Maintaining the docs site](documentation-site.md): public Pages build, publication list and deployment.

Implementation records:

- [Stage 1: reliable collection](stage-1.md)
- [Stage 2: detection workflows](stage-2.md)
- [Stage 3 validation record](stage-3-validation.json) (historical build)
- [Stage 4 validation record](stage-4-validation.json)
- [Stage 5 scanner validation record](stage-5-validation.json)

The JSON examples under [configs](../configs/index.md) are executable documentation:
`rinlock check` validates detection settings; `rinlock prevention --check`
validates prevention settings. Each command's `--help` is the definitive flag
reference. `rinlock vulnerabilities --action check --config FILE` validates scanner settings.
These configuration formats serve different purposes and cannot be interchanged.
