# F40 — `RmShutdownAdapter` destructive-teardown wedge on chip-responsive but incompletely-initialized state

| Field | Value |
|---|---|
| **Sources** | C5 (close-path coverage scope boundary — surfaced by this case); A4 (close-path telemetry — the four-site instrumentation that bounded the wedge to a post-`close-exit` region) |
| **Confidence** | field-bug (confirmed n=2 — see Reproducibility section. Initial 2026-05-29 morning downgrade to "hypothesis" was retracted later same day on careful re-read of the prior-boot journal: the PINPOINT-1 re-test DID wedge — kernel printk went silent at 09:14:01 immediately after the last marker but bash/systemd kept running for ~5 min until something downstream needed the wedged subsystem. The "did not wedge" interpretation was wrong.) |
| **Predecessor evidence** | nvidia-driver-injector `docs/missions/mission-1-egpu-hot-plug-hot-power/experiments/h1-userspace-recovery-2026-05-28.md` (cycle 2 wedge, 21:54 UTC+10); forensic archive `/var/log/mission-1-archaeology/wedge-2026-05-28-21-54/`. **PINPOINT-1 instrumented re-test 2026-05-29 09:13–09:19 UTC+10 reproduced the wedge with full telemetry** — forensic archive `/var/log/mission-1-archaeology/pinpoint1-wedge-2026-05-29/`. |
| **fake-5090 mechanism** | Hybrid substrate state — chip responds normally to passive MMIO probes (PMC_BOOT_0, ReBAR cap reads, AER status reads) AND GSP firmware loads OK at probe AND nominal close-path telemetry succeeds — but the chip's PCIe equalization and LTR state are divergent from cold-boot (Phy16Sta `EquComplete-`, LTR latencies = 0, lane-equalization presets in transient state). Substrate honours all driver work UP TO `nv_shutdown_adapter`/post-shutdown telemetry; from there forward, one of `gpuStateDestroy` / `RmTeardownDeviceDma` / `RmTeardownRegisters` performs chip-touching operations that depend on the missing init state and silently hangs. No Xid, no AER, no sentinel return — the wedge is in-step inside a teardown function call that never returns. |

## Symptom (driver view)

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

### Falsification + further-narrowing experiments queued

- **Disable runtime PM on the device** (`echo on > /sys/bus/pci/devices/0000:04:00.0/power/control`) before the wedge cycle. If wedge prevented: runtime-PM hypothesis confirmed.
- **bpftrace** attached to `pm_runtime_work` / `pci_pm_runtime_*` / `workqueue:workqueue_execute_*` during the wedge window. Last function entered without exit identifies the kernel-side wedge site.
- **PINPOINT-2** with markers inside `nv_shutdown_adapter` (between `rm_disable_adapter` / `nv_kthread_q_stop` / `free_irq` / `rm_shutdown_adapter`) to verify each synchronous step actually completes (cross-check against the implicit assumption that A4's `post-shutdown` log implies all of nv_shutdown_adapter completed).

These three diagnostics can ride one reboot cycle. The persistence-mode mitigation remains the empirically valid production answer (n=2) — the catalog confidence is now restored to field-bug.

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
