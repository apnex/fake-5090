# F41 — Chip ReBAR Control register reset on TB hot-add — H1 broken-BAR1 root cause

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — kernel TB hot-add code path; E27 candidate)* |
| **Confidence** | field-bug |
| **Predecessor evidence** | nvidia-driver-injector `docs/missions/mission-1-egpu-hot-plug-hot-power/experiments/h1-userspace-recovery-2026-05-28.md` §"Observation 1 — chip CTRL register state after TB deauth/reauth"; passive register captures `/var/log/mission-1-archaeology/chip-state-diff-2026-05-28/` |
| **fake-5090 mechanism** | **Bridge-class + chip-side state.** Substrate must model the chip's PCIe Resizable BAR Control register (`PCI_REBAR_CTRL` at extended-cap offset +0x08) as resettable on TB tunnel teardown. Realistic substrate exposes the chip's ReBAR cap with reset-default `BAR_SIZE=0x8` (256MB advertisement) and tracks a "TB tunnel teardown event" that triggers a chip-side reset of the CTRL register's BAR_SIZE field back to default. |

## Symptom (driver view)

A Blackwell GPU (RTX 5090 / RTX PRO 6000) hot-attached via Thunderbolt or USB4 comes up at BAR1=256MB instead of the chip's maximum 32GB advertisement, even though `pci=hpmemsize=64G` and `pci=realloc=on` are set on the kernel cmdline.

`lspci -vvv` shows:

- `Capabilities: [134 v1] Physical Resizable BAR / BAR 1: current size: 256MB, supported: 64MB 128MB 256MB 512MB 1GB 2GB 4GB 8GB 16GB 32GB`
- `Region 1: Memory at 0x4000000000 (64-bit, prefetchable) [size=256M]`

The chip's ReBAR Capability register (`PCI_REBAR_CAP` at cap-offset +0x04) correctly advertises support for 32GB. The chip's ReBAR Control register (`PCI_REBAR_CTRL` at cap-offset +0x08) reads `0x00000821` — BAR_IDX=1 (BAR1), NBAR=1, BAR_SIZE=8 → `2^(8+20)` = 256MB. This is the chip's reset-default advertisement, NOT what the Linux kernel ever wrote, NOT what the prior healthy cold-plug had (which was `0x00000f21` — BAR_SIZE=15 → 32GB).

