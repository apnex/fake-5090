# F47 — Per-poll dead-bus marker coverage gap / counterproductive early marker

| Field | Value |
|---|---|
| **Sources** | C7-292-inflight-deadbus-poll-reader (base/L1, upstream-bound), A13'-292-inflight-aer-earlyfree (addon), A14-292-reopen-failfast-gate (addon) |
| **Confidence** | field-bug (live-FAIL) |
| **Predecessor evidence** | MISSION-1 A13 live-FAIL 2026-06-06 (apnex.31, `netcon3-2026-06-06-A13-live-FAIL-rpcpoll-wedge.log`) + A12/netcon2 silent lockdown wedge (`netcon2-2026-06-05-292-pathB-wedge.log`); design-of-record `design-2026-06-06-292-redesign-C7-A13prime-A14.md` |
| **fake-5090 mechanism** | Surprise-removal A (silent dead-bus) injected *during in-flight GSP bootstrap of a re-open* + multi-engine marker-coverage assertion |
| **Fix status** | DESIGNED, not implemented — C7 + A13' + A14 are the apnex.32 build spec; the patches do not exist yet |

## Symptom (driver view)

A persistence-OFF re-open of a #979-equivalent EQ-diverged TB-tunneled RTX 5090 (the GPU was cold-recovered by `fix-bar1`, last-closed with `WPR2=0`, then re-opened) advances ~1 s into `RmInitAdapter → kgspInitRm → _kgspBootGspRm → kgspBootstrap_GH100`, then the bus dies mid-bringup. An AER Uncorrectable Non-Fatal Completion-Timeout fires from the endpoint; the in-flight bootstrap worker then **storms** instead of aborting:

```
_kgspRpcRecvPoll: GSP RM / LibOS heartbeat timed out
_kgspIsHeartbeatTimedOut: ... timeout 5200 [ms]
tmrGetTimeEx_GH100: Consistently Bad TimeLo value ffffffff
```

~3 lines per iteration, every ~0.1 ms, until the host wedges (netcon3 capture ends mid-storm at +1.075 s). The foreground that scheduled the bounded worker is parked in `wait_for_completion_timeout` holding `nvl->ldata_lock` across `flush_work` (the F44 lock-inversion substrate); the storming worker never frees, so the foreground never joins → host hard-wedge.

The deeper failure is a **coverage gap**, not a single missing check. There are **two** software "GPU-lost" markers, honored by **disjoint** code regions, so freeing a stuck poll by setting one marker is *per-poll whack-a-mole*:

| Marker | Set by | Honored by (the ONLY readers) | Lock to set |
|---|---|---|---|
| `os_pci_is_disconnected` (Linux) | `os_pci_set_disconnected` (WRITE_ONCE) | **only** `osIsGpuBusDead()` (os.c:1884), referenced **only** inside the os.c MMIO readers `osDevReadReg008/016/032` + osinit.c | **lock-free** |
| `PDB_PROP_GPU_IS_LOST` (RM PDB bit) | `gpuSetDisconnectedProperties` / `cleanupGpuLostStateAtomic` | `_kgspRpcRecvPoll` pre-loop (kernel_gsp.c:2813) + `_kgspRpcSanityCheck` (kernel_gsp.c:314) | lock-free to write the bit; but the AER-reachable setter `rm_cleanup_gpu_lost_state` gates behind `rmapiLockAcquire(COND_ACQUIRE)` |

`osIsGpuBusDead` / `os_pci_is_disconnected` are **absent** from `kernel_gsp.c` and `gpu_timeout.c` (grep-confirmed): the GSP poll engines are structurally blind to the Linux marker.

A13 (the predecessor fix) set **only** `os_pci_is_disconnected`, early, directly from the AER handler (nv-pci.c:2981), and **never** `PDB_PROP_GPU_IS_LOST`. Two consequences fall out:

1. **It covered the lockdown poll only by accident.** `timeoutCondWait` (gpu_timeout.c) honors *no* lost-marker — it aborts only on timeout or if its cond-fn's own MMIO read happens to evaluate the cond TRUE. The lockdown cond `_kgspLockdownReleasedOrFmcError` (gh100:499) returns TRUE on `mailbox0 != 0`; the dead-bus value `0xFFFFFFFF != 0` → it exits in one iteration. That accident — not the marker — is what let A13 pass the lockdown gate. The sibling `_kgspFalconMailbox0Cleared` waits for `mailbox0 == 0`; `0xFFFFFFFF != 0` is *never* satisfied → the dead value makes that poll **worse** (polarity trap). Dead-value coverage is fragile, not principled.

