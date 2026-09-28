# Stage 1: reliable collection

> Historical implementation record. Pending-work statements below describe that
> stage. See the [current documentation](index.md) and [stage 4 validation](stage-4-validation.json)
> for subsequent features and successful live kernel tests.

## Delivered

- **Versioned kernel contract:** ABI 2, 416-byte records, size validation before load and header validation on every record. Kernel timestamps use boot time so suspend does not stop their clock.
- **Process identity:** boot UUID + host PID + group-leader start time in nanoseconds, with equivalent parent identity. Thread events share the process identity. PID reuse and reboot produce different IDs.
- **Resource identity:** executable and returned-file device/inode captured from kernel structures, with explicit capture provenance. Snapshot file identity for file-watch events. Kernel FD-table observation is still subject to concurrent mutation; it is not a reference-counted VFS audit event.
- **Guarded enrichment:** PID start time verified around `/proc` reads. Executable/file resolution pins an O_PATH descriptor and checks captured device/inode. Raw paths are retained. Missing enrichment does not discard the event.
- **File notifications:** inotify observes write, attribute, create, delete, and move events. Parent watches survive atomic file replacement; missing directory trees are reported and watched from their nearest ancestor. Five-second snapshots reconcile state. Metadata-only snapshots remain distinguishable from notifications.
- **Visible loss:** kernel ring/map losses and inotify overflow/invalidation generate stored, critical `collector_gap` findings. Inotify overflow counts represent episodes, not known lost-event totals.
- **Coverage:** `/v1/coverage`, `rinlock coverage`, and the TUI health view show capabilities, limitations, disabled/configured sources, counters, and uncovered paths.
- **Diagnostics:** `doctor` checks kernel/BTF/architecture/boot identity/object. `doctor --probe` temporarily attaches every required hook. Read-only checks never claim live readiness.
- **Privileged validation infrastructure:** isolated child workloads exercise exec, shell ancestry, allowed/denied fixture opens, effective-UID restoration, loopback connect, 32 short-lived processes, and repeated attach/detach. A dedicated workflow targets two Ubuntu runner distributions and records actual kernels.

## Validation performed locally

- Existing and expanded Go tests, including race detection and local daemon/API integration.
- Same-size file writes, permission changes, atomic replacement, transient create/delete, missing-parent recovery, and directory-watch invalidation with reconciliation disabled in event-driven tests.
- Deterministic injected inotify overflow tests verify explicit gap reporting and snapshot recovery; they do not simulate kernel memory pressure.
- ABI, process identity, old/new stored schema coexistence, inode checks, raw/resolved-path evidence, coverage API, and unprobed doctor semantics.
- Clang builds the eBPF program with warnings treated as errors.

## Pending release gate

The privileged suite has **not run on this workstation**: noninteractive sudo reports that a password is required. CI configuration has been written but has not been dispatched or observed. No supported-kernel matrix is certified yet.

On an appropriately privileged Linux host:

```sh
make build
sudo ./bin/rinlock doctor --probe
sudo env "PATH=$PATH" make test-live
```

Record the kernel, architecture, BTF availability, build revision, verifier/attachment result, live-test output, and event loss. Do not declare stage 1's native reliability gate complete until those tests pass on each intended supported deployment kernel.

## Remaining boundaries

This is still single-host observation. There is no recursive tree watch, universal file-read/write actor attribution, complete capability/GID monitoring, established-network-session tracking, universal login source, or Kubernetes identity integration. Inotify can coalesce notices; snapshots are not exact per-operation before/after records. A filesystem notification carries no trustworthy actor PID. File identity lacks inode generation, so inode reuse remains possible. Optional enrichment uses the supported architecture's USER_HZ conversion as a guard; the persisted process ID uses full kernel nanoseconds.

High-rate/long-duration stress, deliberate kernel-ring saturation, exhaustive filesystem edge cases, kernel-version certification, and production performance budgets remain release work. Stage 2 policy management/correlation and stage 3 retention/packaging were not included in this change.

## Technical references

- [Linux inotify semantics and limitations](https://man7.org/linux/man-pages/man7/inotify.7.html)
- [Linux CO-RE overview](https://docs.kernel.org/bpf/libbpf/libbpf_overview.html)
- [x86 thread status and TS_COMPAT](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/thread_info.h)