The root cause is **NOT** kernel bridge-window stickiness (the bridge windows resize correctly via pciehp's `pci_assign_unassigned_bridge_resources()` once the chip advertises 32GB — confirmed in the project predecessor). The root cause is the kernel's TB hot-add code path never writes the chip's ReBAR Control register to the maximum supported size. BIOS does this at cold-boot via SMM/UEFI runtime services as part of POST. TB hot-add lives entirely in the kernel runtime path with no equivalent step.

Comparison data from the predecessor: single passive read of `setpci -s <BDF> 0x13c.l` shows:
- Pre-deauth (healthy cold-plug):  `0x00000f21` → BAR_SIZE=15 → 32GB
- Post-deauth/reauth (broken-BAR1): `0x00000821` → BAR_SIZE=8  → 256MB

The chip register reset on TB tunnel teardown is the H1 root cause in a single passive read. The kernel's enumeration code, the bridge windows, and the realloc heuristics are all behaving correctly given the inputs they see from the chip; the chip is just telling them it needs 256MB.

This is structurally distinct from F30 (bridge-window stickiness). F30 frames the failure as bridge-side — the bridge cold-boot-time window is too small and realloc cannot grow it. F41 reframes the same observable symptom (BAR1 unassigned or undersized) as chip-side — the chip's BAR advertisement is wrong. On hardware where BOTH failure modes can manifest, the operator-visible symptom is identical (BAR1 not at full size); the distinction matters because F30's mitigation (`pci=hpmemsize=64G`) does not help F41 (the chip is advertising 256MB, so the kernel sizes the bridge for 256MB regardless of the cmdline-reserved 64GB headroom). F41's mitigation requires either a kernel patch on the TB hot-add path or a userspace recovery sequence that rewrites the chip's ReBAR Control register and re-enumerates the bridge.

The predecessor demonstrated a userspace recovery: with the GPU unbound, memory decoding disabled, and `driver_override=none` locking out auto-bind, the operator writes `setpci -s <BDF> 0x13c.l=0x00000f21` to advertise 32GB, then issues a pciehp slot power cycle to trigger bridge re-allocation through the same algorithm cold-boot uses. The chip's ReBAR Control register write persists across the slot cycle (verified n=2); BAR1 comes up at 32GB. The recovery is shipped in the project as `tools/fix-bar1.sh`. The recovery surfaces F40 (the close-path wedge on incompletely-initialized chip) as a downstream complication; F41 itself is the BAR-allocation root cause.

## Trigger sequence

1. Daemon brings the substrate device to the **cold-plug state** — chip's `PCI_REBAR_CTRL` BAR_SIZE field reads `0xF` (15 → 32GB). Substrate must model BIOS POST as having written this value during the daemon's substrate-init sequence (or equivalently, expose a substrate-init knob that sets the post-POST register state).
2. Test brings the device through enumeration. Kernel walks the PCI hierarchy, reads BAR1 size from the chip, sees 32GB requirement, sizes the bridge accordingly. BAR1 is assigned at 32GB. Healthy state confirmed.
3. Test issues a **TB tunnel teardown event** — substrate's option set includes: TB cable deauthorize (`echo 0 > /sys/bus/thunderbolt/devices/<UUID>/authorized`), chassis power-off + power-on, physical cable yank + replug. Substrate models all of these as chip power-cycle equivalents.
4. Substrate-side: on the TB tunnel teardown event, the chip's `PCI_REBAR_CTRL` BAR_SIZE field is reset to the post-power-on default (`0x8` → 256MB). This is the failure-mode injection point.
5. Test re-authorizes / re-establishes the TB tunnel. Kernel's TB hot-add code path walks the new PCI subordinate, reads BAR1 size from the chip, sees 256MB requirement, sizes the bridge accordingly. BAR1 is assigned at 256MB. Broken-BAR1 state.
6. Test reads `setpci -s <BDF> 0x13c.l` — substrate MUST return `0x00000821` (or whatever the substrate's default value is, as long as it is NOT the maximum 0x00000f21).
7. Test reads `/sys/bus/pci/devices/<BDF>/resource` — substrate MUST report BAR1 size of 256MB at line 2 (resource1).
8. Recovery validation: test executes the recovery sequence (write chip CTRL register to 0xF21, slot-cycle the parent bridge). Substrate MUST honour the register write (the CTRL bits are software-writeable per PCIe spec). Substrate MUST honour the slot-cycle re-enumeration with the new chip advertisement. After the slot cycle, BAR1 MUST be assigned at 32GB.

## Expected post-patch behaviour

**No injector patch addresses F41 directly.** The proper fix is a kernel patch (project work-tracker name: **E27**) that calls `pci_rebar_set_size(pdev, bar_idx, pci_rebar_get_max_size(pdev, bar_idx))` for every ReBAR-capable BAR on a device that has just been added via the TB hot-add code path. The target file is `drivers/thunderbolt/tunnel.c` (TB tunnel commit handler) or `drivers/pci/probe.c` (the runtime hot-add path that should mirror cold-boot enumeration semantics for chip-side ReBAR setup). Both `pci_rebar_get_max_size` and `pci_rebar_set_size` already exist in `drivers/pci/rebar.c` and are `EXPORT_SYMBOL_GPL`.

**E27 scope sharpening** (per F40 cross-reference): calling `pci_rebar_set_size` is necessary but may not be sufficient. F41's recovery sequence leaves the chip in an incompletely-initialized state per F40 — Phy16Sta equalization bits clear, LTR latencies = 0, etc. The kernel TB hot-add path additionally needs to either trigger a more complete equalization handshake OR co-exist with the persistence-mode policy that prevents F40's close-path wedge. The kernel patch likely needs to include both the chip-register-write step AND a policy decision about whether to mark TB-attached devices as persistence-default. This may require coordination with NVIDIA's userspace persistence-mode model.

**Operator mitigations until E27 lands:**
1. The injector's `tools/fix-bar1.sh` userspace recovery script (verified n=2). Requires consumer quiescence + nvidia.ko unbind + chip CTRL register write + slot cycle + persistence engagement before any subsequent open.
2. Avoid TB tunnel teardown events in production (do not deauth, do not chassis-power-cycle, do not yank cable). This requires operational discipline around TB cables and chassis power, plus host-reboot recovery if a teardown occurs.

## Assertion shape

**Trigger-side reproduction (substrate-side):**

- After trigger step 4 (TB tunnel teardown event): substrate MUST reset chip CTRL register to default. Test asserts `setpci -s <BDF> 0x13c.l` returns the substrate's default value (typically `0x00000821` for chips modelled after the 5090).
- After trigger step 5 (TB tunnel re-establish): test asserts BAR1 size at sysfs `/sys/bus/pci/devices/<BDF>/resource` line 2 reflects the chip's current advertisement (typically 256MB if the substrate models the 5090 reset-default).
- After trigger step 8 (recovery): test asserts BAR1 size at sysfs reflects 32GB. Test asserts chip CTRL register reads the post-recovery value (`0x00000f21`).

**Cmdline-fix-failure-witness (negative assertion against F30 mitigation):**

- With kernel cmdline `pci=hpmemsize=64G` in effect, run trigger sequence 1-7. BAR1 size MUST still be 256MB after step 7 (the F30 mitigation does NOT fix F41 because the chip is advertising 256MB and the kernel sizes bridges for the chip's advertised size, regardless of the cmdline-reserved hot-plug headroom).
- This negative assertion distinguishes F41 from F30 and is the diagnostic handle a future operator can use to determine which failure mode is active on their hardware.

**Recovery-validation assertion:**

- After the recovery sequence (substrate models the userspace recovery path from the project), the chip's CTRL register persists at `0x00000f21` across the pciehp slot cycle. Substrate MUST NOT reset the CTRL register on slot cycle (slot cycle is a pciehp event, not a chip power-cycle; only the substrate-modeled "TB tunnel teardown event" resets CTRL).
- After slot cycle, the kernel's bridge re-allocation through `pci_assign_unassigned_bridge_resources` honours the chip's new 32GB advertisement. BAR1 = 32GB at sysfs.

**E27-correctness-assertion (post-patch behaviour):**

- With the proposed E27 patch applied, run trigger sequence 1-5 WITHOUT the recovery sequence. The TB hot-add code path MUST issue a chip-side write equivalent to `pci_rebar_set_size(pdev, 1, 0xF)` as part of its enumeration. Substrate observes this register write and updates its internal state. Subsequent kernel BAR-sizing reads the chip's 32GB advertisement; bridge windows are sized for 32GB; BAR1 = 32GB at sysfs.
- Pre-patch reference (regression-witness): without E27, the same trigger sequence produces BAR1 = 256MB.

**Cross-reference assertion (to F40):**

- F41's recovery sequence produces a chip-state divergence per F40 (Phy16Sta `EquComplete-`, LTR=0, etc.). A test that runs F41's recovery AND then exercises F40's trigger sequence (probe + first LAST-CLOSE without persistence) SHOULD observe F40's wedge (in a future fake-5090 version that models F40). This linkage is what motivates F40's "F41 (or equivalent) as precondition" cross-reference.

## fake-5090 mechanism mapping

**Chip-side ReBAR register state.** Substrate must expose the chip's `PCI_REBAR_CTRL` register at extended-cap offset +0x08 (the cap header itself is at +0x00 with ID 0x0015 = PCI_EXT_CAP_ID_REBAR; CAP register at +0x04; CTRL register at +0x08). The CTRL register's BAR_SIZE field (bits[13:8]) is software-writeable per PCIe spec. Substrate MUST honour writes from `setpci`/config-space; substrate MUST report the written value on subsequent reads UNTIL a substrate-modeled chip power-cycle event occurs.

**TB tunnel teardown event.** Substrate must model at least one of: TB cable deauthorize (sysfs `/sys/bus/thunderbolt/devices/<UUID>/authorized` writes), chassis power-off + power-on, physical cable yank + replug. All three are functionally equivalent at the chip level — they remove power from the chip's PCIe layer and re-establish it. On any of these events, substrate MUST reset chip CTRL register's BAR_SIZE field to the substrate-configured power-on default (typically 0x8 for chips modelled after the 5090, but this is per-chip).

**pciehp slot-cycle event.** Distinct from TB tunnel teardown. pciehp slot-cycle goes through the kernel's `/sys/bus/pci/slots/<N>/power` interface and triggers `pciehp_configure_device` + `pci_bus_add_devices` from the pciehp hot-plug code path. This is NOT a chip power-cycle — substrate MUST NOT reset the chip CTRL register on slot-cycle events. This distinction is what makes the F41 recovery possible: the operator writes the CTRL register, slot-cycles to trigger bridge re-allocation, and the chip retains the new advertisement.

**Real-hardware-only flag.** This substrate model is implementable in fake-5090 v1+ (config-space write/read is standard substrate functionality; the TB tunnel teardown event is the per-test substrate state machine). Note: F41 should NOT be marked "real-hardware-only" like F30 — the substrate model is feasible.

## Source citations

- nvidia-driver-injector `docs/missions/mission-1-egpu-hot-plug-hot-power/experiments/h1-userspace-recovery-2026-05-28.md` §"H1 root cause confirmed + userspace recovery to BAR1=32GB without reboot"
- Forensic archive `/var/log/mission-1-archaeology/chip-state-diff-2026-05-28/` — passive register diff between cold-plug and userspace-recovered
- `drivers/pci/rebar.c` — kernel ReBAR helpers (`pci_rebar_get_max_size`, `pci_rebar_set_size`, `pci_rebar_init`)
- `include/uapi/linux/pci_regs.h` — `PCI_EXT_CAP_ID_REBAR`, `PCI_REBAR_CAP`, `PCI_REBAR_CTRL`, `PCI_REBAR_CTRL_BAR_SIZE` definitions
- nvidia-driver-injector `tools/fix-bar1.sh` — the userspace recovery sequence (verified n=2 on the apnex rig 2026-05-28)
- Upstream proposal name in the project: **E27** — kernel TB hot-add patch to call `pci_rebar_set_size` on hot-added devices

## Cross-references

- [[F30]] — Bridge-window stickiness. SAME OBSERVABLE SYMPTOM (BAR1 unassigned or undersized) but DIFFERENT MECHANISM. F30 framing is bridge-side cold-boot window allocation; F41 framing is chip-side ReBAR advertisement. On hardware where both manifest, F41's mitigation (`tools/fix-bar1.sh` or E27) is required; F30's mitigation (`pci=hpmemsize=64G`) alone does not help F41.
- [[F40]] — open-arm re-init wedge (`RmInitAdapter` GSP lockdown-release stall) on a chip-responsive but incompletely-initialized state. F40 is the downstream complication when F41's userspace recovery is used — the recovery sequence leaves the chip in a state F40 wedges on. `tools/fix-bar1.sh --bind` + persistence engagement is the current operator-mitigation pair (persistence keeps the destructive LAST-CLOSE from re-creating F40's precondition).
- [[F31]] — Boot-time BAR allocation failure. Different trigger (cold-boot resource pressure); F31's UEFI/cmdline mitigations don't apply to F41 because F41 fires on hot-add, not on cold-boot.
- [[F32]] — BAR1/BAR3 boundary DMA failure. Distantly related (BAR1 mis-sizing leads to potential boundary issues). F32's substrate model addresses a different failure-class boundary.