2. **The early marker is counterproductive at the RPC poll.** `osDevReadReg032` short-circuits and returns `NV_GPU_BUS_DEAD_VALUE_U32` at os.c:2050 **before** ever reaching the post-read `DETECTOR_MMIO_DEAD` funnel at os.c:2081 that would call `cleanupGpuLostStateAtomic(DETECTOR_MMIO_DEAD)` and set **both** markers. By pre-setting `os_pci` early, A13 **disables the one self-heal path** that would eventually set `PDB_PROP_GPU_IS_LOST` — the marker `_kgspRpcRecvPoll` actually honors. The vanilla surprise-removal path self-heals *because* that detector fires; A13 suppressed it for the re-open, then the worker advanced to a poll (`_kgspRpcRecvPoll`) that honors only PDB → storm.

A further wrinkle (the §1.3 surprise): the DISCONNECT branch *does* dispatch the full sink `rm_cleanup_gpu_lost_state`, but that wrapper takes the RM API lock with `API_LOCK_FLAGS_COND_ACQUIRE` (non-blocking, the C1/C6 F44 fix). During the storm the worker **holds the reacquired API lock** (`kgspInitRm` releases it across the lockdown cond at kernel_gsp.c:4785, then REACQUIRES at 4818-4820 before the post-INIT_DONE control RPCs), so the AER thread's COND_ACQUIRE **defers** → PDB is never set → storm continues. Routing PDB through `rm_cleanup` from the AER thread cannot fix this; making the acquire blocking re-introduces the F44 deadlock.

## Trigger sequence

The whole point of F47 is to make the multi-engine coverage gap deterministic. The substrate must let a workload reach a *specific* GSP poll engine, then go dead-bus, and assert that the marker propagated by the patch is honored there.

1. Daemon brings the device to a bound state and begins a GSP bootstrap (`kgspBootstrap_GH100` HAL on GB202/5090 — confirmed by `tmrGetTimeEx_GH100` + `_kgspLockdownReleasedOrFmcError` in the capture).
2. Drive the bringup to a chosen poll engine:
   - **E1 / `timeoutCondWait`** family (lockdown, FSP GFW-boot, target-mask, can-send, reset) — early in init.
   - **E2 / `_kgspRpcRecvPoll`** family (INIT_DONE drain and the **post-INIT_DONE control RPCs** `SET_GUEST_SYSTEM_INFO`@5105, `GET_GSP_STATIC_INFO`@5112) — late in init, after `_kgspHeartbeatInit` (kernel_gsp.c:6162) has armed the heartbeat. The storm is necessarily here, because `_kgspIsHeartbeatTimedOut` is only meaningful post-INIT_DONE.
   - **E3 / hand-rolled own-loops** — `GspMsgQueueSendCommand` tx-full (`while(NV_TRUE)`, 1 s/element, sysmem, **no MMIO escape**), `GspStatusQueueInit` link, `kfspWaitForResponse`.
3. Transition to surprise-removal A (silent dead-bus, all MMIO `0xFFFFFFFF`) *while parked in the chosen engine*. Optionally fire a synthetic AER Completion-Timeout (the real re-open's MMIO touch on the dead chip does this).
4. Have the test harness set the patch's marker source (the lock-free `os_pci` flag, as A13' does at AER time) and observe whether the chosen engine self-terminates or storms.
5. Sweep the engine choice across all 13 poll-sites (below) and **both bringup funnels** (`nv_bootstrap_bounded` open/resume; `nv_dynpower_bounded` RTD3/GC6 resume).

### The 13 reachable poll-sites (from design-of-record §1.8)

Engine classes: **E1** = `timeoutCondWait`/`gpuTimeoutCondWait`; **E2** = `_kgspRpcRecvPoll`; **E3** = hand-rolled own-loop. "Covered by A13 alone?" = whether the lock-free `os_pci` marker terminated that site.

