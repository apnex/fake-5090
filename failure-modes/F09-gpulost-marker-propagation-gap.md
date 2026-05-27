# F9 — `osHandleGpuLost` marker-propagation gap

| Field | Value |
|---|---|
| **Sources** | C5-crash-safety, A2-bus-loss-watchdog |
| **Confidence** | field-bug |
| **Predecessor evidence** | MISSION-1 E07 Run 2 (2026-05-26) — direct field observation; recorded in C5 intent doc |
| **fake-5090 mechanism** | Surprise-removal D (microsecond A→B race) — marker-propagation race fixture |

## Symptom (driver view)

The unpatched driver had two parallel "is this GPU lost?" markers:

1. **RM-internal marker** — set by `osHandleGpuLost` when an RM-side path detects loss.
2. **Linux `pdev`-side marker** — used by code paths that talk to the kernel PCI layer.

These were not propagated together. `osHandleGpuLost` would set the RM marker but leave the Linux marker clear; subsequent kernel-side code paths (and the A2 watchdog) kept believing the GPU was live, polled it, generated "disconnect detected" events that *should* have been suppressed, and produced redundant disconnects in the logs.

MISSION-1 E07 Run 2 (2026-05-26) captured this exact pattern in the field: `osHandleGpuLost` fired, RM marker set, but A2's watchdog kept polling because the Linux marker was absent — emitting a redundant `dead-bus detected` line after the RM side had already concluded the device was lost.

## Trigger sequence

The race is microsecond-scale; reproducing it deterministically is the whole point of fake-5090's surprise-removal D primitive.

1. Daemon brings device to healthy bound state; A2 watchdog kthread is running.
2. Test arms a *staged* dead-bus transition with a configurable delay between two signals:
   - **Signal A** (time T0): a path that calls `osHandleGpuLost` (e.g. a user-space MMIO read via the driver hits the dead bus).
   - **Signal B** (time T0 + Δ): the A2 watchdog's next poll period.
3. Set Δ small enough that the A2 poll happens *before* the RM-side marker propagation would naturally complete (on unpatched: never; on patched: must complete atomically with the RM marker set).
4. Run the test sweep across Δ ∈ {1µs, 10µs, 100µs, 1ms, 1 poll-interval, 10 poll-intervals} to characterise the race window.

## Expected post-patch behaviour

Every detection site (`osHandleGpuLost`, `osDevReadReg032`, A2 watchdog dead-bus detect) propagates the disconnect to BOTH markers — RM-internal AND Linux `pdev`-side — within the same critical section. A2's watchdog, on its next poll, sees the Linux marker is already set and uses the "already-disconnected, skip without re-firing latch" path. No redundant `dead-bus detected` log line.

## Assertion shape

**Single-detection invariant:**

- Across all Δ values in the sweep, the count of `dead-bus detected` log lines in `dmesg` MUST equal exactly 1 per dead-bus episode — regardless of which path detected first.
- Specifically: `dmesg | grep -cE 'tb_egpu_qwd.*dead-bus detected'` returns `1`, not `2` or more.

**Counter consistency:**

- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_detections` increments by exactly 1 per episode.
- This holds regardless of whether `osHandleGpuLost` or the watchdog won the race.

**Marker readback (white-box check, if exposed):**

- After signal A fires, both markers MUST be set within bounded time. If the driver exposes either marker via debugfs or sysfs (check intent doc — may not be directly observable), assert both are set.
- Indirect check: the C5 `os_pci_*` short-circuit (F4 assertion) must engage after signal A — proving the Linux marker propagated to where the primitives check it.

**Independent-detection symmetry:**

- Run two variants:
  - **9a:** Δ = 1µs (RM-side wins, watchdog catches markers already set on next poll).
  - **9b:** Δ = -1µs (i.e. watchdog poll happens first; A2 sees dead bus and propagates markers; subsequent `osHandleGpuLost` from user-space sees markers already set and short-circuits).
- Both variants MUST produce exactly 1 detection log line and 1 counter increment.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal D primitive.** This is the most subtle primitive the substrate needs to offer: a "stage A and B at microsecond-precise relative timing" facility for the dead-bus transition. Likely implementation:
  - A daemon command `device.stage_death(rm_path_delay_us=N, mmio_dead_at=T0)` that sets MMIO to `0xFFFFFFFF` at T0 (visible to the watchdog kthread) and signals a separate vfio-user MMIO-read trap to fire only on reads from a specific PID/path (the RM-side call path) at `T0 + N`.
- This is the *only* failure mode that requires precise inter-signal timing within the daemon. All other surprise-removal modes (A, B, C) are coarse-grained transitions.

## Source citations

- `C5-crash-safety.md` §"Marker propagation history" (lines ~46–48 in extracts JSON), §"Both markers propagated at every detection site" (line ~48), §"MISSION-1 E07 Run 2 field observation" (line ~47).
- `A2-bus-loss-watchdog.md` §"First dead-bus read propagates disconnect to both markers" (line ~75 in extracts), §"Already-disconnected GPU skipped without re-firing latch" (line ~77).
- MISSION-1 E07 Run 2 archive (2026-05-26) — the field-observation that turned this from a hypothesis into a confirmed bug.
