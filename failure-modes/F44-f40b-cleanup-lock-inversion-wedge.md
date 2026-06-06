# F44 — F40b bounded-wait timeout-branch lock-inversion deadlock (WPR2-clear lockdown substrate)

**Sources (defense):** **A10** (`f40b-lockfree-sink` — the lock-free `os_pci_set_disconnected` arm in the A6/A7 timeout branch). **Enabled-by:** [F09](F09-gpulost-marker-propagation-gap.md) (the dual-marker design — A10 uses the lock-free *Linux* marker because the RM marker can't arm under the held lock). **Trigger / cause:** [F40](F40-reinit-gsp-lockdown-wedge.md) (the chip-side GSP-lockdown stall, WPR2-clear sub-substrate). **Sibling:** [F42](F42-leaked-bounded-wait-worker-uaf.md) — the *other* bug *in* the A6/A7 bounding mechanism (F42 = leaked-worker UAF; F44 = timeout-branch deadlock).

**Confidence:** **field-bug** — directly reproduced as a host hard-wedge 2026-06-02 (`rung5-tbcure.sh --variant tb-only`, 2 reboots), then **source-confirmed end-to-end** by the `lockdown-reopen-wedge-fix-design` workflow. This is the **first live exercise of the WPR2-clear lockdown substrate** (R2–R4's 30 contained fires all hit the WPR2-already-up fast-fail twin, where this never manifests).

| Field | Value |
|---|---|
| **fake-5090 mechanism** | Chip-state substrate `userspace-recovered + WPR2=0` (a clean LAST-CLOSE drove `WPR2→0`, but the chip is still #979-divergent). On re-open, the chip **answers** the polled GSP lockdown-status MMIO (HWCFG2/MAILBOX0) but **never releases lockdown and posts no FMC error**, so `gpuTimeoutCondWait` spins-with-yield for the full RM `gpuTimeout`. The substrate must also expose the **second-contender** path (a CTO/AER `error_detected` arriving during the poll) to close the cycle. Distinct from F40's `userspace-recovered` substrate by the **WPR2=0** value (F40's classic repro is WPR2-already-up). |

## Symptom (driver view)

A **re-open** of a cleanly-shut-down (`WPR2=0`) but divergent eGPU **hard-wedges the host instantly and silently** — no Xid, no AER in `dmesg`, no panic, no oops, no NMI backtrace, no `hung_task`. The kernel log **stops dead** at the prior clean close. A6 is present *and armed* (`is_external=1`, not the first-open coverage gap A9 closed) — and it **still wedges**. This is the discovery that **A6's bounded-wait alone does NOT contain the lockdown substrate**.

## Mechanism (source-confirmed lock chain)

1. **Foreground** `nvidia_open` takes `down(&nvl->ldata_lock)` (nv.c:2029/2105) and calls `nv_open_device_for_nvlfp_bounded` **while holding `ldata_lock`** (nv.c:2113, released only at :2119).
2. **Worker** (`system_long_wq`, nv.c:1884) → `rm_init_adapter` → `rmapiLockAcquire(API_LOCK_FLAGS_NONE, RM_LOCK_MODULES_INIT)` = **blocking WRITE-lock** on the RM API rwlock `g_RmApiLock` (osapi.c:1761; rmapi.c:639 non-conditional). It holds this write-lock across `RmInitAdapter → kgspBootstrap_GH100 → gpuTimeoutCondWait(_kgspLockdownReleasedOrFmcError)` (kernel_gsp_gh100.c:1050) — on the WPR2-clear divergent chip the poll **never exits**, so the write-lock is held for the full RM `gpuTimeout` (~4 s graphics / ~30 s compute).
3. **A6's 200 ms timeout** fires; the **foreground (still holding `ldata_lock`)** calls `rm_cleanup_gpu_lost_state` (nv.c:1905) → `rmapiLockAcquire(API_LOCK_FLAGS_NONE, RM_LOCK_MODULES_DESTROY)` = **BLOCKING acquire of the SAME `g_RmApiLock` the worker holds** (osapi.c:1906). The "API lock contended, deferring" branch (osapi.c:1916-1926) is **UNREACHABLE** — `rmapiLockAcquire` only returns `NV_ERR_TIMEOUT_RETRY` under `RMAPI_LOCK_FLAGS_COND_ACQUIRE` (rmapi.c:595), and `NONE=0x0` (rmapi.h:63) lacks `COND_ACQUIRE=NVBIT(0)` (rmapi.h:64). So the foreground parks behind the worker for up to `gpuTimeout`, **before `flush_work` (nv.c:1943) even runs**.

**The inversion:** `ldata_lock` (held by fg) → waits-on → `g_RmApiLock` (held by worker) → not released until `gpuTimeoutCondWait` returns. A **soft sleeping pile-up** on *both* locks (the API lock is a sleeping rwlock; `ldata_lock` a semaphore; the poll yields via `schedule_timeout(1)`), finite at `gpuTimeout` **unless a second blocking contender closes a true cycle**:

- **PRIMARY (the instant-wedge trigger):** the GPU's **CTO/AER `error_detected`** fires during the stuck poll; `nv_pci_error_detected` (nv-pci.c:2931) → `rm_cleanup_gpu_lost_state` (nv-pci.c:2998) → `rmapiLockAcquire(NONE)` = the **same blocking write-lock the worker holds**, on the PCIe error-recovery workqueue → genuine cycle (worker waits for the chip event the AER machine would deliver; AER machine waits for the API lock the worker holds). This is the ledger's **H-OA6** lock-inversion.
- **SECONDARY:** a trailing `rmmod` → `nv_pci_remove_helper` blocks on `ldata_lock` (nv-pci.c:2456) → a third parked thread, widening the wedge to the whole driver.

**Why A6/R0/C5 don't contain it:** A6 bounds only the *wait* (200 ms), not the worker's poll; the **C5 sink (`PDB_PROP_GPU_IS_LOST`) cannot be SET** because `cleanupGpuLostStateAtomic` needs the very `g_RmApiLock` the worker holds; R0's `flush_work` is *downstream* of `rm_cleanup`'s blocking acquire, so it never runs. All three were validated only on the WPR2-fast-fail twin (worker exits in µs).

**Invisibility (environmental):** the obpc kernel has `CONFIG_DETECT_HUNG_TASK` **unset** and `hardlockup_panic=0`, so a sleeping deadlock produces **no log, panic, oops, NMI, or hung_task** — the kernel log simply stops. (Capture for confirmation: kdump is active → set `hardlockup_panic=1`/`softlockup_panic=1` + sysrq-c.)

## Expected post-patch behaviour (A10)

**A10** inserts `os_pci_set_disconnected(nv->handle)` — the **lock-free Linux dead-bus marker** (single `WRITE_ONCE(pdev->error_state, …)`, no API lock; already exported, already callsited in C5) — in BOTH A6/A7 timeout branches, **before** `rm_cleanup`. The worker's GSP poll re-reads HWCFG2/MAILBOX0 each iteration via `osDevReadReg032`, whose **first statement is `if (osIsGpuBusDead(pGpu)) return NV_GPU_BUS_DEAD_VALUE_U32`** (C5/os.c) — and `osIsGpuBusDead` consults `os_pci_is_disconnected` **independently of the API lock and of `PDB_PROP_GPU_IS_LOST`**. So the marker makes the worker's poll **self-terminate in microseconds** → `flush_work` joins immediately → A6's 200 ms `-EIO` fast-return is **restored on the lockdown substrate**. Adjuncts: **C1** (`rm_cleanup` `NONE→COND_ACQUIRE`, removes the foreground self-deadlock) + **C3** (AER `error_detected` non-blocking, removes the second contender). This is the **provably self-terminating worker** the R0 stop-rule feared was unachievable from kernel-open — achievable *because* the C5 patch already put a lock-free dead-bus check in `osDevReadReg032`.

## Assertion shape

- **Bug present:** re-open of a `WPR2=0` divergent chip → host wedge (instant via the AER second-contender, or a finite ~`gpuTimeout` soft-block holding `ldata_lock` without it). dmesg shows the clean close then **nothing**.
- **Fix verified (A10):** the same re-open returns **bounded `-EIO` (~200 ms), host alive**, worker self-terminated (`flush_work` joins in µs), no deadlock; concurrent `rmmod`/AER no longer parks. Regression guard: the **WPR2-fast-fail path still fast-fails** (R2-style; A10's marker is set only on the A6/A7 *timeout*, never reached on the fast-fail).
- **Observability:** `tb_egpu_f40b_fires` increments; post-fire `tb_egpu_state` reflects the sink; the A6 `timed out` line is present and the syscall returns.

## fake-5090 mechanism mapping

A **host-kernel lock-inversion** (foreground+worker on `ldata_lock`/`g_RmApiLock`, closed by the AER/rmmod second-contender), gated on a chip-state substrate that holds the GSP lockdown poll open (WPR2-clear, lockdown-never-releases, no-FMC-error). Reproduce by: substrate value `userspace-recovered + WPR2=0 + lockdown-stuck` + a re-open + a modeled CTO/AER second-contender; assert the foreground blocks on the API-lock acquire (osapi.c:1906) and the worker holds the lock across the poll. Phase: chip-state substrate + lock-model + second-contender injection. Discriminator: does the timeout branch set the lock-free Linux marker (A10, worker self-terminates) or block on the RM API lock (pre-A10 wedge)?

## Source citations

- Lock chain: `kernel-open/nvidia/nv.c` (ldata_lock :2029/2105/2113, A6 worker :1884, timeout `rm_cleanup` :1905, `flush_work` :1943); `src/nvidia/arch/nvalloc/unix/src/osapi.c` (`rm_init_adapter` :1761, `rm_cleanup_gpu_lost_state` :1889-1949, blocking acquire :1906, dead deferring branch :1916-1926); `src/nvidia/inc/kernel/rmapi/rmapi.h` (`COND_ACQUIRE`=NVBIT(0) :64); `kernel_gsp_gh100.c` (`gpuTimeoutCondWait` :1050).
- AER second-contender: `nv-pci.c` (`nv_pci_error_detected` :2931 → `rm_cleanup_gpu_lost_state` :2998, err_handler :3065-3070).
- Defense A10 mechanism: `os_pci_set_disconnected` (os-pci.c) + `osIsGpuBusDead`/`os_pci_is_disconnected` short-circuit in `osDevReadReg032` (C5/os.c).
- Forensics: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/wedge-2026-06-02-lockdown-reopen-forensics.md`; fix design workflow `lockdown-reopen-wedge-fix-design` (2026-06-02).