| # | Poll-site | Eng | Marker honored today | Covered by A13 (os_pci) alone? |
|---|---|---|---|---|
| 1 | `kfspWaitForSecureBoot` GFW-boot (gh100:828 / gb202:80) | E1 | none (cond/timeout) | accidental (encoding-dependent) |
| 2 | `kfspWaitForGspTargetMaskReleased_GH100` (gh100:1158) | E1 | none | accidental (≠0 → TRUE) |
| 3 | `_kfspWaitForCanSend` (kern_fsp.c:364) | E1 | none | accidental |
| 4 | **`_kgspLockdownReleasedOrFmcError`** (gh100:1050) **[netcon2 silent origin]** | E1 | none | **YES, accidental** (0xFFFFFFFF→≠0→TRUE 1 iter) |
| 5 | `_kgspFalconMailbox0Cleared` / `_kgspSpdmBootedOrFmcError` (gh100:620/662) | E1 | none | **NO** (`==0` polarity trap; CC/SPDM off → unreachable today) |
| 6 | `kgspResetHw` asserted/deasserted (gh100:133/144) | E1 | none | accidental; **TMR/PTIMER-dead edge** |
| 7 | `_kgspHasCcCleanupFinished` (gh100:1271) | E1 | none | accidental (CC off) |
| 8 | `_kgspWaitForResetAccess` (gb100:83) | E1 | none | accidental |
| 9 | `kgspWaitForRmInitDone → _kgspRpcRecvPoll(INIT_DONE)` (6151→2682) | E2 | **PDB_PROP_GPU_IS_LOST** | **NO** (gates on PDB, not os_pci) |
| 10 | **POST-INIT_DONE ctrl RPCs** (5105/5112 → 2682) **[netcon3 STORM]** | E2 | **PDB_PROP_GPU_IS_LOST** | **NO** (+ worker holds reacquired API lock → AER sink defers) |
| 11 | **`GspMsgQueueSendCommand` tx-full** (message_queue_cpu.c:544) | E3 | **none** — `while(NV_TRUE)`, 1 s/elem, **NO MMIO escape** | **NO** (sysmem; os_pci immune) — *worst silent F44* |
| 12 | `GspStatusQueueInit` link (message_queue_cpu.c:322/382) | E3 | partial (MMIO escape only) | partial/accidental |
| 13 | `kfspWaitForResponse` (kern_fsp.c:411) | E3 | none — hand-rolled | accidental |

**Plus the funnel gap (GAP-1):** A13's AER early-free is gated on `bootstrap_in_flight`, set only in `nv_bootstrap_bounded` (nv.c:1924). The second bringup funnel `nv_dynpower_bounded` (RTD3/GC6 resume, nv.c:2094-2117) has its own `queue_work` and never arms the flag → on a dynpower-driven re-bringup the marker is never set at AER time and the wedge reproduces regardless of poll coverage.

## Expected post-patch behaviour (C7 + A13' + A14, apnex.32)

The fix is **C7 = a read-only dead-bus poll-reader** placed at the engine chokepoints, so *every* GSP poll engine *reads* the lock-free `os_pci` marker A13' already sets, via one new thin exported predicate `osIsGpuBusLost(pGpu)` (wraps the existing `osIsGpuBusDead` = `os_pci OR PDB`). Read-only: **no lock, no PM-state write, no precondition violation** — the predicate is TRUE only when a healthy bus never is, so every added check is a pure-FALSE no-op on a live bus.

- **C7-e1/decl** — add exported `NvBool osIsGpuBusLost(OBJGPU *pGpu) { return osIsGpuBusDead(pGpu); }` in os.c (~1896); declare in the project-owned `nv-gpu-lost.h` (NOT generated NVOC headers). `osIsGpuBusDead` stays `static inline` so the hot per-register read path is not de-inlined.
- **C7-e2** — `gpu_timeout.c` `timeoutCondWait` loop top: `if (osIsGpuBusLost(pGpu)) { status = NV_ERR_TIMEOUT; break; }`. **One** edit covers all `gpuTimeoutCondWait` sites #1-#8, polarity-independent (closes the site-#5 `==0` trap and the site-#6 dead-PTIMER edge).
- **C7-e3** — `_kgspRpcRecvPoll` pre-loop (2813): OR-in `|| osIsGpuBusLost(pGpu)` into the existing `bFatalError || PDB_PROP_GPU_IS_LOST` (preserves the existing `bPollingForRpcResponse = NV_FALSE` clear → returns `NV_ERR_RESET_REQUIRED` before the for-loop and the `done:` label → **zero storm prints**). Closes #9/#10.
- **C7-e4** — `_kgspRpcSanityCheck` (314): OR-in `|| osIsGpuBusLost(pGpu)` → `NV_ERR_GPU_IS_LOST`; frees a worker already parked mid-for-loop on its next iteration.
- **C7-e5** — wrap the `done:` heartbeat-print blocks (2974-2981) in `if (!osIsGpuBusLost(pGpu)) { ... }`: the **predicate call** is what reads the dead PTIMER and emits "Bad TimeLo"; belt-and-suspenders.
- **C7-e6 (REQUIRED)** — the three hand-rolled E3 loops (msgq:544 highest priority, msgq:322, kern_fsp.c:411): explicit `if (osIsGpuBusLost(pGpu)) { ...; break; }` short-circuits — do not rely on the accidental MMIO escapes.
- **A13' (marker source, KEPT + extended)** — keep the lock-free `os_pci_set_disconnected` early-free verbatim; **arm `bootstrap_in_flight` around the `nv_dynpower_bounded` `queue_work` too** (closes GAP-1); **fix the lock-model comment** at nv.c:1959-1968 (the worker does NOT hold only the GPU-group lock — the API lock is REACQUIRED at 4818-4820 for the post-INIT_DONE RPCs, so `rm_cleanup`'s COND_ACQUIRE may defer).
- **A14 (defense-in-depth, addon)** — fix-bar1 sticky-bit re-open fail-fast gate in BOTH funnels; refuses a diverged-recovered (`diverged_recovered && reopen_gsp_torndown`) re-open with `-EIO` *before* any GSP poll is entered. Deterministic for the known substrate, probabilistic on novel divergence → ships **with** C7, never instead.

