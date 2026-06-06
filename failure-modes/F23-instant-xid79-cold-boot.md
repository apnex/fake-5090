# F23 — Instant Xid 79 on cold-boot — eGPU not yet usable, fires before sustained-load class

| Field | Value |
|---|---|
| **Sources** | A3 (recovery) — adjacent; C5 (sink-state primitive) — partially covers |
| **Confidence** | community-evidence |
| **Predecessor evidence** | NVIDIA issue #1151 (2026-05-15; RTX 5090, vanilla 580.95.05, kernel 6.6 / 6.16; **instant Xid 79 at cold-boot**, GPU never reaches usable state) |
| **fake-5090 mechanism** | Topology surprise primitive — substrate appears at PCI hierarchy but transitions to "bus-loss" state during `rm_init_adapter` (between PCI probe acceptance and first GSP RPC), producing Xid 79 before any workload runs |

## Symptom (driver view)

Xid 79 ("GPU has fallen off the bus") is the canonical bus-loss-during-operation Xid; the F1/F4/F9 class deals with the steady-state version of this signal. F23 is the **cold-boot variant**: Xid 79 fires before the GPU ever reaches a usable state. The driver enumerates the device, begins `rm_init_adapter`, and either inside `rm_init_adapter` or shortly after its completion observes the GPU has fallen off — without any operator-side action, without any sustained workload, with a freshly-booted host.

The #1151 reporter's case: vanilla driver 580.95.05 on a current 5090, two distinct kernels (6.6 and 6.16), every boot. The GPU is never usable. `nvidia-smi -L` returns "No devices found" or an Xid-79-stuck device. `modprobe -r nvidia ; modprobe nvidia` does not recover; only a full host reboot helps, and the next boot reproduces.

This is structurally distinct from F14 (IOMMU + Gen3↔Gen4 retraining → GSP_LOCKDOWN — a different cascade signature, GSP_LOCKDOWN_NOTICE rather than Xid 79, and a known-mitigated-via-cmdline class). F23's signature is "Xid 79 with no preceding GSP_LOCKDOWN, no preceding link-retraining drama in `dmesg`, no obvious AER signal, no operator-visible cause." The cascade-class-design-v4 analysis classifies #1151 as a hardware-class signal (perhaps a defective 5090 sample, perhaps a sample-population-level issue) that the injector cannot fix but for which the patches' detection paths should fail fast and cleanly rather than retry-storm.

The relevant C5/A3 behaviour: when Xid 79 fires inside `rm_init_adapter`, the v3+v4 sink primitive should be set (via the funnel-1 MMIO path if the signal source is a sentinel read, OR via the C5 v4 detector [c] hook if the source is a GSP timeout). A3's post-rmInit-FAIL recovery WOULD attempt up to 3 reset attempts; without the H1 cap (which IS present in A3 v1), this would be a recovery-storm root. With H1 cap + H3 persistent kill, A3's behaviour is: 3 attempts → all fail → mark persistent → never auto-retry on this boot. The user is informed cleanly; the cascade does not amplify.

## Trigger sequence

1. Substrate is configured with "broken-at-init" mode: the device probes successfully (PCI enumeration completes, BAR resources allocated, `pci_enable_device` succeeds), but transitions to bus-loss state (MMIO reads return `0xFFFFFFFF`) at a configurable point during `rm_init_adapter` or shortly after.
2. Host boots; nvidia.ko binds; `nv_pci_probe` runs; `nv_open_device` invoked; `RmInitAdapter` runs.
3. At the configured point, substrate transitions to bus-loss. Driver's MMIO funnel detects sentinel reads OR the GSP RPC times out, depending on the configured trigger point.
4. Detector fires; sink-state set; `rm_init_adapter` returns failure; A3's post-rmInit-FAIL recovery triggers.
5. A3 attempts reset; substrate's reset behaviour configured to fail (the test scenario assumes the chip is genuinely broken, so resets should not produce a working device). A3's H1 cap fires after 3 attempts; A3's H3 persistent-kill marks the device.
6. User sees Xid 79 in dmesg, sees the persistent-kill marker via A4's telemetry, gets a clean "this GPU is dead, no further attempts will be made this boot" signal.

## Expected post-patch behaviour

**The injector cannot recover a genuinely broken cold-boot GPU.** What it CAN provide:

1. **Fail-fast detection.** Xid 79 at cold-boot trips the same C5 v4 sink primitive as steady-state Xid 79. No 45 s GSP timeouts compounding; no MMIO completion timeouts compounding; the cascade is contained at the first signal.
2. **Bounded retry behaviour.** A3's H1 cap (3 attempts in v1) ensures the host does not spin attempting recovery forever. A3's H3 persistent-kill marker prevents `udev`/`nvidia-persistenced`/operator scripts from re-triggering the cascade by re-probing the device.
3. **Clean operator signal.** A4's close-path telemetry surfaces the persistent-kill state via sysfs (per A4's design); operator tooling can read "this GPU is in cold-boot-broken-persistent state" and respond appropriately (page the user, route workloads to other GPUs, etc.).
4. **No host wedge.** Other GPUs in the system (if any) remain fully functional — no F19 cross-contamination as long as F19's follow-up F1 patch is also applied; pre-F1, F23 inherits F19's contamination behaviour on multi-GPU systems.

