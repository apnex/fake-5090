# F40 — chip re-init wedge on userspace-recovered eGPU (PCIe Completion Timeout race)

> **Note on the title.** The original "`RmShutdownAdapter` destructive-teardown wedge" framing was disproved by the 2026-05-29 morning/afternoon investigation, then partially re-vindicated by the evening A7 Test A n=2 validation. The picture as of 2026-05-29 22:08 is that F40 is a **two-arm failure class**:
>
> - **F40-open arm** (open path, RmInitAdapter MMIO hang) — fires on cycle-2 open of a userspace-recovered chip. F40-precondition-state-dependent. Closed by A6.
> - **F40-shutdown arm** (rmmod path, rm_shutdown_adapter MMIO hang) — fires on every healthy rmmod on this hardware. Structural, NOT F40-precondition-dependent. Closed by A7.
>
> The original "destructive teardown wedge" title turns out to describe the F40-shutdown arm correctly (rm_shutdown_adapter is INSIDE destructive teardown), even though the morning investigation thought it was only an artefact of the open-path arm. Title preserved for traceability. Body has been refreshed twice now (morning revision pinning the open-path mechanism, evening A7 Test A revision capturing the shutdown-path mechanism). Historical investigation log preserved at the bottom.

| Field | Value |
|---|---|
| **Sources** | C5 (sink primitive used by both F40b timeout paths); A4 (close-path telemetry that bounded the investigation); **A6** (F40b Tier 2 bounded-wait wrapper that structurally closes the OPEN arm); **A7** (F40b Tier 2 bounded-wait wrapper that structurally closes the SHUTDOWN arm — added 2026-05-29 evening) |
| **Confidence** | field-bug (n=13+ wedge reproductions during 2026-05-29 day); OPEN arm structurally closed via A6 at n=2 validation (F40B-TEST runs 19:48 and 19:50); SHUTDOWN arm discovered via 20:52 forensics report and structurally closed via A7 at n=2 validation (Test A runs 21:56:55 and 22:06:35, see `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/A7-test-A-validation-2026-05-29.md`) |
| **Predecessor evidence** | injector repo `docs/missions/mission-1-egpu-hot-plug-hot-power/experiments/h1-userspace-recovery-2026-05-28.md`; forensic archives `/var/log/mission-1-archaeology/wedge-2026-05-28-21-54/` through `/var/log/mission-1-archaeology/tbv2n2-wedge-2026-05-29/` (13 wedge boots, all OPEN arm); `/var/log/mission-1-archaeology/a7-deploy-wedge-2026-05-29/FORENSICS-REPORT.md` (1 SHUTDOWN arm wedge, the trigger for the A7 work) |
| **fake-5090 mechanism** | Substrate models a chip in *userspace-recovered-after-prior-bind* init state. Chip responds normally to PASSIVE MMIO probes (PMC_BOOT_0, ReBAR cap reads, AER status reads). Chip does NOT respond to the chip-init MMIO TLP that `RmInitAdapter` issues during `nv_open_device_for_nvlfp` on the second open of a fresh fd cycle. PCIe Completion Timeout fires on the root complex; AER signaling MAY race against a kernel-wide deadlock from the blocked MMIO read. Without the F40b Tier 2 wrapper, the deadlock wins frequently and the host hangs hard. |

## Current understanding (2026-05-29 evening — supersedes investigation log below)

### Mechanism (PINNED — n=13 reproductions, Test B v2 directly observed AER UESta=0x00004000)

1. **Precondition.** The chip enters a *userspace-recovered-after-prior-bind* substrate state when the boot cycle has been:
   - Prior nvidia.ko bind + persistence engagement + CUDA workload (or equivalent chip-touching activity that engages GSP), then
   - Destructive uninstall via `nv_pci_remove_helper → nv_shutdown_adapter`, then
   - TB deauth/reauth → broken-BAR1 (F41 fires), then
   - `fix-bar1.sh` userspace recovery (restores BAR1 sizing but NOT chip-internal init invariants), then
   - modprobe nvidia (without persistence engagement).

2. **Cycle 1 (first nvidia-smi -L or equivalent).** Open → MMIO → LAST-CLOSE. Close-path runs cleanly through `nv_shutdown_adapter` → WPR2 → 0. The chip remains MMIO-responsive (BAR0 reads continue to work for ≥30 sec; see BAR-test forensics).

3. **Cycle 2 (the wedge trigger).** Any chip-touching open of `/dev/nvidia0` after cycle 1's LAST-CLOSE:
   - Goes through VFS / chrdev / `nvidia_open` → foreground branch → `nv_open_device_for_nvlfp` → `RmInitAdapter`
   - `RmInitAdapter` issues an MMIO TLP to a chip register (a GSP boot / init register)
   - The chip does not produce a PCIe Completion for that TLP
   - The CPU executing the MMIO read is blocked
   - PCIe Completion Timeout (50 ms default at root complex) eventually fires
   - The AER state machine SHOULD signal `Uncorrectable (Non-Fatal) Completion Timeout` (UESta=0x00004000, bit 14)
   - The kernel SHOULD invoke C5's `pci_error_handlers.error_detected` callback, set sink-state, return ERROR_RECOVERY, and the MMIO read returns 0xFFFFFFFF for the driver to handle

