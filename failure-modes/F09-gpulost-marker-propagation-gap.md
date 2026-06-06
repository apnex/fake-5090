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

## Update 2026-06-06 (A13 live-FAIL)

The #292 A13 live-FAIL (apnex.31, captured `netcon3-2026-06-06-A13-live-FAIL-rpcpoll-wedge.log`) promotes the two-marker gap from a watchdog *log-redundancy* nuisance to a **host-wedge** root cause, and pins the exact propagation geometry at source (`/root/open-gpu-kernel-modules`, re-verified 2026-06-06). Cross-ref **F47** (the in-flight re-open GSP bring-up deadlock this wedge lives inside) and the **C7 fix** below.

**Live-confirmed two-marker truth (disjoint readers).** The two "GPU-lost" markers are honored by structurally disjoint code regions — neither reads the other's flag:

| Marker | Set by | The ONLY readers | Lock to set |
|---|---|---|---|
| `os_pci_is_disconnected` (Linux) | `os_pci_set_disconnected` (WRITE_ONCE, lock-free) | **only** `osIsGpuBusDead()` (os.c:1884), referenced **only** inside the os.c MMIO readers `osDevReadReg008/016/032` (+ osinit.c) | lock-free |
| `PDB_PROP_GPU_IS_LOST` (RM PDB bit) | `gpuSetDisconnectedProperties` / `cleanupGpuLostStateAtomic` | `_kgspRpcRecvPoll` pre-loop (kernel_gsp.c:2813-2814) + `_kgspRpcSanityCheck` (kernel_gsp.c:314-318) | lock-free bit write; but the AER-reachable setter `rm_cleanup_gpu_lost_state` gates behind `rmapiLockAcquire(COND_ACQUIRE)` |

- **`os_pci_is_disconnected` changes only what a poll READS, never the loop's abort predicate.** It is honored *exclusively* by `osIsGpuBusDead` inside the os.c MMIO readers, which makes a dead-bus register read return `0xFFFFFFFF` (`NV_GPU_BUS_DEAD_VALUE_U32`). It does not appear in any loop-exit test. A poll only aborts on it *accidentally*, when a cond-fn's own MMIO read of `0xFFFFFFFF` happens to evaluate its condition TRUE (e.g. lockdown `_kgspLockdownReleasedOrFmcError` mailbox0`!=0` → exits in 1 iter), and *fails* when polarity inverts (e.g. SPDM `_kgspFalconMailbox0Cleared` waits mailbox0`==0` → `0xFFFFFFFF != 0` → spins to full timeout — the marker makes that poll **worse**).
- **`_kgspRpcRecvPoll` honors ONLY `PDB_PROP_GPU_IS_LOST`** (pre-loop short-circuit + per-iteration `_kgspRpcSanityCheck`). With only the Linux marker set, `PDB` stays clear, `IS_CONNECTED` stays TRUE, `osIsGpuShutdown` stays FALSE → **no gate trips** → the GSP RPC poll storms the dead bus (the netcon3 heartbeat-timeout storm).
- **`osIsGpuBusDead` / `os_pci_is_disconnected` are ABSENT from `kernel_gsp.c` and `gpu_timeout.c`** (grep-confirmed: zero hits in `kernel_gsp.c`, `kernel_gsp_gh100.c`, `gpu_timeout.c`). The GSP poll engines (`timeoutCondWait`, `_kgspRpcRecvPoll`) are structurally blind to the Linux marker. This is the live analogue of the original A2-watchdog gap — the *same* "RM set, Linux clear (or vice-versa)" disjointness, now at the GSP-init poll layer rather than the watchdog.

**Why "set both markers from AER" does NOT close it.** The AER DISCONNECT sink *does* dispatch the full PDB-setter — `nv_pci_error_detected` DISCONNECT branch (nv-pci.c:3023) calls `rm_cleanup_gpu_lost_state(AER_FATAL)` — but that wrapper acquires the RM API lock with `API_LOCK_FLAGS_COND_ACQUIRE` (osapi.c:1913, the non-blocking F44-fix acquire). During the storm the GSP bring-up **worker holds the reacquired API lock** (`_kgspBootGspRm` REACQUIRES it at kernel_gsp.c:4818-4820 for the post-`INIT_DONE` control-RPC phase), so the AER thread's `COND_ACQUIRE` **defers** → `PDB_PROP_GPU_IS_LOST` is never set → the poll keeps storming. The earlier "A13 never attempts PDB" framing is sharpened: the PDB-setter **is** dispatched but is **COND_ACQUIRE-deferred**. This rejects any fix that routes PDB through `rm_cleanup_gpu_lost_state` from the AER thread, and rejects forcing that acquire blocking (re-introduces the F44 wedge).

**A13's early os_pci marker is COUNTERPRODUCTIVE, not merely insufficient.** Pre-setting `os_pci_is_disconnected` makes `osDevReadReg032` short-circuit (os.c:2050) and return the dead value **before** reaching the post-read `DETECTOR_MMIO_DEAD` funnel (os.c:2081-2092) that would call `cleanupGpuLostStateAtomic(DETECTOR_MMIO_DEAD)` and set **both** markers. So A13 disables the one self-heal path that would have eventually set the very `PDB_PROP_GPU_IS_LOST` that `_kgspRpcRecvPoll` honors. The vanilla surprise-removal path self-heals *because* that detector fires; A13 suppressed it for the re-open.

**The C7 fix (designed, not yet built — target apnex.32).** The chosen #292 fix supplies the missing **reader** rather than another marker-setter: a read-only thin predicate `osIsGpuBusLost(pGpu)` (wraps the existing `osIsGpuBusDead` = `os_pci OR PDB`) wired into all 13 reachable GSP poll-sites — the two engine chokepoints `timeoutCondWait` (gpu_timeout.c) and `_kgspRpcRecvPoll` + `_kgspRpcSanityCheck` (kernel_gsp.c), plus 3 hand-rolled loops (`GspMsgQueueSendCommand` msgq:544 `while(NV_TRUE)`, `GspStatusQueueInit` msgq:322, `kfspWaitForResponse` kern_fsp.c:411). Read-only ⇒ no lock, no PM-state write, no precondition violation; TRUE only on a dead/lost bus ⇒ pure-FALSE no-op on a healthy bus. Companion `A13'` keeps the lock-free `os_pci` marker source and arms `bootstrap_in_flight` on the 2nd funnel `nv_dynpower_bounded` (RTD3/GC6) too; `A14` is a fix-bar1 sticky fail-fast gate (defense-in-depth). **Status: DESIGNED, patches do not yet exist.** Design-of-record: `/root/nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design-2026-06-06-292-redesign-C7-A13prime-A14.md`; live-FAIL finding: `.../finding-2026-06-06-A13-live-FAIL-rpcpoll-wedge.md`.

**Fixture implication.** The surprise-removal D primitive's marker-propagation assertion should extend beyond the A2-watchdog pair to a **GSP-poll-reader** assertion: a dead-bus marker set lock-free at AER time must be HONORED (loop-abort) by the GSP init poll engines, not merely *readable* via an MMIO sentinel. The accidental/polarity-trap coverage above is exactly the kind of per-poll fragility the fixture should make deterministic.