## Assertion shape

**Cold-boot detection (after substrate broken-at-init transition):**

- The MMIO funnel OR GSP timeout detector (whichever the substrate triggers) MUST fire within bounded time after `rm_init_adapter` is invoked.
- `dmesg | grep -cE 'NVRM: Xid.*79'` SHOULD be ≥ 1 and ≤ (H1 cap + 1) — the cap permits one Xid 79 per retry attempt plus the initial fire.

**A3 bounded-retry verification:**

- `dmesg | grep -cE 'A3.*recovery.*attempt'` (or A3's canonical retry-attempt log) MUST be ≤ 3 (H1 cap).
- `dmesg | grep -cE 'A3.*persistent.*kill'` (or A3's H3 canonical-log) MUST be ≥ 1 after the 3rd failed attempt.

**Persistent-kill regression-witness:**

- After A3 marks persistent, ANY subsequent `nv_pci_probe` invocation (simulated by triggering `udev` re-probe via test harness) MUST return without re-entering `rm_init_adapter`.
- `dmesg | grep -cE 'A3.*recovery.*attempt'` after the initial 3 MUST NOT increase.
- Per-device sysfs path (A4 telemetry surface) MUST expose persistent-kill state — verifiable via `cat /sys/bus/pci/devices/<BDF>/<A4-state-path>` (exact path TBD by A4's intent-doc).

**Clean teardown:**

- Total wall-clock time from substrate broken-at-init transition to host settled in persistent-kill state MUST be bounded (target: ≤ 30 s — 3 × A3-retry-window + cleanup).
- `nv_pci_remove` (if invoked) completes within ≤500 ms.
- Subsequent `nvidia-smi -L` returns the persistent-kill state cleanly (does not hang, does not segfault).

**Multi-GPU isolation (regression-witness for F19 follow-up F1):**

- Two-device substrate: GPU-A broken-at-init, GPU-B healthy.
- Pre-F1: GPU-B may show contamination (currently expected; documents F19's gap).
- Post-F1: GPU-B MUST report healthy via `nvidia-smi -i <GPU-B>` throughout.

## fake-5090 mechanism mapping

- **Phase 2 topology surprise primitive.** Substrate must support "device appears at PCI probe but transitions to bus-loss at a configurable point inside `rm_init_adapter`." Configurable trigger points: (a) immediately after `pci_enable_device`, (b) at first MMIO write, (c) at first GSP RPC submission, (d) at a specific GSP boot phase. Different trigger points exercise different detector inputs (MMIO funnel vs. GSP RPC timeout vs. heartbeat timeout).
- **Reset-behaviour configuration.** Substrate must support "reset attempts succeed at producing config-space recovery but the resulting device returns to bus-loss state immediately on first MMIO." This emulates a genuinely broken chip where reset is mechanically possible but not therapeutically useful.
- **Persistent-kill observability.** The substrate does not need to model the persistent-kill state itself — it's a host-side construct. The substrate only needs to keep returning bus-loss after every reset; the host-side A3 patch maintains the persistent-kill flag.
- **No new primitive** for F23 versus existing F03/F04 surprise-removal A — the substrate primitive is the same; the test orchestration (fire surprise-removal during `rm_init_adapter` instead of post-init) and assertion shape (cold-boot rather than runtime) are what's distinct.

## Source citations

- NVIDIA issue #1151 — RTX 5090, vanilla 580.95.05, kernel 6.6 / 6.16, instant Xid 79 at cold-boot
- `_community-signal.md` §"#1151 — RTX 5090: instant Xid 79 at cold-boot (eGPU never usable)" (tagged as hardware-class; classifies as fail-fast detection target rather than recoverable cascade)
- `A3-recovery.md` §"H1 cap (3 attempts max)" and §"H3 persistent-kill marker"
- `cascade-class-design-v4.md` §"DETECTION INPUTS" §"What v4 does NOT cover" (hardware-broken cold-boot cases are unrecoverable)
- F10 (recovery storm) — F23 is the "bounded version" of F10's failure mode when the underlying cause is unrecoverable; F10 exists because A3 v0 lacked H1/H3, F23 exists because even with H1/H3 the underlying chip class is real and needs a documented test path
- F14 (IOMMU/Gen3↔Gen4 → GSP_LOCKDOWN) — explicit contrast: F14 has a different cascade signature and is operator-fixable via cmdline; F23 is the same-symptom-different-class case
