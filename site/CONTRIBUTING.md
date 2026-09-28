# Contributing

RINLock is developed as a Linux amd64 agent. Install Go 1.24+, Clang with a BPF
backend, Linux development headers and libbpf headers. Build with `make build`,
then run `bin/rinlock demo` for a synthetic UI without root.

Read [architecture](docs/architecture.md) and the [documentation index](docs/index.md).
Keep changes focused. Preserve the kernel/userspace ABI with size/version checks,
the root/unprivileged boundary, immutable event evidence, bounded queues, and
explicit loss reporting. Never turn a suspicion score directly into an enforcement
decision. New authorization behavior needs both denied and allowed-path tests.

Before proposing a change:

1. Format Go with `gofmt` and add meaningful tests for changed behavior.
2. Run `make test` and `go vet -buildvcs=false ./...`.
3. Validate package/install changes with `make package` and
   `python3 scripts/check-package.py`.
4. Run the [privileged kernel suite](docs/releasing.md) for BPF, ABI, identity,
   collector or enforcement changes. Describe any untested environment explicitly.
5. Update configuration examples, CLI help and relevant docs. Include screenshots
   for visible dashboard changes using the documented capture script.

Pull requests should describe the triggering behavior, resulting change,
validation performed, and material limitations. Do not commit binaries, databases,
credentials, environment files or live host telemetry. Synthetic fixtures are
appropriate for screenshots and tests, but must remain labelled synthetic.

Contributions are licensed under GPL-2.0-only, consistent with `LICENSE`.
Third-party code must retain its original notices and compatible license.

For documentation changes, install `requirements-docs.txt` in a virtual
environment and run `make docs` with that environment's Python. This validates
the website and generated agent formats. See [the publishing guide](docs/documentation-site.md)
for preview instructions and the explicit public-file list.