Post-fix, both captured wedges close at the **same structural property** (lock-free marker honored by every poll engine), independent of which poll the diverged chip dies at:

- **netcon3 (A13 storm):** the first `_kgspRpcRecvPoll` re-entry after the marker is set hits the C7-e3 pre-loop → returns `NV_ERR_RESET_REQUIRED` **before** the for-loop/`done:` → none of the ~46 heartbeat / 47 "Bad TimeLo" storm lines are emitted → `kgspInitRm` unwinds → API lock released → `flush_work` joins → foreground `-EIO`, `ldata_lock` released. Storm eliminated at source.
- **netcon2 (silent lockdown):** with A13' arming the marker on whichever funnel, the lockdown `timeoutCondWait` (C7-e2) breaks on `osIsGpuBusLost` on the next iteration → unwind → `flush_work` joins well before the +3000 ms budget.

## Assertion shape

**Per-engine coverage invariant (the load-bearing assertion):**

- For EACH of the 13 poll-sites and BOTH funnels, drive the bringup to that engine, go dead-bus, set the marker. The worker MUST self-terminate within bounded iterations (≤1 poll period for E1; the next re-entry for E2; the next loop iteration for E3). No site may rely on the `0xFFFFFFFF != 0` accident.
- Specifically assert the storm is **absent at source**, not hidden: `dmesg | grep -cE '_kgspRpcRecvPoll: .*heartbeat timed out'` returns `0`, and `grep -c 'Consistently Bad TimeLo'` returns `0`, after the marker is set.

