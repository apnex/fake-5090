# F45 — cold-bringup RM-API rwsem deadlock (failed cold open → deferred-open status-read)

**Sources (defense):** **A11** (`f45-deadlock-breaker` — D1 lock-free early-bail in `nv_open_device_for_nvlfp` + the `is_external_gpu`-gated `COND_ACQUIRE` in `rm_get_adapter_status`). **Depends-on:** **C6** (`cond-acquire-rwlock-fix` — the inverted `os_cond_acquire_rwlock_*` primitive; A11's COND would corrupt the rwsem without it). **Enabled-by:** [F09](F09-gpulost-marker-propagation-gap.md) (D1 reads the C5 `os_pci_is_disconnected` marker). **Trigger / cause:** a stochastic H16-class PCIe transient at GSP boot during a **cold** first-open. **Sibling:** [F44](F44-f40b-cleanup-lock-inversion-wedge.md) — both are global-`g_RmApiLock` inversions, but F44 is the A6/A7-*timeout* `rm_cleanup` blocking acquire (re-open lockdown substrate) and F45 is the **deferred-open status read on a failed cold init** (distinct trigger, distinct site).

**Confidence:** **field-bug** — directly reproduced as a host hard-wedge on obpc 2026-06-02 (cold engage `nvidia-smi -pm 1`; immune to SIGKILL, FLR, TB unauthorize+reauthorize; reboot-only), then **source-confirmed** by the `f45-deadlock-fix-design` workflow (8 agents, adversarial review). No vmcore (the kdump capture kernel itself hung re-probing the wedged eGPU — see `nvidia-driver-injector/.../wedge-2026-06-02-kdump-capture-failure-forensics.md`), so the cycle is reconstructed from 4 sysrq-t stacks + source.

| Field | Value |
|---|---|
| **fake-5090 mechanism** | Chip-state substrate `cold-init-fails-at-GSP-boot`: a fresh `modprobe` + first open enters `rm_init_adapter → kgspInitRm`; under **relaxed GSP init locking** (default on a consumer 5090, SPDM/CC off) the API lock is *released* across the GSP heartbeat poll, the injected PCIe transient makes the **GSP heartbeat time out** (`SET_GUEST_SYSTEM_INFO failed 0xf` → `RmInitAdapter 0x62:0xf:2131`), and the C5 sink declares the GPU lost. The substrate must then route a **second open** (or the deferred-open worker's own status read) into `rm_get_adapter_status`'s blocking API-lock acquire while a concurrent writer holds it. Distinct from F40/F44 by the **cold-init-failure** trigger (no prior clean close, no WPR2 substrate). |

## Symptom (driver view)

A **failed cold first-open** of the eGPU hard-wedges the host with the same invisibility as F44 — no Xid, no panic, no `hung_task` (`CONFIG_DETECT_HUNG_TASK` unset). The chip is declared lost (`cleanupGpuLostStateAtomic: GPU 0 lost via detector_class=3`, `tb_egpu recover: trigger gated (sink-set)`), the host stays *responsive to non-GPU work* (kubectl/bash live), but **every RM-API path is wedged** and no host-side reset clears it.

## Mechanism (source-confirmed lock chain — reconstructed, no vmcore)

The single global RM API rwsem `g_RmApiLock` (always taken WRITE — reads are downgraded for OSAPI, rmapi.c:572-578). Four wedged actors (sysrq-t):

1. **`nv_open_q` deferred-open worker** (single-threaded): cold `nv_open_device` failed → the open-failure arm calls `rm_get_adapter_status_external → rm_get_adapter_status → rmapiLockAcquire(READ→WRITE, blocking)` (osapi.c:2745) and **parks** there. It can never reach `complete_all(&nvlfp->open_complete)` (nv.c:1997).
2. **`nvidia_close`** of the SIGKILLed engage process waits in `nv_wait_open_complete` (nv.c:2690) for that worker's `complete_all` → blocked.
3. **`irq/NN-pciehp`** (surprise-removal path) does `nv_linux_stop_open_q → nv_kthread_q_stop → nv_kthread_q_flush` of the SAME single-threaded `open_q` (nv-pci.c:2451) → its flush sentinel can't run behind the parked worker → blocked.
4. A second `nvidia-smi` is a second queued API-lock writer, amplifying starvation.

**The keystone** is the parked worker (1): it holds the single `open_q` thread and gates (2) and (3). The cold-init thread itself releases the API lock cleanly on its failure return (osapi.c:1777) and is **not** the stranded holder — the permanent holder is an uncaptured/transient RM-API writer (no vmcore to name it). Reboot-only because an **unkillable kthread** (the worker) sits inside a kernel rwsem; FLR resets hardware but not a software rwsem, and TB unauthorize removes the device but the deadlock is lock-ordering, not MMIO.

**Latent primitive bug (found here):** `os_cond_acquire_rwlock_{read,write}` (os-interface.c) are **inverted** — `if (down_*_trylock(sem)) return TIMEOUT_RETRY` treats the rwsem trylock's `1=acquired` as failure (opposite of `down_trylock`'s `0=acquired`). Latent because no stock code takes the API lock with `COND_ACQUIRE`; any F44/F45 COND fix would have corrupted the rwsem (leak on success / release-unheld on contention) until C6 corrected it.

## Expected post-patch behaviour (A11 + C6)

- **C6** corrects the conditional-acquire primitive (the `!`), so `RMAPI_LOCK_FLAGS_COND_ACQUIRE` is genuinely non-blocking and reports acquisition truthfully (live-confirmed `down_write_trylock free=1/held=0`).
- **A11/D1** (lock-free, primitive-independent): on a failed open of an already-lost eGPU, `nv_open_device_for_nvlfp` sets `adapter_status = NV_ERR_GPU_IS_LOST` **without** the `rm_get_adapter_status_external` round-trip → the worker always reaches `complete_all` and returns. Cuts the keystone edge for the common case.
- **A11** (the COND site): `rm_get_adapter_status` takes the API lock with `COND_ACQUIRE` when `is_external_gpu`, so a contended acquire returns `TIMEOUT_RETRY` (→ best-effort `NV_ERR_OPERATING_SYSTEM`) instead of parking. Covers the not-yet-lost-but-contended case.

**Residual (no vmcore):** the cut frees the worker + pciehp flush + close; if a 5th thread *permanently* holds the write lock (vs transient starvation), other waiters stay wedged → reboot still needed. All source evidence points to transience; a `drgn` read of `g_RmApiLock` on the next occurrence settles it.

## Assertion shape

- **Bug present:** cold open fails at GSP boot on an eGPU → deferred-open worker parks on `rmapiLockAcquire` (osapi.c:2745) → close (`nv_wait_open_complete`) + pciehp (`nv_kthread_q_flush`) wedge → host RM-API dead, reboot-only.
- **Fix verified (A11+C6):** the same failed cold open → the worker returns (D1 bails, or COND yields), `open_complete` fires, the open_q flush drains, host alive. Regression guard: healthy opens and dGPUs never reach the modified arm (open-failure + `is_external_gpu` gated).
- **Observability:** `adapter_status` surfaced via `NV_ESC_WAIT_OPEN_COMPLETE` is best-effort only (never an internal gate).

## fake-5090 mechanism mapping

A **host-kernel lock-ordering deadlock** on the global API rwsem, seeded by a cold-init GSP-boot failure (PCIe transient) that strands a single-threaded deferred-open worker on a blocking API-lock acquire; closed structurally by the close-wait and the surprise-removal `open_q` flush both depending on that worker. Reproduce by: substrate `cold-init-fails-at-GSP-heartbeat` + a second API-lock contender + the deferred-open path; assert the worker parks at the status-read acquire and never signals `open_complete`. Discriminator: does the failed-open arm short-circuit (A11/D1 — worker returns) or block on `g_RmApiLock` (pre-A11 wedge)? Also models the C6 primitive (COND acquire truthful vs inverted).

## Source citations

- Driver: `nv.c:1781-1799` (D1), `:1997` (`complete_all`), `:2690` (`nv_wait_open_complete`); `osapi.c:2737-2754` (`rm_get_adapter_status`/A11), `:1777` (cold-init lock release); `os-interface.c:333-355` (C6); `nv-pci.c:2451` (open_q flush); `rmapi.c:572-578` (READ→WRITE), `:595` (COND), `:605` (the inversion consumer).
- Field: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/wedge-2026-06-02-coldboot-apilock-deadlock.md` (+ `-stacks.txt`, `-dmesg.txt`), `f45-deadlock-fix-design-workflow-2026-06-02.json`.
