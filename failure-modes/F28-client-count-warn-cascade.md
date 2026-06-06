# F28 — Client-count `WARN` cascade — `nv_close_device` WARN_ON triggers on lost-GPU teardown

| Field | Value |
|---|---|
| **Sources** | C5 (v4 detector class for close-path; A4 close-path-telemetry observability) |
| **Confidence** | hypothesis |
| **Predecessor evidence** | `cascade-scope-audit.md` §"close-path WARN classification" — `WARN_ON(nv->open_count != 0)` and similar client-count invariants in `nv-pci.c` / `nv-pat.c` are reachable on lost-GPU teardown when the per-device file descriptors are still open at surprise-removal time |
| **fake-5090 mechanism** | Surprise-removal A primitive + active-client scenario (test harness keeps a per-device file descriptor open at surprise-removal time; the close-path WARNs are observable in the resulting cascade) |

## Symptom (driver view)

`nv_close_device` and adjacent close-path functions in `kernel-open/nvidia/nv-pci.c` and `nv-pat.c` contain `WARN_ON(condition)` invariant checks on client counts (`nv->open_count`, per-device resource counts, per-context handles). These invariants are designed for the steady-state case: "if a userspace client closes the FD, the bookkeeping should reach zero before the device tears down." Under surprise removal with active clients (a userspace process holds an open `/dev/nvidia0` FD when the cable is pulled), the bookkeeping does NOT reach zero by the time the close path runs — the userspace process is still alive, still holds the FD, and the FD's close handler runs from the userspace process's context AFTER the kernel-side teardown has already reached the WARN_ON site.

Each WARN_ON fires its `WARN_ON_ONCE` message plus a kernel stack trace into `dmesg`. The stack traces are large (typically 30-50 lines each); multiple WARN_ON sites can fire in close succession during the cascade; cumulative dmesg cost can be hundreds of lines per cascade event. This compounds the F24 dmesg-noise problem and obscures load-bearing signals.

The bug is structurally similar to F15 (bare `NV_ASSERT` on teardown chain) but in kernel-side code rather than RM-internal code, and using kernel's `WARN_ON` rather than RM's `NV_ASSERT`. The fix pattern is the same: gate the WARN_ON on whether sink-state is set ("WARN only if NOT in lost-GPU teardown — under lost-GPU teardown, the non-zero count is expected").

## Trigger sequence

1. Daemon brings device to a healthy bound state.
2. Test harness launches a userspace process that opens `/dev/nvidia0`, holds the FD open, and waits.
3. Test fires surprise-removal A.
4. Driver-side cascade detector fires; sink-state set.
5. `nv_pci_remove` runs; kernel-side close-path teardown begins; reaches WARN_ON sites.
6. WARN_ON conditions evaluate: `nv->open_count != 0` (the userspace process still holds the FD), per-device resource counts non-zero, per-context handles non-zero.
7. WARN_ON_ONCE fires for each violated invariant; stack traces emitted.
8. After kernel-side teardown completes, the userspace process is signalled (the FD's underlying inode is invalidated); userspace `close()` call eventually runs; the FD's close handler runs in userspace process context against a now-removed device.

## Expected post-patch behaviour

Each WARN_ON site is gated on sink-state:

```c
if (cond && !osIsGpuBusDead(pGpu)) {
    WARN_ON_ONCE(cond);
}
```

Or equivalently, the WARN_ON is replaced with a sink-state-aware variant:

```c
NV_WARN_ON_UNLESS_GPU_LOST(pGpu, cond);
```

Implementation choice depends on whether the project prefers per-site gating (cheap, mechanical) or a new macro (consistent with the `NV_ASSERT_OR_GPU_LOST` precedent from F15).

Under sink-state, the WARN_ON does not fire; the close-path runs to completion without dmesg noise from the violated invariants. Under steady-state, the WARN_ON still fires as designed — catching the bug the invariant was originally written to detect.

The userspace process eventually closes its FD; the kernel handles the close cleanly against the already-torn-down device (this is F12's territory, already covered).

## Assertion shape

**WARN_ON suppression under surprise-removal A:**

- Run the trigger sequence with an active userspace FD-holder.
- `dmesg | grep -cE 'WARN_ON.*nv_close_device|WARN_ON.*nv-pci.c|WARN_ON.*nv-pat.c'` (or the project's canonical WARN_ON signatures) MUST be 0 attributable to the cascade window.

**Per-site verification (source-side, CI):**

- The cascade-scope-audit's close-path WARN classification table lists specific sites. For each named site, the source MUST contain the `!osIsGpuBusDead(pGpu)` guard OR use the `NV_WARN_ON_UNLESS_GPU_LOST` macro.
- Any new `WARN_ON` introduced in `nv-pci.c` / `nv-pat.c` / adjacent close-path files MUST be reviewed against the same gate requirement.

**Steady-state preservation (negative — F28 fix MUST NOT suppress legitimate WARNs):**

- Run a scenario where the device is healthy but a bug causes a real invariant violation (e.g. a test harness deliberately leaks a context handle then triggers a clean module unload). The WARN_ON MUST fire as designed — sink-state is not set, the gate evaluates true, the WARN fires.

**Userspace close-path graceful handling:**

- After the kernel-side cascade-window teardown completes, the userspace FD-holder process MUST eventually receive its `close()` completion (per F12's existing assertion).
- `dmesg` MUST NOT contain a second wave of WARN_ON from the userspace-triggered close — that path is also subject to sink-state, which is still set, so the gates suppress the second wave.

**Dmesg-budget contribution:**

- F28's contribution to the F24 dmesg-budget MUST be 0 lines under surprise-removal A (each suppressed WARN_ON saves ~30-50 lines of stack trace).

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25-F27.
- **Active-client test scenario.** The test harness must include a userspace process that opens `/dev/nvidia0` and holds the FD open at surprise-removal time. This can be a simple binary (open + sleep + close) or a real CUDA program (a long-running compute task). The substrate does not need anything new — the close-path WARNs are on the kernel side, observable via dmesg.
- **Per-context-handle tracking (optional).** For exercising WARN_ON sites that depend on per-context-handle counts (rather than just `open_count`), the test harness's userspace process should allocate at least one resource (memory map, channel, anything that increments the per-context tracking). The substrate accepts these allocations during the healthy phase and does not need special handling during the cascade.

## Source citations

- `cascade-scope-audit.md` §"Close-path WARN classification" — site enumeration
- `cascade-class-design-v4.md` §"DETECTION INPUTS" §"Close-path WARN gating"
- `kernel-open/nvidia/nv-pci.c` (`nv_close_device` and adjacent functions; specific WARN_ON sites TBD by audit)
- `kernel-open/nvidia/nv-pat.c` (per-device WARN sites)
- F12 entry (`/dev/nvidia0` close-path wedge — sibling failure on the same path; F28 is the WARN-class subset)
- F15 entry (the bare `NV_ASSERT_OR_GPU_LOST` precedent in RM code; F28 is the kernel-side WARN_ON_UNLESS_GPU_LOST analog)
- F24 entry (dmesg-noise budget — F28 suppression contributes to the budget)
