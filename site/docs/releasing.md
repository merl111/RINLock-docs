# Testing and releases

## Local development gates

```sh
make test                     # BPF compile, Go race tests, shared C policy tests
go vet -buildvcs=false ./...
make package
python3 scripts/check-package.py
python3 scripts/smoke.py --seconds 30
```

The package check verifies hashes, archive members, staged installation, config
preservation, permissions and systemd unit syntax. It never installs host services.
The synthetic soak exercises storage, incidents, notification failures/retries,
shutdown and offline compaction. It is not a kernel performance benchmark.

## Privileged kernel tests

For a root shell on a disposable compatible Linux host:

```sh
make test-live
make test-prevention-live
```

For a rootful Docker daemon on a BPF-LSM-enabled Linux host:

```sh
DOCKER_HOST=unix:///var/run/docker.sock scripts/test-container.sh
```

Omit `DOCKER_HOST` when the active context already selects the correct daemon.
The script compiles static test binaries, runs an ephemeral Alpine container with
networking disabled, mounts source read-only, creates temporary test cgroups,
attaches actual hooks and cleans up. Set `RINLOCK_TEST_IMAGE` to an alternate
Alpine image/tag or immutable digest (the default is pinned by digest). Privileged containers share the host
kernel: they cannot add BPF LSM support missing from the host. Host PID and cgroup
views are used for consistent process identity. Rootless containers are insufficient.

The suite verifies native collection/reattachment, prevention syscall outcomes,
audit overflow, collector-to-daemon persistence, root control requests, enforcement
before the first consumer and after daemon disconnect, and emergency disable.
It does not validate the installed systemd hardening profile or every kernel.

The `privileged prevention` workflow is manually dispatched on a dedicated
disposable self-hosted runner labelled `rinlock-bpf-lsm` with rootful Docker,
Go, Clang and libbpf headers installed. It is not run on untrusted pull requests.
The regular `checks` workflow runs userspace/build/package tests. The existing
`privileged sensor` workflow tests native collection on its Ubuntu runner matrix.

## Screenshots

README screenshots are captured from the real running synthetic demo through a
pseudo-terminal; no fabricated enforcement results are displayed. To regenerate:

```sh
make build
python3 -m venv /tmp/rinlock-shots
/tmp/rinlock-shots/bin/pip install pillow pyte pexpect
/tmp/rinlock-shots/bin/python scripts/screenshots.py
```

The renderer uses DejaVu Sans Mono and saves PNGs in `docs/images`. Review the
images before committing. They show demo fixtures, not host credentials or activity.

## Versioning and release workflow

Development builds use the Makefile's `VERSION` (currently `0.5.0-dev`). The release
workflow runs when a tag matching `v*` is pushed and validates it as `vX.Y.Z` or
`vX.Y.Z-alpha.N`, `-beta.N`, `-rc.N`. Major-zero and suffixed versions are marked
prereleases. It builds only Linux amd64, the supported native architecture.

Before tagging, run all local and privileged gates on the commit to release,
update the changelog/docs and record kernel/service validation. The hosted release
workflow cannot establish BPF LSM compatibility for deployment targets; that
evidence comes from the kernel suite and installed-service tests. A green hosted
build alone is not a production-readiness claim.

```sh
git tag -a v0.5.0 -m 'RINLock 0.5.0'
git push origin v0.5.0
```

The workflow validates modules, runs race tests/vet, builds a static binary and
BPF objects, validates the package, runs a synthetic smoke test, creates a
corresponding-source archive with vendored dependency source/licenses, uploads
artifacts, and publishes a GitHub Release with generated notes. Build jobs have
read-only repository permissions; only the final publish job has contents write.
Tags must already exist; the workflow never creates an implicit tag. Rerunning a
release replaces its assets from the same tag. Do not move published version tags.

Assets:

- `rinlock-VERSION-linux-amd64.tar.gz`: binary, both BPF objects, configuration,
  installer, systemd files, documentation and GPL license.
- `rinlock-VERSION-source.tar.gz`: exact project source plus vendored Go dependencies.
- Individual `.sha256` files and `SHA256SUMS`.

Verify `sha256sum -c SHA256SUMS` after downloading the corresponding files.
Checksums detect corruption; they are not detached publisher signatures.
No tag/release is created merely by building or installing the repository.

## Distribution and licensing

Project-authored code is GPL-2.0-only; see the root `LICENSE`. The binary archive
includes `THIRD_PARTY_NOTICES`, regenerated with `python3 scripts/notices.py`
when runtime dependencies change. BPF program license
declarations remain GPL-compatible. Dependencies retain their own upstream
licenses. The corresponding-source release includes vendored dependency source
and license files; preserve them when redistributing. Standard compiler/system
header prerequisites are listed in the README. Build source releases with:

```sh
VERSION=0.5.0 scripts/source-package.sh
```

This archives committed HEAD, not uncommitted workspace edits. Inside the extracted
source archive, run `make VERSION=0.5.0` with the documented toolchain; Go uses the
included vendor directory. `make install` installs files and reloads systemd unit
definitions, but does not enable services or prevention.

## Vulnerability adapter validation

Run `rinlock vulnerabilities --action check --config configs/vulnerabilities-example.json`.
The regular test suite uses local fixtures and HTTP receivers without feed downloads.
For the real Trivy adapter test, see [vulnerabilities.md](vulnerabilities.md#validation).
The release does not bundle the Trivy executable or advisory databases. Scanner
configuration/drop-in examples are shipped but remain inactive after installation.
