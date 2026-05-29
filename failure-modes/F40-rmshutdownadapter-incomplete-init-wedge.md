# F40 — `RmShutdownAdapter` destructive-teardown wedge on chip-responsive but incompletely-initialized state

| Field | Value |
|---|---|
| **Sources** | C5 (close-path coverage scope boundary — surfaced by this case); A4 (close-path telemetry — the four-site instrumentation that bounded the wedge to a post-`close-exit` region) |
| **Confidence** | field-bug (confirmed **n=4**, see Reproducibility section). Re-cuts and corrections to original framing tracked in §"Mechanism (revised 2026-05-29 evening — wedge is in RE-INIT, not in teardown)" below — most important: the entry's TITLE is preserved for historical traceability but is now a misnomer. The wedge fires on the FIRST OPEN AFTER a clean LAST-CLOSE, not in `RmShutdownAdapter` itself. |
| **Predecessor evidence** | nvidia-driver-injector `docs/missions/mission-1-egpu-hot-plug-hot-power/experiments/h1-userspace-recovery-2026-05-28.md` (cycle 2 wedge, 21:54 UTC+10); forensic archive `/var/log/mission-1-archaeology/wedge-2026-05-28-21-54/`. **PINPOINT-1 instrumented re-test 2026-05-29 09:13–09:19 UTC+10 reproduced the wedge with full telemetry** — forensic archive `/var/log/mission-1-archaeology/pinpoint1-wedge-2026-05-29/`. |
| **fake-5090 mechanism** | Hybrid substrate state — chip responds normally to passive MMIO probes (PMC_BOOT_0, ReBAR cap reads, AER status reads) AND GSP firmware loads OK at probe AND nominal close-path telemetry succeeds — but the chip's PCIe equalization and LTR state are divergent from cold-boot (Phy16Sta `EquComplete-`, LTR latencies = 0, lane-equalization presets in transient state). Substrate honours all driver work UP TO `nv_shutdown_adapter`/post-shutdown telemetry; from there forward, one of `gpuStateDestroy` / `RmTeardownDeviceDma` / `RmTeardownRegisters` performs chip-touching operations that depend on the missing init state and silently hangs. No Xid, no AER, no sentinel return — the wedge is in-step inside a teardown function call that never returns. |

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
