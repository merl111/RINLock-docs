![RINLock logo: an R integrated with a lock and circuit](docs/images/rinlock-logo.png){ .brand-logo width="144" height="144" }

<p class="hero-label">Linux security · Terminal first · Agent ready</p>

# Know what runs. Control what matters.

<p class="hero-lead">RINLock connects runtime behavior, explicit eBPF prevention,
and known vulnerabilities in one terminal dashboard.</p>

[Get started](getting-started.md){ .md-button .md-button--primary }
[Agent integration](docs/agents.md){ .md-button }

## Choose your workflow

<div class="grid cards" markdown>

- **Investigate behavior**

    Observe process execution, credential access, outbound connections, file and
    account changes. Review scored events and correlated incidents.

    [Detection configuration](docs/configuration.md)

- **Prevent explicit actions**

    Apply audit or enforcement policies for execution, file opens and network
    connections within selected cgroup workloads.

    [Prevention guide](docs/prevention.md)

- **Find vulnerable packages**

    Scan local targets using cached Trivy advisories. Prioritize CVEs listed in
    CISA KEV and receive asynchronous change alerts.

    [Vulnerability scanning](docs/vulnerabilities.md)

- **Automate with an agent**

    Use documented JSON commands, Unix-socket APIs, raw Markdown and a
    versioned manifest. Find the right guide without scraping navigation.

    [Machine-readable documentation](docs/agents.md)

</div>

## A dashboard in your terminal

```sh
make build
./bin/rinlock demo
```

The demo runs without kernel access, host scanning or webhooks. Use
`rinlock dashboard` to connect to an existing daemon. Press `1`–`6` to move between
events, rules, health, incidents, prevention and vulnerabilities.

![The vulnerability tab in a real terminal session](docs/images/dashboard-vulnerabilities.png)

This image shows a real Trivy scan of a disposable dependency manifest; runtime
events in that session are synthetic.

## Scope and status

RINLock targets a single Linux amd64 host. Prevention and vulnerability scanning
are opt-in. Scores express policy priorities; a CVE match is potential exposure,
not proof of compromise. Read [coverage and architecture](docs/architecture.md)
and the [security model](SECURITY.md) before deployment.

This site documents a published snapshot of the development branch. The version and source commit
appear on every page and in the [agent manifest](agent-manifest.json).
Historical implementation notes are labelled separately. Source and GitHub releases
require access to the private repository; these documentation pages are public.
