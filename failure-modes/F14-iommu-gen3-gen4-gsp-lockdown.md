# F14 — IOMMU + Gen3↔Gen4 retraining → GSP_LOCKDOWN cascade

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — mitigated via kernel cmdline `iommu=off` and bridge `LnkCtl2` bit 5 cap)* |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #6, PM #7 (Lever H9a), PM #9 (host_reset); Lever T cmdline; bridge LnkCtl2 bit-5 cap |
| **fake-5090 mechanism** | **Real-HW only** — route to `scripts/aer-harness` on real Barlow Ridge bridge; not simulated in fake-5090 v1 |

## Symptom (driver view)

On the real eGPU host's Thunderbolt Port A, every cold-cold-boot produces a `GSP_LOCKDOWN` cascade. `dmesg` shows multiple `NVRM: ... GSP_LOCKDOWN_NOTICE` events during early boot, `rm_init_adapter` fails, `nvidia-smi -L` returns "No devices found." Port B works.

Root-cause analysis (per predecessor `iommu-gsp-lockdown-analysis.md`) is a multi-cause interaction:

1. **IOMMU rejecting DMA** from the GPU during early init (Lever T mitigation: `iommu=off` kernel cmdline parameter).
2. **PCIe link Gen3↔Gen4 retraining instability** on the Barlow Ridge TB bridge downstream port — when the link spontaneously retrains between Gen3 and Gen4 during driver init, config-read timeouts cascade into the driver classifying the GPU as PCI-not-PCIe, `rm_init` fails, GSP firmware sees host-side comm failure, and trips lockdown (Lever H17 mitigation: cap `LnkCtl2` bit 5 — Hardware Autonomous Speed Disable — on the bridge before `nvidia.ko` binds).
3. An earlier hypothesis (Cause 3 in `_predecessor.md` line 117) initially blamed Port A H9a service tightening of DevCtl2 Range B — falsified by Q2 (2026-05-08 deliberate cap removal reproduced the failure at n=1, confirming Cause 2 is load-bearing).

The interaction is *between the bridge silicon, the host firmware, the kernel's IOMMU layer, and the GSP firmware boot sequence* — none of these are inside `nvidia-driver-injector`. There's no patch in the injector for F14 because the fix lives elsewhere (kernel cmdline + an out-of-tree `aorus-egpu-bridge-link-cap` service per predecessor line 223).

## Trigger sequence

**Real-hardware only.** Reproducing F14 requires:

1. A real Barlow Ridge (Intel JHL9540 or equivalent) Thunderbolt 4 controller in the host.
2. A real NVIDIA Ada/Blackwell GPU in the eGPU enclosure.
3. A real cold-cold-boot (power off enclosure, wait, power on, observe). On Port A without the `LnkCtl2` bit 5 cap and without `iommu=off`, F14 reproduces at high rate.

The fake-5090 v1 substrate cannot reproduce F14 because:
- Modelling Gen3↔Gen4 retraining at the per-microsecond level required to trigger the cascade is out of scope for vfio-user; the kernel's PCIe retraining state machine runs against root-port silicon, not config-space simulation.
- Modelling IOMMU DMA rejection requires kernel-side cooperation that bypasses the synthetic device entirely.
- Modelling the GSP firmware's lockdown response requires actual GSP firmware running on a GPU — not present in fake-5090.

F14 therefore routes to `scripts/aer-harness` (the predecessor's real-hardware test infrastructure) for validation.

## Expected post-patch behaviour (system-level, not driver-internal)

With both mitigations applied:

1. **Lever T** (`iommu=off` kernel cmdline, or an equivalent per-device IOMMU disable): IOMMU does not reject GPU DMA during init.
2. **Lever H17 / `aorus-egpu-bridge-link-cap`**: `LnkCtl2` bit 5 (Hardware Autonomous Speed Disable) set on the TB switch downstream port *before* `nvidia.ko` binds — eliminates Gen4 retraining as a cause.

With these, cold-cold-boot succeeds; `rm_init_adapter` completes; no `GSP_LOCKDOWN_NOTICE` events in `dmesg`. The injector's own patches (C1–C5, E1, A1–A5) are *unaffected* by F14 — they're load-bearing for other failure modes but not for this one.

## Assertion shape

**Real-HW only assertions** (run via `scripts/aer-harness`, NOT under `fake-5090`):

- After cold-cold-boot of the eGPU enclosure on Port A:
  - `dmesg` MUST NOT contain `GSP_LOCKDOWN_NOTICE` events attributable to the boot window.
  - `nvidia-smi -L` returns a non-empty device list within bounded time (default: 30 s).
  - `cat /sys/bus/pci/devices/<TB_bridge_BDF>/<LnkCtl2_proxy>` shows bit 5 set (proxy path depends on how the cap service exposes its state).
  - `cat /proc/cmdline` contains `iommu=off` (or equivalent).

**Regression-detection assertion** (predecessor Q2 2026-05-08 method):

- Deliberately remove the `LnkCtl2` bit 5 cap mid-experiment (`aorus-egpu-bridge-link-cap` service stop). Cold-cold-boot. F14 SHOULD reproduce (n=1 in Q2). This is the *negative-test* path that confirms the cap is load-bearing — used during mitigation regression-testing, not in routine CI.

**fake-5090 substitution** (what fake-5090 v1 CAN assert):

- The injector's other failure modes (F1–F13) run cleanly under fake-5090 even though F14 is unreproducible. This serves as a presence-test for the boundary: fake-5090's value is exercising the 13 reproducible modes without needing real hardware, while F14 remains the documented gap covered by `scripts/aer-harness`.

## fake-5090 mechanism mapping

- **Real-HW only.** F14 lives in the test catalogue specifically to document *why* fake-5090 v1 doesn't cover it and to point at the `scripts/aer-harness` route for validation.
- A *future* fake-5090 v2 with full bridge-silicon modelling (Gen3↔Gen4 LTSSM state machine, IOMMU integration, GSP firmware emulation) could theoretically attempt F14 — but this is a multi-order-of-magnitude scope expansion. Documented but not planned for v1.

## Source citations

- aorus-5090-egpu `docs/failure-modes-index.md` row 6 (GSP_LOCKDOWN cascade at boot) and the linked `docs/iommu-gsp-lockdown-analysis.md`.
- `_predecessor.md` lines 111–127 (Symptom + Cause 1/2/3 + Verification of Q2 falsification of Cause 3).
- Predecessor archive `archive/iommu-off-test-2026-05-07-145453/` (Lever T validation).
- Predecessor archive `archive/gen3-fail-2026-05-07-165158/` (Lever H17 / bridge-link-cap validation; the H10/H16/H17 ledger entries).
- `aorus-egpu-bridge-link-cap` service (predecessor `_predecessor.md` line 223) — the operational mitigation pattern.

## Why F14 is in the index despite no injector patch

The `nvidia-driver-injector` is one layer of the defence stack. F14 is documented here so:

1. Test authors understand fake-5090 v1's coverage boundary explicitly.
2. Operators triaging real-hardware failures can map a `GSP_LOCKDOWN` symptom to a known-mitigated-elsewhere root cause without first searching the injector for a missing patch.
3. Future scoping work for fake-5090 v2 (or for an injector-side mitigation, if one becomes practical) has a starting point in the existing test corpus.