**Polarity-trap (site #5) regression:**

- Reproduce a cond-fn whose dead-bus value evaluates the cond FALSE (the `mailbox0 == 0` shape). Pre-C7, this spins to full timeout (`os_pci` makes it worse). Post-C7-e2, the `osIsGpuBusLost` marker break terminates it regardless of cond polarity.

**Counterproductive-early-marker regression (the A13 lesson):**

- Assert that the production marker source (A13') does NOT pre-empt the `DETECTOR_MMIO_DEAD` self-heal in a way that *removes* the only path to `PDB_PROP_GPU_IS_LOST` while leaving a PDB-only poll reachable. With C7 present this is moot (every engine reads `os_pci` directly), but the fixture must prove C7 closes the gap A13's early marker opened: a worker that advances past the lockdown gate to `_kgspRpcRecvPoll` MUST still terminate.

**Dual-funnel symmetry (GAP-1):**

- Run the repro on `nv_bootstrap_bounded` (open/resume) AND `nv_dynpower_bounded` (RTD3/GC6 resume). Both MUST arm the marker source (assert `bootstrap_in_flight` is set around each funnel's `queue_work`) and both MUST self-terminate identically.

**Observability-amplifier isolation (the netcon3 method):**

- Run the primary repro at `console_loglevel=8` (netconsole armed) AND minimised observability (`printk` rate-limited, async `/dev/kmsg`, external liveness probe, passive sysrq). **SURVIVAL at BOTH loglevels** is the proof the storm is removed at source, not merely hidden by reduced printk pressure (the netcon3 host died from a synchronous netconsole printk-storm amplifying the genuine F44 hold). See F11 — observability perturbs this bug.

**No-regression (each n≥3):**

- Clean cold-plug works (every C7 check FALSE, A14 gate inert): BAR1 32 GiB, persistence engages, no new log lines.
- WPR2 fast-fail re-open still contained (live bus → `osIsGpuBusLost` FALSE → identical fast-fail), not refused by A14.
- Persistence-ON open/close/re-open works (WPR2 not cleared → A14 inert).
- No new blocking lock acquire anywhere on the AER/poll path (the `COND_ACQUIRE` is NOT made blocking — that is the F44 wedge).

## fake-5090 mechanism mapping

- **Phase 1** BAR-MMIO state machine (scriptable per-read `0xFFFFFFFF`) + **surprise-removal A** primitive, fired *while parked at a specific GSP poll engine*. F47 is the only failure mode whose substrate must reach a *named GSP poll-site* before going dead-bus — it needs a way to stage bringup progress (e.g. gate the lockdown mailbox / INIT_DONE drain / control-RPC reply) so the test controls *which* of the 13 sites is active at dead-bus time.
- **AER Completion-Timeout** injection (the re-open's MMIO touch on the dead chip) is optional but matches the field repro and exercises the COND_ACQUIRE-defer race in the two-marker table.
- The daemon-side MMIO read counter is test infrastructure: a *covered* engine stops reading the dead bus within bounded iterations; an *uncovered* engine read-storms (the F4 surface). F47 asserts the storm stops at every engine, which is strictly more than F4's single-reader bound.
- This is the multi-engine generalisation of F09 (marker-propagation gap): F09 is the A2-watchdog-vs-`osHandleGpuLost` two-marker race on the *teardown/detection* path; F47 is the GSP-poll-engine two-marker coverage gap on the *in-flight re-open bringup* path, where the second marker (`PDB_PROP_GPU_IS_LOST`) is unreachable because the early `os_pci` marker pre-empts the self-heal funnel that would set it.

## Cross-references

- **F09** (`osHandleGpuLost` marker-propagation gap) — the predecessor two-marker bug on the detection/teardown path; F47 is the bringup-path / poll-engine analogue. F09's "propagate to BOTH markers at every detection site" is exactly the property A13's early single-marker set *broke* for the re-open by suppressing `DETECTOR_MMIO_DEAD`.
- **F04** (dead-bus MMIO sentinel) — the `0xFFFFFFFF` sentinel readers F47's poll engines either honor (E1 accidentally, via the cond-fn's MMIO read) or ignore (E2/E3); C7 replaces the per-read sentinel accident with a principled per-loop marker read.
- **F10** (recovery storm) — sibling *storm* class: F10 is unbounded *recovery retries*; F47 is an unbounded *poll re-call* storm (≈23 returns vs one abort) inside a single bringup. Same "unbounded loop, no principled abort" surface, different layer.
- **F11** (observability-induced failure) — the netcon3 host death was amplified by the synchronous netconsole printk-storm at `console_loglevel=8`; F47's validation MUST run dual-loglevel to separate the genuine F44 hold from the printk amplifier.
- **F16** (`_issueRpcAndWaitLarge` bypass) — sibling RPC-funnel coverage gap: F16 is a multi-chunk Control RPC path that skips the `PDB_PROP_GPU_IS_LOST` funnel; F47 site #10 is the post-INIT_DONE control RPCs reaching `_kgspRpcRecvPoll`, the same PDB-honoring funnel — both are "a reachable RPC path does not honor the lost-marker." F47's C7 chokepoint at `_kgspRpcRecvPoll`/`timeoutCondWait` is the structural answer to the whack-a-mole F16 represents per-call.

## Source citations

- design-of-record `design-2026-06-06-292-redesign-C7-A13prime-A14.md` §1.1 (two-marker table), §1.2 (lockdown-accident vs RPC-poll), §1.3 (COND_ACQUIRE defer), §1.4 (counterproductive early marker, os.c:2050 vs 2081), §1.5 (lock-model GAP-5), §1.8 (13-poll-site table + GAP-1 funnel gap), §2.1/§3 (C7 + A13' + A14 patch plan), §4 (no-regression), §5 (dual-loglevel validation), §6 (GAP-1..GAP-9).
- finding `finding-2026-06-06-A13-live-FAIL-rpcpoll-wedge.md` — the live-FAIL on apnex.31 (netcon3 timeline, root cause, "whack-a-mole across GSP polls," REDESIGN addendum: A13 counterproductive).
- Captures: `captures/netcon3-2026-06-06-A13-live-FAIL-rpcpoll-wedge.log` (apnex.31 storm), `captures/netcon2-2026-06-05-292-pathB-wedge.log` (apnex.30 silent lockdown).
- Source anchors (open-gpu-kernel-modules, verified 2026-06-06): `os.c` 1884/2050/2081; `kernel_gsp.c` 2813/314/2974-2981/4785/4818-4820/6162; `gpu_timeout.c` `timeoutCondWait`; `message_queue_cpu.c` 544/322/382; `kern_fsp.c` 364/411; `nv.c` 1924/1959-1968/2094-2117; `nv-pci.c` 2974/2981/3023; `gpu.c` 5266-5290; `nv-priv.h` 352.