4. **The race that determines outcome.** AER must complete before the kernel deadlocks on the blocked CPU:
   - **Path A (won by AER):** AER fires in time, C5 catches it, sink-set, EIO returned to userspace. Host stays alive. Observed n=1 of 12 attempts (Test B v2; bpftrace's heavy kprobe overhead provided the scheduling slack).
   - **Path B (won by deadlock):** the CPU is stuck waiting for MMIO completion AND holding kernel state that AER processing needs. Cascading deadlock locks up the whole kernel before AER can fire. Observed n=11 of 12 attempts (DIFF-3, DIFF-4, VERIFY, TBv2-n2, and the earlier reproductions). Host hard-wedges; reboot required.

5. **The chip-side root cause is OUT-OF-SCOPE for this catalog.** The chip not responding to RmInitAdapter's MMIO on a userspace-recovered substrate IS the underlying defect. We do not have visibility into the chip's silicon/firmware state; this is NVIDIA's territory (or future kernel-side TB-aware patches). What we DO control is the host-side response to the chip's misbehaviour.

### Mechanism — SHUTDOWN arm (added 2026-05-29 evening — n=2 validation, A7 Test A)

The 20:52 FORENSICS-REPORT documented a host wedge during a k8s-driven pod restart that included `rmmod nvidia → modprobe nvidia`. A6 was loaded; the wedge was on the rmmod-side, not the modprobe-side. A7 was developed to wrap the chip-touching MMIO inside `nv_shutdown_adapter` (specifically `rm_disable_adapter` and `rm_shutdown_adapter`) in the same bounded-wait pattern A6 uses on the open path.

A7 Test A n=2 (21:56:55, 22:06:35) produced a sharper finding than expected: **the rm_shutdown_adapter MMIO hang fires on every healthy rmmod on this hardware**, not just on chips already in F40-precondition state.

1. **Precondition.** None — the chip can be in any healthy state. Both test runs were on chips that had been operating cleanly with persistence engaged. No prior F40 fire, no userspace-recovered substrate, no chronic timeout symptom. Just a normal `kubectl exec entrypoint.sh uninstall` on a healthy driver.

2. **Teardown sequence inside `nv_shutdown_adapter`.**
   - `rm_disable_adapter(sp, nv)` — A7's first wrap. Chip responds within the 200 ms budget. Wrapper logs "completed within budget", returns. **Healthy.**
   - Host-side teardown — kthread stops, IRQ teardown, MSI-X mutex frees. No chip touches. **Healthy.**
   - `rm_shutdown_adapter(sp, nv)` — A7's second wrap. Chip does NOT respond within the 200 ms budget. A7's wrapper times out, sets C5 sink via `rm_cleanup_gpu_lost_state`, returns. **Structural hang.**
   - Remainder of `nv_shutdown_adapter` (FLR check, NUMA memory queue stop) — completes cleanly on the host side. `nv_pci_remove_helper` returns. rmmod exits with code 0.
   - The leaked worker (still in flight inside `rm_shutdown_adapter`'s RM closed code) eventually observes the C5 sink and exits. The refcount-2 protocol guarantees no struct leak.

3. **What's wedged in the chip.** Unknown. The MMIO that hangs inside `rm_shutdown_adapter` is RM closed code — we cannot see what register it's writing or what acknowledgement it's waiting for. Plausible candidates include GSP shutdown coordination, memory-allocator teardown that needs a chip ack, or a final state-flush register. None of this matters for A7 — the wrapper bounds the wait and lets teardown continue.

4. **What this proves.**
   - The 20:52 wedge wasn't an aberration. It was the expected outcome of running rmmod without A7 on this hardware.
   - Every aorus.<NN> uninstall before A7 landed was a near-miss against the same wedge.
   - A7 is **load-bearing for production** — not "defense before A9 lands," but the primary mechanism keeping the host alive across routine pod restarts and image upgrades.
   - The patch intent's "Scenario: rmmod on wedged chip" stays correct, but the patch intent's framing of A7 as F40-precondition-protection is too narrow. A7 protects EVERY rmmod, not just edge-case ones.

5. **What it does NOT prove.**
   - Whether the rm_shutdown_adapter hang is hardware-specific (TB4 + Blackwell-eGPU + this kernel), driver-specific (open-driver vs proprietary), or workload-history-dependent (CUDA-touched-then-rmmod vs never-touched-then-rmmod). Test A held all these constant.
   - Whether the chip would respond at 250 ms, 500 ms, or never (the worker is leaked after 200 ms; we don't observe its eventual fate beyond the sink-aware fail-fast that lets it exit).
   - Whether the close-path caller of `nv_shutdown_adapter` (`nv_stop_device → nv_shutdown_adapter` on LAST-CLOSE without persistence) also exhibits this hang. A7 wraps both callers; only the rmmod caller was tested.

6. **Difference vs the OPEN arm.** The OPEN arm requires F40-precondition (userspace-recovered chip after prior teardown + TB cycle + fix-bar1 + modprobe). The SHUTDOWN arm requires nothing — every healthy rmmod hits it. They share the same wrapper primitive (heap-allocated work, bounded-wait, C5 sink on timeout), the same C5 detector class (`NV_GPU_LOST_DETECTOR_AER_FATAL` as placeholder), and the same "leaked worker exits when MMIO fails-fast post-sink-set" exit mechanism. But they are different in trigger profile.

### Structural close (A6 + A7 — F40b Tier 2 bounded-wait wrapper, OPEN arm + SHUTDOWN arm)

**A6** (committed at `nvidia-driver-injector@c8d3c68`, validated 2026-05-29 evening at n=2 via F40B-TEST) wraps `nv_open_device_for_nvlfp` for E1-classified eGPUs in a bounded-wait pattern:

- Schedule the chip-touching init on `system_long_wq` with a refcounted heap-allocated work struct
- Syscall thread waits with timeout (`NVreg_TbEgpuOpenTimeoutMs`, default 200 ms = 4× PCIe CTO)
- On completion within timeout: propagate the worker's rc (happy path)
- On timeout: call `rm_cleanup_gpu_lost_state(sp, nv, NV_GPU_LOST_DETECTOR_AER_FATAL)` (sets C5 sink), return -EIO. Worker is leaked and exits when its in-flight MMIO fails-fast post-sink-set.

A6 makes the structurally clean outcome deterministic instead of timing-dependent. F40B-TEST n=2 confirmed at 201.7 ms and 203.5 ms wall-clock for the timeout, matching the 200 ms budget. Host stays alive; bash receives `Input/output error`; subsequent operations on the GPU see C5 sink-state and fail-fast until the chip is recovered.

**A7** (committed at `nvidia-driver-injector@429615c` as part of the F40b family v1 commit, validated 2026-05-29 evening at n=2 via A7 Test A) wraps `rm_disable_adapter` and `rm_shutdown_adapter` inside `nv_shutdown_adapter` for E1-classified eGPUs in the same bounded-wait pattern A6 uses, generalised to `void (*)(nvidia_stack_t *, nv_state_t *)` callees:

- Each chip-touching RM call (rm_disable_adapter, rm_shutdown_adapter) is wrapped independently — both go through `nv_f40b_shutdown_bounded(rm_call, sp, nv, call_name)`
- Same `system_long_wq` + refcounted heap-allocated work struct primitive as A6
- Timeout budget controlled by `NVreg_TbEgpuShutdownTimeoutMs` (default 200 ms; separate knob from the open-path budget)
- On completion: log "completed within budget", return; nv_shutdown_adapter proceeds with its next teardown step
- On timeout: set C5 sink via `rm_cleanup_gpu_lost_state`, return; nv_shutdown_adapter proceeds with its next teardown step. Leaked worker exits when its in-flight MMIO fails-fast post-sink-set.

A7 Test A n=2 (21:56:55 wall-clock 200 ms ± measurement granularity, 22:06:35 same) confirmed the timeout fires deterministically on healthy unloads. rm_disable_adapter always completes within budget; rm_shutdown_adapter always times out (on this hardware as of 2026-05-29). Host stays alive across the timeout; the leaked worker exits cleanly; subsequent rmmod completes within ~1 second wall-clock. **Without A7, every rmmod on this hardware would wedge the host** — the 20:52 forensics report attests to exactly this outcome.

### Operational recovery (current — userspace-orchestrated)

Once F40b fires, the chip is in C5 sink-set state and unusable. Recovery requires the chip's PCIe / init state to be reset. The current sequence (validated by F40B-TEST 1 → F40B-TEST 2 between-run recovery + the explicit runtime reset test at 20:02) is:

```
rmmod nvidia                                    (cleanly unloads — C5 sink → chip-touching teardown skipped)
echo 0 > /sys/bus/thunderbolt/.../authorized    (TB deauth)
echo 1 > /sys/bus/thunderbolt/.../authorized    (TB reauth — chip re-enumerates with broken-BAR1)
setpci -s 04:00.0 COMMAND=0:3                   (clear mem decoding for fix-bar1's safety check)
fix-bar1.sh                                     (write ReBAR CTRL=0x00000f21, pciehp slot cycle)
modprobe --ignore-install nvidia                (loads F40b nvidia.ko, binds)
nvidia-smi -pm 1                                (engage persistence — optional)
```

This whole sequence completes in ~17 seconds wall-clock; the host stays alive throughout. No reboot required.

### Future direction (A8 + A9 — fully in-driver target)

The current operational recovery requires userspace coordination (rmmod, TB sysfs, fix-bar1.sh, modprobe). The project target — per `feedback_native_in_driver_hardening` and the Windows TDR reference model — is to move the entire recovery sequence into the driver itself. Two future patches address this:

- **A8 — sysfs observability** (committed 2026-05-29 evening at `nvidia-driver-injector@429615c`; **A8 sysfs surface didn't materialise at runtime** despite all symbols loading — A8 used `pci_driver.driver.dev_groups` which compiles but doesn't produce sysfs entries on this driver. A8 v2 will switch to A3's `sysfs_create_group` pattern). Expose `tb_egpu_state`, `tb_egpu_f40b_fires`, `tb_egpu_recovery_count`, `tb_egpu_recovery_failures`, `tb_egpu_last_recovery_ns` as per-PCI-device sysfs attributes. Read-only; consumed by monitoring (Prometheus, nvidia-smi, ops dashboards). No event-driven uevent — observability wants state, not transitions.
- **A9 — in-driver recovery state machine** (draft only; promotion gated on A8 v2 landing). On F40b timeout (OPEN or SHUTDOWN arm), schedule a recovery worker that performs `pci_reset_bus` → TB rebind via thunderbolt kernel APIs → kernel-side ReBAR resize via `pci_resize_resource()` → re-probe → sink clear → state → healthy. No rmmod/modprobe cycle needed.

After A8+A9, the F40 wedge becomes a transient event: detection → fail-fast (EIO) → in-driver recovery (~5-10 sec) → GPU back to healthy. The injector's userspace orchestration role disappears.

Design context for the target state lives in `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/in-driver-recovery-target-2026-05-29.md`. A9 patch draft lives in `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/A9-patch-intent-draft-2026-05-29.md` (graduates to `docs/patch-intents/A9-f40b-in-driver-recovery.md` when implementation begins).

### Minimum-viable reproduction recipes (current as of 2026-05-29 evening)

**OPEN arm (F40-open) — requires F40-precondition state**

```
1. Cold boot
2. Deploy injector DS → nvidia.ko bound + persistence engaged
3. Run nvbandwidth via diag container (any CUDA workload sufficient)
4. Uninstall via injector + delete DS (rmmod nvidia + nvidia_uvm)
5. echo 0 > /sys/bus/thunderbolt/devices/0-1/authorized; sleep 2
6. echo 1 > /sys/bus/thunderbolt/devices/0-1/authorized; sleep 4
7. setpci -s 04:00.0 COMMAND=0:3
8. /root/nvidia-driver-injector/tools/fix-bar1.sh
9. echo > /sys/bus/pci/devices/0000:04:00.0/driver_override
10. modprobe --ignore-install nvidia
11. nvidia-smi -L                                    ← cycle 1, completes cleanly
12. exec 3</dev/nvidia0                              ← cycle 2
    Expected without F40b: host wedge (kernel-wide freeze, hard reboot required)
    Expected WITH A6 (current code): -EIO after ~200 ms, "Input/output error", host alive
    Expected WITH A9 (future): -EIO momentarily, in-driver recovery, retry succeeds within ~5 sec
```

**SHUTDOWN arm (F40-shutdown) — requires NO precondition**

```
1. Cold boot
2. Deploy injector DS → nvidia.ko bound + persistence engaged
3. (optionally) run any healthy workload
4. kubectl exec <injector-pod> -- /entrypoint.sh uninstall
    Expected without A7: rmmod hangs indefinitely inside rm_shutdown_adapter; host wedge during subsequent kernel work; hard reboot required (the 20:52 forensics outcome)
    Expected WITH A7 (current code): rm_disable_adapter "completed within budget"; rm_shutdown_adapter "timed out after 200 ms — declaring GPU lost"; rmmod returns 0 within ~1 sec; host alive; chip reloadable on next modprobe
    Expected WITH A9 (future): rm_shutdown_adapter timeout triggers in-driver recovery; chip back to healthy within ~5-10 sec; no rmmod/modprobe cycle needed
```

The SHUTDOWN arm recipe is **trivially reproducible on any healthy aorus.<NN> deployment** — there's no F40-precondition setup. Just uninstall a healthy driver and observe `journalctl -k -f`. A7 Test A n=2 (21:56:55 and 22:06:35) used exactly this recipe.

### Cross-references

- F40b structural close design: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/F40b-structural-fix-2026-05-29.md`
- F40b OPEN-arm implementation (A6): `nvidia-driver-injector/patches/addon/A6-f40b-bounded-wait-open.patch`
- F40b OPEN-arm patch intent: `nvidia-driver-injector/docs/patch-intents/A6-f40b-bounded-wait-open.md`
- F40b SHUTDOWN-arm implementation (A7): `nvidia-driver-injector/patches/addon/A7-f40b-bounded-wait-shutdown.patch`
- F40b SHUTDOWN-arm patch intent: `nvidia-driver-injector/docs/patch-intents/A7-f40b-bounded-wait-shutdown.md`
- A7 Test A n=2 validation report: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/A7-test-A-validation-2026-05-29.md`
- A8 sysfs observability patch (currently broken — A8 v2 forthcoming): `nvidia-driver-injector/patches/addon/A8-f40b-sysfs-observability.patch` + intent at `nvidia-driver-injector/docs/patch-intents/A8-f40b-sysfs-observability.md`
- In-driver recovery target design: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/in-driver-recovery-target-2026-05-29.md`
- A9 patch intent draft: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/A9-patch-intent-draft-2026-05-29.md` (will graduate to `docs/patch-intents/A9-f40b-in-driver-recovery.md` when implementation begins)
- 20:52 SHUTDOWN-arm forensics report: `/var/log/mission-1-archaeology/a7-deploy-wedge-2026-05-29/FORENSICS-REPORT.md`
- F41 (upstream root-cause sibling — chip ReBAR Control register reset on TB hot-add): `F41-...md`

---

# Historical investigation log (preserved 2026-05-29 evening — superseded by §Current understanding above)

The sections below preserve the day-of-investigation reasoning in chronological order. Multiple hypotheses were proposed and falsified before the mechanism was pinned. Reading them top-to-bottom recovers the intellectual path; reading just the §Current understanding above recovers the result.

## Mechanism (revised 2026-05-29 evening — wedge is in RE-INIT, not in teardown)

The 2026-05-29 evening Test #1 (2x `nvidia-smi -L` cycle with bpftrace + post-modprobe attach attempt, on userspace-recovered chip) produced a controllable, fast reproduction. Cycle 1's close-path completed cleanly through `post-shutdown` and `close-exit` with WPR2 driven to 0. Cycle 2's `nvidia-smi -L`, fired 2 seconds later, wedged the host with **zero kernel journal output after the cycle-2 kmsg start marker**. Forensic archive: `/var/log/mission-1-archaeology/c1-test1-wedge-2026-05-29/`.

**Revised mechanism (now n=4):** F40 is a **chip re-init wedge** on the OPEN path after a clean `nv_shutdown_adapter` on a chip in the userspace-recovered state. The wedge mechanism is:

1. F41 (chip ReBAR Control register reset on TB hot-add) leaves the chip in a state where some chip-internal pre-conditions for re-init are not met (the signature is `0x110094 == 0xbadf2100`, surfacing as `gpuHandleSanityCheckRegReadError_GH100` on the first MMIO).
2. The first `LAST-CLOSE` after probe runs `nv_shutdown_adapter` → `RmShutdownAdapter`'s destructive teardown (`gpuStateDestroy`, DMA teardown, BAR ioremap teardown, gpumgr detach). This step COMPLETES — A4's `post-shutdown` and `close-exit` telemetry fire, WPR2 → 0, the chip is in a "MMIO-responsive, GSP-off" state. **The destructive teardown is not the problem.**
3. The next `nv_open_device` on `/dev/nvidia0` runs the first-fd-init branch, which calls `RmInitAdapter` to re-bring the chip up. On a userspace-recovered chip, that re-init path hangs in chip-touching code that never returns — the chip's PCIe equalization / GSP-boot pre-conditions are not in the state `RmInitAdapter` expects.

**The minimum-viable trigger** is **2x `nvidia-smi -L` with 2s between them** on a userspace-recovered chip. No nvbandwidth, no CUDA, no 10-minute idle. Yesterday's C1 reproduction at 10m 41s was the same mechanism with a heavier trigger (nvbandwidth's CUDA init) and longer-than-necessary delay.

**What this changes from the original framing:**

- The original title ("`RmShutdownAdapter` destructive-teardown wedge") is now a historical misnomer. The teardown completes. Title preserved for traceability.
- The original "Symptom (driver view)" section below (claiming the wedge fires "from close-exit forward, … inside a teardown function call that never returns") is wrong. Close-exit fires. Callback-exit fires. The teardown call chain returns. The wedge fires on the SUBSEQUENT OPEN's `RmInitAdapter`.
- The original "Trigger sequence" step 6 (substrate hangs inside a destructive teardown step) is wrong. Substrate model needs to model the OPEN-path re-init hanging on the userspace-recovered chip-state, not a teardown step.
- The fake-5090 substrate spec moves accordingly: a single test cycle is open-1 → close-1 (clean) → open-2 (wedge), not open-1 → close-1 (wedge).

**Why earlier reproductions misled the framing:**

- PINPOINT-1 (this morning) showed printk going silent at `callback-exit`. The 5m 41s wait until a "user-visible wedge" was actually idle until SOME downstream subsystem made a chip-touching system call equivalent to a second open. The "5m 41s silent kernel" interpretation was wrong; the kernel was alive, the chip was alive, and the wedge only fired when something touched the chip again.
- C1 (yesterday) showed the wedge at 10m 41s on nvbandwidth start. nvbandwidth was just the second open in that case; CUDA init wasn't required, the 10-min idle wasn't required.
- Run 2 of yesterday's PINPOINT-2 (`rmmod` path wedge) was a rmmod-cycle that included a subsequent rebind / re-init. The wedge fired on the re-init half of the rmmod+modprobe cycle, not on the rmmod half.

All four reproductions are consistent with the corrected mechanism. Earlier "runtime PM" / "deferred RCU" / "kworker chip touch" candidate mechanisms are no longer needed to explain the data.

### Precondition refinement (Test #1 REDO, 2026-05-29 evening)

A second-attempt reproduction (Test #1 REDO at 12:09–12:10 UTC+10, on a fresh boot) ran the SAME minimum-viable-trigger procedure (TB deauth/reauth → fix-bar1 → modprobe → 2x nvidia-smi -L) but **did NOT wedge**. Five cycles of `nvidia-smi -L` (4 with bpftrace running, 1 with bpftrace killed for native-speed control) all completed cleanly with paired `nv_open_device` / `nv_stop_device` / `nv_shutdown_adapter` ENTER/RETURN events. Shutdown took ~620–638 ms per cycle, consistent across cycles.

**Key kernel-side difference:** the divergent-state signature `gpuHandleSanityCheckRegReadError_GH100: addr: 0x110094, regvalue: 0xbadf2100` **was NOT emitted** during the REDO. In all four prior wedge reproductions (yesterday cycle 2; PINPOINT-1 morning; C1 morning; Test #1 morning) the sentinel WAS emitted. The chip was NOT in the F40-triggering state.

**The procedural difference between the REDO boot and the four wedge boots:**

- **Wedge boots:** earlier in the same boot, the chip had been through one full production cycle — injector DS deployed, persistence engaged via `nvidia-smi -pm 1`, nvbandwidth CUDA workload run, then uninstall via `entrypoint.sh uninstall` (which rmmods nvidia + nvidia_uvm). THEN the test cycle started (TB recycle → fix-bar1 → modprobe → 2x nvidia-smi -L).
- **REDO boot:** post-boot, no injector DS deployment, no persistence engagement, no CUDA workload. The test cycle ran directly against a chip that had only ever seen its BIOS-time cold-plug and now its first TB recycle.

**Updated F40 precondition stack (n=4 wedge + n=1 no-wedge):**

1. Prior probe + persistence-engagement + CUDA workload (or equivalent chip-touching activity that engages GSP) **in the same boot**.
2. Then a destructive uninstall path that rmmods nvidia and runs `nv_pci_remove_helper` → `nv_shutdown_adapter`.
3. Then a TB deauth/reauth → broken-BAR1 (F41 fires).
4. Then `fix-bar1.sh` userspace recovery (recovers BAR1 sizing; does NOT clear chip-internal divergence).
5. Then modprobe nvidia without persistence engagement.
6. Then a first OPEN → MMIO → LAST-CLOSE cycle (close-path is safe; the gating signature `0x110094 == 0xbadf2100` becomes observable at the first MMIO).
7. Then a SECOND OPEN → wedge in `RmInitAdapter`.

Remove step (1) or (2) and the chip is not in the divergent state — `0x110094` reads a normal value, no wedge fires. **TB recycle + fix-bar1 alone is not sufficient.** The chip needs prior persistence/CUDA-bound activity that left chip-internal init state pointed at a sub-state where the post-recovery re-init hangs.

**The `0x110094 == 0xbadf2100` sentinel is therefore the canonical precondition canary.** It is necessary AND sufficient (per n=4 + n=1 evidence so far) as a probe-time predictor of the wedge.

**For F40b Tier 1 (probe-time poison flag): this strengthens the design's confidence.** The signature isn't just a candidate canary — it is the failure mode's gating signal. Read `0x110094` at probe time; if it returns the sentinel, the chip is in the F40-triggering state and the poison flag must be set; if it reads normally, the chip is safe and the flag should not be set. Variant A (refuse all re-init when flagged) is the right default. The detector has high signal-to-noise.

**For the fake-5090 substrate spec:** the substrate's "chip init state" parameter has at least three values now, not two:

- `cold-boot-equivalent` — fresh BIOS-POST state, no prior driver activity. `0x110094` reads normally. No wedge from any subsequent close+open cycle.
- `userspace-recovered-fresh` — chip went through TB recycle + fix-bar1 BUT was not bound to nvidia.ko + persistence-engaged earlier in this lifetime. `0x110094` reads normally. **The previously-conjectured F40 precondition state is NOT this; this state is safe.** Test #1 REDO is the n=1 evidence point.
- `userspace-recovered-after-prior-bind` — chip went through nvidia.ko probe + persistence + CUDA workload + destructive uninstall, then TB recycle + fix-bar1. `0x110094` reads as `0xbadf2100`. This is the actual F40 state. n=4 evidence.

The substrate's `0x110094` read MUST return `0xbadf2100` for the wedge to fire; the wedge MUST NOT fire if it reads normally. This is empirically tight.

### Outstanding to-do: validate full reproduction with reading the canary at every state

A confirmation run is queued: deploy injector → engage persistence → run nvbandwidth → uninstall → TB recycle → fix-bar1 → BEFORE modprobe, dump BAR0+0x110094 → modprobe → during cycle 1, observe `gpuHandleSanityCheck` warning OR explicitly read 0x110094 → 2x nvidia-smi -L → expect wedge on cycle 2. This run validates the precondition stack end-to-end and provides one more data point on the sentinel-signature reliability. Reboot cost: 1 (the wedge will fire).

### Test #1 FULLPRE result (2026-05-29 evening — partial falsification)

The full-precondition reproduction ran and wedged on cycle 2 as predicted (**n=5 wedge confirmed**, plus the n=1 no-wedge from Test #1 REDO without prior persistence). The precondition stack as documented above (steps 1–7) IS empirically necessary.

**BUT — two prior claims in this section are now partially falsified:**

1. **The `0x110094 == 0xbadf2100` sentinel is NOT necessary.** This run wedged without emitting the sentinel anywhere in the wedge boot's kernel journal. The earlier "necessary AND sufficient" framing is retracted. Updated assessment: the sentinel correlates with the wedge (n=4 of 5) but does NOT gate it. As a single-register canary for an F40b probe-time poison flag, it would catch most wedges but miss this one. See `feedback_single_datapoint_inferential_overreach_2026_05_26` — the missing cell (sentinel-absent-with-wedge) was exactly what we hadn't tested before claiming sufficiency, and today's run produced it.

2. **The wedge is NOT in `nv_open_device` / `RmInitAdapter`.** bpftrace (post-modprobe attach, 8 traceable kprobes attached) captured cycle 1 completely — 5 ENTER / 5 RETURN matched, including `nv_shutdown_adapter` at LAST-CLOSE — and captured **ZERO** cycle 2 events. The wedge fired before `nv_open_device` was ever called. The wedge site is in the syscall path BEFORE `nvidia_open` queues work to `nv_open_q`.

**New leading hypothesis: PCI runtime-PM resume.** The gap between cycle 1 RETURN (12:20:18) and cycle 2 start (12:21:16) was 58 seconds — longer than Linux's default 5-sec auto-suspend window. The GPU likely auto-suspended to D3hot during the idle. Cycle 2's `open()` syscall would trigger `pci_pm_runtime_resume` BEFORE any nvidia.ko file_operations callback runs. On a chip whose D3→D0 PCIe link retrain / GSP-state restoration can't complete from the userspace-recovered state, that resume hangs synchronously in the PM core. This is testable with a single differential experiment: re-run the precondition sequence with `power/control=on` on the GPU + audio function before cycle 1. If 5 cycles survive, runtime-PM resume is the site.

Forensics: `/var/log/mission-1-archaeology/c1-test1-fullpre-wedge-2026-05-29/FORENSICS-REPORT.md`.

### Mechanism pinned, but AER+C5 path is NOT reliable (2026-05-29 evening, Test B v2 + VERIFY + TBv2-n2)

**The F40 wedge mechanism IS a PCIe Completion Timeout AER race during chip re-init MMIO** — confirmed by Test B v2's direct observation of `UESta=0x00004000` (CTO bit) and the C5 state machine output. But this race is genuinely non-deterministic and the AER+C5 path catches the failure only intermittently. n=2 reproducibility check (TBv2-n2) wedged with identical setup as Test B v2's EIO.

**Three adjacent runs investigated this:**

**Test B v2 (cycle 1 + cycle 2 + bpftrace running with 47 probes)** — cycle 2 reached `nvidia_open`'s `foreground-pre-init`, called `nv_open_device_for_nvlfp`, issued MMIO to the chip. ~1 second later, kernel reported:

```
pcieport 0000:00:07.0: AER: Uncorrectable (Non-Fatal) error message received from 0000:04:00.0
tb_egpu recover: AER error_detected fired on 0000:04:00.0 (channel state=1)
                  AER UESta=0x00004000  ← bit 14 = Completion Timeout (CTO)
tb_egpu recover: error_detected -> NEED_RESET (scheduling bus reset; attempts=1/3)
tb_egpu recover: error_detected -> DISCONNECT (rate-limited (H2); attempts=1/3)
tb_egpu recover: trigger gated (sink-set: GPU already declared lost (C5 sink)); emitting PERMANENT_FAIL
NVRM: tb_egpu [OPEN]: site=foreground-post-init pid=40328 rc=-5
NVRM: tb_egpu [OPEN]: site=exit pid=40328 rc=-5
```

C5 caught the AER, set sink-state, returned -EIO. Host stayed alive. Bash got `Input/output error`.

**VERIFY (same sequence, no bpftrace)** — cycle 2 wedged. NO PINPOINT-3 markers fired for cycle 2 (the kernel froze before `nvidia_open`'s `entry` marker could fire). NO AER message. NO C5 recovery output. The host froze hard, required reboot.

**Mechanism narrative**:

1. cycle 2 bash `exec 3</dev/nvidia0` → kernel open-syscall path → `nvidia_open` → foreground branch → `nv_open_device_for_nvlfp` → `nv_open_device` → `RmInitAdapter` issues an MMIO TLP to a chip register.
2. The chip (in userspace-recovered post-shutdown state) does not produce a completion for the TLP.
3. The PCIe root complex's Completion Timeout timer starts (50 ms default).
4. The CPU executing the MMIO read is blocked waiting for completion data.
5. THE RACE: when CTO fires, an AER interrupt is delivered. The AER handler (tb_egpu_aer_handler / C5 recovery) is supposed to be invoked on a different CPU, set sink-state, and abort the MMIO. Whether this races against a kernel-wide deadlock determines the outcome:
   - bpftrace running (47 kprobes in the path): kprobe overhead provides scheduling slack. AER handler runs in time. C5 catches, returns -EIO.
   - no bpftrace: MMIO blocking + scheduler critical sections + lock acquisition order races deadlock the kernel before AER handler can run. Host wedges.

**TBv2-n2 (cycle 1 + cycle 2 + IDENTICAL 47 bpftrace probes as Test B v2)** — same setup as Test B v2, intended as the n=2 reproducibility check. Outcome: WEDGE. No PINPOINT-3 markers fired for cycle 2; no AER message; no C5 recovery output. Same wedge pattern as VERIFY. Test B v2's EIO was thus n=1 of 2 attempts — NOT a reliable structural outcome.

**Cumulative result:**

| Test | Outcome |
|---|---|
| Test B v2 (47 probes) | -EIO via AER+C5 (n=1 outlier) |
| VERIFY (no bpftrace) | wedge |
| TBv2-n2 (47 probes — repro of Test B v2) | wedge |

n=2 of 3 wedge, n=1 of 3 -EIO. The mechanism is the same race in all three; the kernel-side resolution varies.

**Implications for fix path**: C5's AER recovery infrastructure IS correct in design (Test B v2 proved it can handle the failure cleanly). But AER processing doesn't always fire in time before the kernel deadlocks. The structural fix is therefore F40b Tier 2 — a bounded-wait wrapper around `nv_open_device_for_nvlfp` that forces a deterministic timeout. The bounded-wait wrapper produces the same end-state (sink-set + EIO) regardless of whether AER fires naturally, decoupling the fix from the race timing.

Test B v2 is **evidence the C5 machinery works when given the chance**; it is **not evidence that AER fires reliably enough to give it the chance.**

### Structural close VALIDATED (2026-05-29 evening, F40B-TEST n=2)

The A6 patch (`patches/addon/A6-f40b-bounded-wait-open.patch`, committed at `nvidia-driver-injector@c8d3c68`) implements the bounded-wait wrapper. Two consecutive test cycles confirmed the structural close works:

**F40B-TEST run #1 (19:48):**
- Cycle 1 (nvidia-smi -L): all 3 fd opens completed within 200 ms budget, rc=0; LAST-CLOSE close-path ran cleanly with WPR2 → 0.
- Cycle 2 (bash `exec 3</dev/nvidia0`): bounded worker scheduled, MMIO hung inside RmInitAdapter, syscall thread woke from timeout at **201.7 ms** (matches the 200 ms budget). F40b code called `rm_cleanup_gpu_lost_state(sp, nv, NV_GPU_LOST_DETECTOR_AER_FATAL)`. C5 sink propagated, emitted `PERMANENT_FAIL`. Open returned `-EIO`. Bash reported `Input/output error`. Host stayed alive.

**F40B-TEST run #2 (19:50):**
- Same precondition (rmmod + TB recycle + fix-bar1 + modprobe). Same cycle 1 + cycle 2 sequence.
- Cycle 2 timed out at **203.5 ms**. Same F40b log sequence. Same -EIO outcome. Host stayed alive.

Both runs produced identical kernel markers:

```
NVRM: tb_egpu [F40b]: open scheduled to bounded worker (timeout=200 ms)
NVRM: tb_egpu [F40b]: open completed within budget rc=0     ← cycle 1 opens
NVRM: tb_egpu [F40b]: open scheduled to bounded worker (timeout=200 ms)
NVRM: tb_egpu [F40b]: open timed out after 200 ms — declaring GPU lost ...
NVRM: GPU 0000:04:00.0: tb_egpu recover: trigger gated (sink-set: GPU already declared lost (C5 sink)); emitting PERMANENT_FAIL
```

**The F40 wedge class is now structurally closed.** The fix is deterministic (200 ms upper bound on the wait, configurable via `NVreg_TbEgpuOpenTimeoutMs`), does not depend on AER firing in time, and produces clean -EIO to userspace with the chip declared lost via C5's sink mechanism. The leaked worker thread is bounded in lifetime (it exits when its MMIO fails-fast post-sink-set).

Forensics: F40B-TEST runs live in the host journal at 19:48–19:50 on 2026-05-29 (not archived — this is the SUCCESS case, no wedge to forensic).

Implementation: `patches/addon/A6-f40b-bounded-wait-open.patch` in the nvidia-driver-injector repo.

Forensics: `/var/log/mission-1-archaeology/verify-wedge-2026-05-29/FORENSICS-REPORT.md`, `/var/log/mission-1-archaeology/tbv2n2-wedge-2026-05-29/FORENSICS-REPORT.md` (this round); `/var/log/mission-1-archaeology/diff3-wedge-2026-05-29/FORENSICS-REPORT.md` and onward (preceding investigation).

### Implications for F40b architecture (replaces the older "Three-layer architecture" section below)

The F40b structural fix design at `nvidia-driver-injector/docs/missions/.../design/F40b-structural-fix-2026-05-29.md` needs to re-cut the wrap site:

- **OLD (now invalid):** bounded-wait wrapper around `nv_shutdown_adapter`, PCIe link-disable fallback on shutdown hang, `DETECTOR_TEARDOWN_TIMEOUT`.
- **NEW (informed by Test #1 data):** the cheapest correct fix is a **probe-time poison flag** keyed on the divergent-state signature (`0x110094 == 0xbadf2100` on first MMIO, or per a more reliable indicator if one emerges from further characterization). When the flag is set, the next `nv_open_device` after a `LAST-CLOSE` returns `-EIO` instead of running `RmInitAdapter`. Userspace gets an explicit "chip needs PCI re-enum" signal; the kernel never enters the hanging code path. This dodges the wedge by refusing to enter the hanging operation; no bounded-wait worker required.
- **NEW (alternative — bounded wait if poison-flag detection is unreliable):** wrap `nv_open_device`'s first-fd-init branch (entry to `RmInitAdapter`) in the bounded-wait pattern; on timeout, `pci_dev_set_disconnected` + PCIe link-disable + return `-EIO` from open. New C5 detector class: `DETECTOR_REINIT_TIMEOUT`.

F40a (probe-time persistence engagement) is still valuable, but for a different reason than the original design said: persistence keeps `usage_count > 0` so LAST-CLOSE never fires `nv_shutdown_adapter`, which means the chip never enters the post-shutdown state, which means the wedge-triggering re-init path is never reached. F40a doesn't "detect and prevent" the wedge — it stays out of the state machine where the wedge lives.

### Reproduction recipe (canonical, n=4, 2026-05-29 evening)

```
1. Get chip into userspace-recovered state:
   a. echo 0 > /sys/bus/thunderbolt/devices/0-1/authorized    # TB deauth
   b. sleep 2
   c. echo 1 > /sys/bus/thunderbolt/devices/0-1/authorized    # TB reauth → broken-BAR1 (256 MiB)
   d. sleep 4
   e. setpci -s 04:00.0 COMMAND=0:3                           # clear mem decoding (fix-bar1 safety check)
   f. /root/nvidia-driver-injector/tools/fix-bar1.sh          # recover BAR1 to 32 GiB

2. Bind driver (NO persistence):
   a. echo > /sys/bus/pci/devices/0000:04:00.0/driver_override   # fix-bar1 leaves this "none"
   b. modprobe --ignore-install nvidia                           # host policy blocks plain modprobe

3. Reproduce:
   a. nvidia-smi -L      # cycle 1 — close-path completes; LAST-CLOSE drives WPR2 → 0
   b. sleep 2
   c. nvidia-smi -L      # cycle 2 — wedges immediately on RmInitAdapter

Expected: cycle 2 never returns. Host journal stops at cycle-2 start kmsg marker.
Recovery: hard reboot.
```

Two gotchas required for reproducibility on this hardware that the original F40 entry did not surface:
- `fix-bar1.sh` sets `driver_override=none` and does not clear it. Modprobe (with or without `--ignore-install`) silently no-ops in that state ("The NVIDIA probe routine was not called for 1 device(s)"). Cure: `echo > driver_override` before modprobe.
- Host has `install nvidia /bin/false` in `/etc/modprobe.d/nvidia-driver-injector.conf` (the container-only-load policy). Cure: `modprobe --ignore-install nvidia` or run from inside the injector container.

## Symptom (driver view) — original framing, see §Mechanism for corrected understanding

After a non-cold-boot init path leaves the chip in a state where PCIe equalization (Phy16Sta `EquComplete+`/Phase1/2/3 bits) was not run to completion — observed on this project after the userspace H1 recovery sequence (chip ReBAR Control rewrite + pciehp slot cycle, per F41) — the chip presents as healthy by every passive probe:

- `lspci -s <BDF> -vvv` shows speed/width at Gen3 x4 with no error flags.
- BAR0 MMIO reads return plausible values; PMC_BOOT_0 reads the chip's identity register correctly.
- `nvidia.ko` `probe()` succeeds; GSP firmware loads; WPR2 is set normally (`0x07f4a000`).
- `nvidia-smi -L` and other metadata queries succeed.

The first `LAST-CLOSE` after probe (e.g. on `nvidia-smi -L` exit) enters `nvidia_close_callback → nv_close_device → nv_stop_device`. Without persistence mode engaged, `nv_stop_device` takes the non-persistent branch:

```c
nv_acpi_unregister_notifier(nvl);
nv_shutdown_adapter(sp, nv, nvl);              // -> RmShutdownAdapter
tb_egpu_close_diag(nvl, "post-shutdown", ...);  // A4 telemetry: fires OK, WPR2 cleared
```

A4's four close-path telemetry sites (close-entry, pre-stop, post-shutdown, close-exit) all fire successfully — each reading PMC_BOOT_0 cleanly from BAR0. The chip is responding to passive MMIO at every site.

After `close-exit` fires, control returns up the kernel stack:

- `nv_free_file_private` runs (memory cleanup).
- `bRemove = (!surprise_removal) && (usage_count==0) && rm_get_device_remove_flag(...)` is evaluated.
- If `bRemove==true`, `pci_stop_and_remove_bus_device(nvl->pci_dev)` is invoked.
- Async power-state transition completes from the `rm_unref_dynamic_power(... NV_DYNAMIC_PM_COARSE)` call inside `nv_stop_device`.

From here, the host enters a silent, system-wide wedge: kernel journal goes dark for non-NVIDIA events too (systemd timers stop firing, etc.). No Xid, no AER, no hung-task warning, no panic, no oops. The only recovery is reboot.

The failure surface is structurally distinct from F12 (close-path → next-open hang; chip-state mismatch surfaces on subsequent open), F18 (`RmShutdownAdapter` assert on `NV_ERR_GPU_IS_LOST` return), F22 (workload silent hang via forward-progress stall), F26 (DRM teardown hang during surprise removal). F40's preconditions: chip alive at every detection site, no sentinel returns, no GPU-lost classification possible — the wedge is in a chip-touching teardown step that never returns and never produces a status the existing detectors could short-circuit on.

The chip-side delta between `RmShutdownAdapter` and `RmDisableAdapter` identifies the candidate wedge sites. `RmDisableAdapter` (persistence-engaged path) performs only: interrupt disable + client-resource cleanup + RC-timer stop + `gpuStateUnload`. `RmShutdownAdapter` adds: `RmDestroyPowerManagement` + `gpuStateDestroy` + DCE firmware shutdown + `gpumgrDetachGpu` + `gpumgrDestroyDevice` + `RmTeardownDeviceDma` + `RmClearPrivateState` + `RmUnInitAcpiMethods` + `RmTeardownRegisters`. Of these, `gpuStateDestroy`, `RmTeardownDeviceDma`, and `RmTeardownRegisters` perform chip-side operations that depend on the chip's internal power-management / DMA-controller / register-mapping state being internally consistent. F40's wedge is in one of those three steps (pinpoint TBD by fine-grained telemetry — task #252 in the injector project).

## Trigger sequence

1. Daemon brings device through a non-cold-boot init path that leaves chip's PCIe equalization status bits clear in `Phy16Sta` and LTR latencies = 0 (per F41 — the chip ReBAR Control register reset on TB hot-add, plus a recovery sequence that rewrites the register and triggers pciehp slot-cycle re-enumeration; equivalent chip-state divergences exist for other recovery paths that don't include a full chip reset).
2. Substrate-side chip presents as healthy to every passive MMIO read:
   - `PMC_BOOT_0` returns plausible chip-identity value (`0x1b2000a1` for Blackwell).
   - ReBAR Control register (extended cap offset `0x13c` for this chip) returns the value the driver wrote at probe time.
   - AER status registers return clean (no errors).
3. `nvidia.ko` probe runs: COMMAND register transitions from `0x0000` to `0x0007` (IO + Memory + BusMaster enabled); driver succeeds; nvidia-nvlink core init line emitted; "external GPU detected" line emitted (E1's `os_pci_is_thunderbolt_attached` returned true).
4. Test issues `nvidia-smi -L` (or any query that opens then closes `/dev/nvidiactl` without engaging persistence).
5. Substrate-side: `LAST-CLOSE` close-path fires through A4's four telemetry sites successfully. PMC_BOOT_0 reads `0x1b2000a1` at each site. Post-shutdown clears WPR2 to `0` (because `nv_shutdown_adapter` ran `RmShutdownAdapter`). close-exit fires with WPR2=0 and PMC_BOOT_0 still readable.
6. From close-exit forward, substrate behaviour for chip-touching teardown operations diverges from cold-boot-chip behaviour: at least one of `gpuStateDestroy` / `RmTeardownDeviceDma` / `RmTeardownRegisters` performs a chip-side operation that never returns. Substrate models this as "operation X enters, never produces completion, no error signal." Kernel journal silence ensues; no further dmesg lines from any subsystem.

The reproduction MUST be run twice — once with the chip in **cold-plug state** (the comparator, where the same close-path completes successfully n=13+ times in the predecessor experiment with no wedge) and once with the chip in **userspace-recovered state** (where the same close-path wedges on first `LAST-CLOSE`). The wedge is a function of chip internal state, not of any per-process or per-FD state.

## Expected post-patch behaviour

F40 is currently **uncovered by the injector patch set** — C5's close-path coverage assumes a chip-died failure model (a detector fires on an MMIO-dead signal, sink-state is set, cleanup runs in lost-aware mode). On F40 no detector has a signal to fire on. The injector's responses are:

1. **Operator mitigation (n=2 confirmed on real hardware 2026-05-28):** engage persistence mode (`nvidia-smi -pm 1` or `nvidia-persistenced`) immediately after `modprobe nvidia` and before any other open/close. The persistence flag routes `nv_stop_device` through `rm_disable_adapter` (lighter teardown) which skips the chip-touching destructive steps. GSP stays loaded across close; WPR2 stays at `0x07f4a000`; the wedge is bypassed. The injector container's entrypoint already does this — that's why production binds-via-injector have never hit F40. The host-side `tools/fix-bar1.sh --bind` script in the injector repo now also engages persistence after modprobe.

2. **Driver-side hardening (candidate work, not yet patched):** any of —
   - **A new pre-shutdown detector class** that catches "chip-responsive-but-internally-incomplete" before `RmShutdownAdapter`'s destructive steps run. The signature is chip-side register state divergence (Phy16Sta bits clear, LTR=0) plus a positive E1 external-GPU classification. The sink primitive would route through `cleanupGpuLostStateAtomic` with a new detector class (e.g. `DETECTOR_INCOMPLETE_INIT`) and the close-path would complete cleanly through the existing lost-aware funnels.
   - **Step-level hardening** inside `RmShutdownAdapter` — wrap `gpuStateDestroy` / `RmTeardownDeviceDma` / `RmTeardownRegisters` in bounded timeouts that abort cleanly on unresponsive chip ops, falling back to the same lost-state path.
   - **External-GPU policy default** — for E1-classified external GPUs, default the close path to `rm_disable_adapter` regardless of `NV_FLAG_PERSISTENT_SW_STATE`. Trades the "deprecated persistence mode" requirement for a transport-class policy.
   - **Chip-state-aware recovery** at the H1 fix layer — replicate enough BIOS-equivalent chip-side equalization that F41's recovery sequence produces a chip whose `RmShutdownAdapter` expectations hold. Likely infeasible from userspace per project investigation 2026-05-28; would need to happen in the kernel TB hot-add path simultaneously with the F41 (E27) fix.

The injector's F40 design owner is C5; the design boundary is currently scoped to the chip-died case. F40 documents the **out-of-scope class** that's been empirically validated against, with the persistence-mode operator mitigation as the current production answer.

## Assertion shape

**Trigger reproduction (substrate-side, in userspace-recovered chip-state):**

- Trigger sequence step 5: after `LAST-CLOSE`, the substrate MUST emit A4's four close-path telemetry lines (close-entry / pre-stop / post-shutdown / close-exit) with PMC_BOOT_0 returning the chip-identity value at each site, WPR2 transitioning from set to cleared at post-shutdown, and the `(LAST-CLOSE)` annotation on the post-shutdown and close-exit sites.
- Trigger sequence step 6: after close-exit, the substrate MUST stop responding to chip-touching teardown calls. Specifically: the substrate's mock implementations of the kernel function call chain `pci_stop_and_remove_bus_device → nvidia.ko's pci remove callback → RmShutdownAdapter`'s destructive operations MUST hang inside one of those calls without returning. The test harness's wall-clock must exceed a configurable wedge-timeout (default 30 s) before declaring the wedge reproduced.

**Negative assertions (what F40 is NOT):**

- `dmesg | grep -cE 'NVRM: Xid'` during the wedge MUST be 0 (no Xid fires — this is what distinguishes F40 from F20 / F21 / F23 / F39).
- `dmesg | grep -cE 'AER:'` during the wedge MUST be 0 except for the C4-injected probe-time "AER: unmasked Uncorrectable Internal Error at probe" line, which is a hardening artefact not an error.
- `dmesg | grep -cE 'NVRM: GPU.*lost via detector_class='` during the wedge MUST be 0 (no detector fires; this is what distinguishes F40 from the entire chip-died failure class F3 / F4 / F15 / F16 / F17 / F18 / F20 / F22).
- `dmesg | grep -cE 'NV_ASSERT_FAILED'` during the wedge MUST be 0 (no assertion fires; this distinguishes F40 from F15 / F18 / F28 specifically).
- `dmesg | grep -cE 'hung_task'` during the wedge SHOULD be 0 within the host's hung-task timeout window (default 120 s on Fedora kernels). The wedge hangs the entire kernel scheduler in this test's observation, including the hung-task detector itself.

**Persistence-mode-prevention assertion (operator mitigation, regression-witness):**

- Run trigger sequence twice. Run A: NO persistence engaged; substrate-side wedge MUST reproduce.
- Run B: substrate models the persistence ioctl as setting `NV_FLAG_PERSISTENT_SW_STATE`. After `modprobe nvidia`, test invokes `nvidia-smi -pm 1`-equivalent before any `nvidia-smi -L`. Substrate then routes `nv_stop_device` through the `rm_disable_adapter`-equivalent code path (no chip-touching destructive teardown). `LAST-CLOSE` MUST complete cleanly with WPR2 STAYING at `0x07f4a000` at the substrate's "post-shutdown" telemetry site (the persistence path leaves GSP loaded).
- Run B's success demonstrates the prevention is correct; Run A's wedge demonstrates the failure mode exists.

**A4 telemetry visibility (must-have, not nice-to-have):**

- A4's four telemetry sites MUST emit even when the wedge is imminent. The post-shutdown line specifically MUST capture the wedge-precursor state (chip still responding to PMC_BOOT_0 read; WPR2 cleared). The close-exit line MUST emit immediately after, also with PMC_BOOT_0 readable. Diffing these against the cold-plug-comparator's close-exit (in the comparator run, the host stays responsive after close-exit) is the diagnostic handle.

**Cross-reference assertion (to F41):**

- F40's substrate state requires F41's chip-side ReBAR Control register reset behaviour to have been triggered first (or an equivalent non-BIOS-POST init path). A test that runs F40's trigger sequence on a cold-plug-state substrate MUST NOT reproduce the wedge; this confirms F40's chip-state-dependent nature and prevents misclassifying any close-path wedge as F40.

## Reproducibility (revised 2026-05-29 — wedge IS deterministic; earlier "no wedge" interpretation was wrong)

**The F40 wedge is reproducible under the documented trigger sequence. n=2 confirmed.**

### What the PINPOINT-1 instrumented re-test produced (2026-05-29 09:13–09:19)

```
[CLOSE]:    site=close-entry     usage_count=1 (LAST-CLOSE)  WPR2=0x07f4a000 wpr2_up:YES
[CLOSE]:    site=pre-stop        usage_count=1 (LAST-CLOSE)  WPR2=0x07f4a000 wpr2_up:YES
[CLOSE]:    site=post-shutdown   usage_count=0 (LAST-CLOSE)  WPR2=0x00000000 wpr2_up:no
[CLOSE]:    site=close-exit      usage_count=0 (LAST-CLOSE)  WPR2=0x00000000 wpr2_up:no
[POSTCLOSE]: site=post-close-exit bRemove=0     ← bRemove is FALSE
[POSTCLOSE]: site=post-free-private
[POSTCLOSE]: site=path-B-post-unlock-ldata      ← Path B
[POSTCLOSE]: site=pre-kmem-free                 ← skipped pci_stop_and_remove (bRemove=0)
[POSTCLOSE]: site=callback-exit                 ← function returned cleanly
─── kernel printk goes SILENT at this point ───
~5min 41s of userspace-only activity (systemd timers, k3s, audit)
─── eventually some downstream subsystem needed the wedged kernel state and hung ───
Manual reboot required
```

### Implications — what the PINPOINT-1 data proves

1. **`pci_stop_and_remove_bus_device` is RULED OUT as the wedge mechanism.** `bRemove == 0` consistently on both healthy and userspace-recovered chips on this hardware (kallsyms-confirmed). `pci_stop_and_remove_bus_device` is never called from `nvidia_close_callback` in either case. The "smoking-gun" candidate identified in the F40 v1 trigger sequence is not actually reachable.

2. **The wedge is OUTSIDE `nvidia_close_callback`.** Every PINPOINT-1 marker through `callback-exit` fires successfully. The function returns. So does `nv_stop_device`, `nv_close_device`, etc. Whatever wedges happens AFTER the close callback returns.

3. **The wedge is NOT immediately observable.** Userspace (`nvidia-smi`, the test bash, `/dev/kmsg` writes, systemd timers, k3s, audit) all continued normally for ~5 minutes 41 seconds after the close-path completed. The earlier 2026-05-29 morning interpretation — "the bash continued and reported success, ergo no wedge" — was wrong. The wedge was already present; it just had no immediate observable. Future PINPOINT-class experiments MUST instrument AFTER `callback-exit` (kernel-side; not reachable from NVIDIA-driver patches alone — needs bpftrace / ftrace / kprobes).

### Candidate wedge mechanisms (now narrowed)

The wedge fires somewhere AFTER `nvidia_close_callback` returns. Candidates:

- **Async runtime-PM transition.** `rm_unref_dynamic_power(NV_DYNAMIC_PM_COARSE)` schedules a deferred power-state change to D3. The actual chip-side transition can happen on a kernel workqueue some time later. On a userspace-recovered chip in incomplete-init state, that PM transition could hang.
- **Deferred RCU callbacks** queued by the close path.
- **Kernel workqueue items** that touch the chip (TB tunnel poll, AER scan).
- **VFS / process fd cleanup** post-callback.

The 5-min-41s delay between close completion and user-visible wedge strongly suggests **deferred work scheduled by close-path**, not an immediate hang.

### Falsification experiments — runtime PM hypothesis is PARTIALLY confirmed but INSUFFICIENT

Three diagnostics rode one reboot cycle 2026-05-29 09:13–09:58. Results:

**Run 1 — PINPOINT-2 + bpftrace + `power/control=on` (runtime-PM-disable)**

Sequence: cold-plug → TB deauth/reauth → fix-bar1.sh (no --bind) → set `power/control=on` on 04:00.0 + 04:00.1 → modprobe nvidia → nvidia-smi -L → wait.

Result: **Host alive at 6m 30s after `nvidia-smi -L` returned.** Kernel printk responsive throughout. All A4 + PINPOINT-1 + PINPOINT-2 markers fired through `callback-exit` and `exit`. bpftrace counts:
- pci_pm_runtime_suspend: **0 calls** (runtime PM correctly disabled)
- pm_runtime_work: 164 cycles, all matched ENTER/RETURN
- No nvidia.ko deferred work captured

This is suggestive but **not conclusive** evidence for the runtime-PM-suspend hypothesis. Yesterday's wedge fired at 5m 41s; our 6m 30s observation barely cleared that threshold. We cannot rule out that the wedge would have fired had we waited longer.

**Run 2 — restore-attempt wedge (same boot, ~30 min later)**

Sequence: after Run 1's observation window, restore production state: `echo > unbind` GPU + `rmmod nvidia_uvm` (already gone) + `rmmod nvidia` + apply DSes with production aorus.17 image. **The host wedged silently 5+ minutes into this sequence.** Forensics: kernel printk went silent between my `rmmod nvidia` (~09:59) and the eventual "nvidia-nvlink: Unregistered Nvlink Core" message at 10:04:46 (5+ min gap). Reboot required.

Implications:
1. `power/control=on` does NOT prevent the wedge on the `unbind`/`rmmod` code path. The rmmod path goes through `nv_pci_remove_helper → nv_shutdown_adapter` directly, bypassing the `nv_stop_device` persistence check AND apparently the runtime-PM-disable mitigation.
2. The runtime PM hypothesis is at best PARTIAL. Either runtime PM is one of multiple async wedge mechanisms, or the actual mechanism is something `power/control=on` happens to slow down without preventing.
3. The "Run 1 host survived 6m+" result alone is not strong evidence — the restore wedge in Run 2 (same boot, same chip state lineage) demonstrates that the wedge can fire and that no sysfs-level intervention we've tried fully prevents it.

**Production mitigation remains persistence engagement (n=3).** Confirmed across cycle-1 healthy use (2026-05-28 evening), cycle-2 wedge-recovery + bind (2026-05-28), today's cold-boot injector startup (2026-05-29), and today's post-restore re-bind. Persistence works because it routes the `nv_stop_device` close-path through `rm_disable_adapter` (lighter teardown, no destructive RmShutdownAdapter call). The unbind/rmmod path that wedged in Run 2 is NOT taken in normal production flow — production uses the injector's `uninstall` subcommand or just leaves the driver loaded.

### Open questions

- **Why did Run 1 survive but Run 2 wedge** under similar chip-state conditions (both with chip in userspace-recovered state)? Hypothesis: cumulative state from multiple destructive teardowns weakens the chip's response capacity; or the unbind path's specific code differs from the LAST-CLOSE path's specific code in a wedge-relevant way.
- **What kernel-side function actually hangs?** bpftrace was attached during Run 1 (survived) but not during Run 2 (wedged). Need to instrument Run 2's pattern specifically.
- **Why no SHUTDOWN markers fired during Run 2's rmmod?** Either the rmmod path doesn't go through nv_shutdown_adapter (contradicting the source code), OR printk was already wedged when nv_shutdown_adapter ran. The latter would mean the wedge is BEFORE nv_shutdown_adapter on the rmmod path — a different code site than the LAST-CLOSE path.

Catalog confidence remains field-bug n=3 (yesterday cycle 2; this morning Run 1's apparent survival was a false negative; this morning Run 2's restore attempt). The wedge mechanism is still partially unknown.

## Refined architecture goal (2026-05-29 evening — design-phase decision)

The project's earlier framing ("persistence engagement prevents the wedge; that's the production answer") is operationally true but architecturally insufficient. F40 represents a failure CLASS — destructive teardown hanging on a chip in a state the driver doesn't fully model — that should be closed STRUCTURALLY in the NVIDIA driver fork, not merely dodged in production via the persistence policy.

### Reference model — Windows driver

The Windows NVIDIA driver presumably handles this class correctly. The Windows OS model includes:
- Bounded driver-callback contracts (the OS gives the driver N ms to release HW, then force-removes)
- Explicit chip-state validation at each step (sentinel reads, abort-early on `0xFFFFFFFF`)
- PCIe link-level fallback when the driver can't make progress

Linux PCI driver model has none of this by default — `device_release_driver` waits as long as the driver's `remove` callback takes, MMIO reads block the CPU until PCIe completion timeout. We need to build these protections INTO our nvidia.ko fork.

### Principle: tear down what needs tearing down — make it cancellable

`RmShutdownAdapter`'s extra work is correct and required at rmmod time:
- DMA mappings MUST be undone (else IOMMU/kernel VA leak)
- BAR ioremaps MUST be undone (else kernel VA leak)
- `gpumgrDetachGpu` MUST run (else next probe sees stale state)
- `gpuStateDestroy` MUST free RM-side structures (else permanent leak per device-bind)

Skipping these via "always use `rm_disable_adapter` on rmmod path" was incorrect — it leaks resources and leaves the GPU manager registry corrupt. The right fix is to make each destructive step ABORT-SAFE.

### Three-layer architecture (F40b)

1. **Bounded-wait wrapper** around destructive teardown — schedule the call on a worker thread; main thread waits with a configurable timeout. If the worker hangs in MMIO (the F40 case), the main thread proceeds to escalation rather than freezing the host.
2. **PCIe link-disable fallback** — on timeout, write the `PCI_EXP_LNKCTL_LD` bit on the parent bridge. This is the kernel's own mechanism for surprise removal. After link disable, MMIO to the device fails fast (CRS or `0xFFFFFFFF`) instead of hanging.
3. **C5 sink-state escalation with new detector class** — fire `cleanupGpuLostStateAtomic(pGpu, DETECTOR_TEARDOWN_TIMEOUT)` to mark the GPU as lost. Subsequent cleanup runs in "lost mode" via C5's existing sink-state awareness, doing only kernel-side resource work (DMA unmap, iounmap, gpumgr detach) — no further chip touches.

### Decoupling from E27 (the F41 kernel patch)

E27 fixes F41 (chip ReBAR Control reset on TB hot-add) at the kernel level. F40b fixes F40 at the driver level. They are INDEPENDENT — either alone closes the user-visible failure for the TB-eGPU case:

| Failure scenario | E27 alone | F40b alone | Both |
|---|---|---|---|
| TB deauth → broken-BAR1 (F41) | ✓ (no broken-BAR1) | ✗ (still need `fix-bar1.sh`) | ✓ |
| F40 close-path wedge | ✓ (precondition absent) | ✓ (bounded wait + sink) | ✓ |
| F40 rmmod-path wedge | ✓ (precondition absent) | ✓ (bounded wait + sink) | ✓ |
| Chip-state divergence from other causes | ✗ | ✓ (detector fires regardless) | ✓ |

F40b's structural closure is broader than E27's indirect coverage. Even if some future code path or hardware variant produces a similar chip-state divergence, F40b's detector fires and the cleanup completes safely. This is the architectural value of fixing in-driver vs. only in-kernel.

### Status

Design phase — currently pending characterization tests (PINPOINT-2 + bpftrace with wedge fire, rmmod-path on cold-plug chip, persistence+uninstall on recovered chip) to inform the bounded-wait timeout values, validate the link-disable fallback works on TB-tunneled bridges, and confirm whether rmmod-path wedge is the same mechanism as the close-path wedge or a separate failure mode.

Canonical design doc: `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/design/F40b-structural-fix-2026-05-29.md`.

## fake-5090 mechanism mapping

**Substrate-side hybrid state machine.** This is a more elaborate substrate model than F22's "wedge-without-signal" because the substrate must:
1. Continue presenting all passive MMIO probes normally (PMC_BOOT_0, AER, ReBAR — these are read by A4 telemetry and by RM at probe).
2. Continue responding to GSP firmware load and the WPR2 register transitions during `RmShutdownAdapter`'s teardown sequence — the post-shutdown telemetry must see WPR2 cleared, not stuck.
3. Withhold completion on one specific chip-touching operation inside one of the post-shutdown destructive steps. Which step is the wedge target is a substrate configuration parameter (the test can set it to "wedge in gpuStateDestroy", "wedge in RmTeardownDeviceDma", or "wedge in RmTeardownRegisters") — F40's pinpoint task in the injector project will determine which is the production-evidence target.

**Chip-state divergence model.** Substrate exposes a "chip init state" parameter with at least two values: `cold-boot-equivalent` (full BIOS POST chip-side state — Phy16Sta `EquComplete+`, LTR latencies populated, lane-equalization presets at cold-boot values) and `userspace-recovered` (the chip-state observed after F41's recovery sequence — Phy16Sta clear, LTR=0, lane-equalization presets at slot-cycle-equivalent values). The wedge is reproducible only with the `userspace-recovered` parameter; the comparator (cold-boot-equivalent) is the n=13+ no-wedge demonstration.

**Telemetry exposure.** Substrate exposes A4's four telemetry sites with their canonical log strings. The test's primary diagnostic handle is the "what was logged last before the wedge began" question; the substrate's job is to make that question answerable.

**Out of scope for fake-5090 v1:** The chip-side ReBAR Control register reset and the corresponding chip-state divergence is the F41 substrate (see F41 mechanism mapping). F40 layers ON TOP of F41 — F41 produces the chip-state divergence, F40 demonstrates that the close path wedges on that state. A test of F40 alone (assuming F41's substrate state) is reasonable; F41's substrate state can be modelled as a substrate configuration parameter.

## Source citations

- `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/experiments/h1-userspace-recovery-2026-05-28.md` §"Cycle 2 wedge — close-path hazard after userspace recovery" §"Chip-state divergence — passive register dump" §"Prevention — persistence mode (confirmed n=2)" §"Close-path coverage analysis (post-mortem)"
- Forensic archive `/var/log/mission-1-archaeology/wedge-2026-05-28-21-54/` — prior-boot journal showing wedge moment; `chip-state-diff-2026-05-28/` — passive register diff between cold-plug and userspace-recovered
- `kernel-open/nvidia/nv.c:2105` — `nv_stop_device` body, the `if (PERSISTENT_SW_STATE) rm_disable_adapter else nv_shutdown_adapter` branch
- `src/nvidia/arch/nvalloc/unix/src/osinit.c:2370` — `RmShutdownAdapter` body (destructive teardown sequence)
- `src/nvidia/arch/nvalloc/unix/src/osinit.c:2514` — `RmDisableAdapter` body (persistence-path lighter teardown)
- C5 `cascade-class-design-v4.md` — design coverage scope (chip-died failure model); F40 is the documented out-of-scope class
- A4 `nv-tb-egpu-close.c` — passive close-path telemetry; ruled out as wedge mechanism

## Cross-references

- [[F12]] — `/dev/nvidia0` close-path wedge (Problem 2). Different shape (close → next-open hang). F40 is wedge ON close, not on subsequent open.
- [[F15]] — `NV_ASSERT_OR_GPU_LOST` macro under-applied. Same chain (`nv_pci_remove → rm_shutdown_adapter → RmShutdownAdapter`), different failure (assertion fire vs. step hang).
- [[F18]] — `RmShutdownAdapter` assert at `osinit.c:2462`. Sibling on same chain; fires on `NV_ERR_GPU_IS_LOST` from `gpuStateDestroy`. F40 fires when `gpuStateDestroy` (or successor step) doesn't RETURN at all.
- [[F22]] — Silent hard hang under workload. Different trigger (workload not teardown); A2's Q-watchdog catches it. F40 has no forward-progress signal to catch because it's a teardown wedge, not a workload wedge.
- [[F26]] — `nvidia-drm` teardown hang. Surprise-removal-triggered; assumes lost GPU. F40 has no surprise-removal trigger.
- [[F38]] — Recovery-primitive gap. Documentation-class. Distantly related (recovery scope boundaries).
- [[F41]] — Chip ReBAR Control register reset on TB hot-add. F41 produces the chip-state divergence that F40 wedges on. F40 requires F41 (or equivalent) as precondition.
