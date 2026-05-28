# F30 — Bridge-window stickiness — TB/USB4 downstream-port windows under-sized for hot-plug BARs

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — bridge-side kernel mitigation; documented as known-mitigated-elsewhere)* |
| **Confidence** | community-evidence |
| **Predecessor evidence** | aorus-5090-egpu PM #6 + PM #7 (Barlow Ridge bridge-window investigation); kernel `pci=hpiosize=` and `pci=hpmemsize=` cmdline documentation |
| **fake-5090 mechanism** | **Bridge-class only.** Substrate cannot model bridge-window register state; failure-index entry documents the boundary and operator mitigations |

## Symptom (driver view)

TB/USB4 controllers expose downstream PCIe ports whose memory windows (`PCI_MEMORY_BASE` / `PCI_MEMORY_LIMIT` registers on the bridge) are sized at cold-boot time based on whatever devices are present or expected. The kernel's enumeration logic chooses window sizes from a combination of:

1. Devices actually present at cold-boot (allocates exact-fit windows).
2. The `pci=hpiosize=` / `pci=hpmemsize=` cmdline parameters (allocates hot-plug-reserve windows for downstream ports likely to receive hot-plug devices).
3. Distribution defaults (typically conservative: 1 MB hpiosize, 2 MB hpmemsize per hot-plug-capable downstream port).

When a hot-plug device attaches whose BAR requirements exceed the pre-sized window, the kernel CAN attempt to resize the window (re-walking the bridge hierarchy and renegotiating windows). However, this resize is gated by `pci=realloc` (F29's territory) AND requires that the parent bridge's window has enough headroom to grow into. On TB/USB4 controllers integrated into laptop chipsets, the parent bridge windows are often tight; the resize fails; the hot-plug device's BARs are not assigned; the driver sees the same symptom as F29.

The bridge-window-stickiness aspect specifically: even with `pci=realloc=on`, if cold-boot allocated a 256 MB window to the TB downstream port (because the cold-boot-time hot-plug-reserve heuristic said "TB downstream ports usually attach modest devices"), and a 5090 attaches needing 32 GB at BAR1, the kernel attempts to grow the window but the parent (the integrated TB host controller) has insufficient headroom. The resize fails; F30 fires.

The injector cannot fix this — the bridge-window state is on the bridge silicon, accessed by the kernel's PCIe layer before `nvidia.ko` ever sees the device. Operator mitigations: pre-size at cold-boot time via `pci=hpmemsize=64G` (very aggressive; may conflict with other devices) or via per-board bridge-window programming via `setpci` before TB enumeration (board-specific; requires knowing the bridge BDF in advance).

## Trigger sequence

**Bridge-class only.** Reproducing F30 requires:

1. A real TB/USB4 controller with the under-sized cold-boot-window behaviour.
2. A real hot-pluggable device whose BAR requirements exceed the pre-sized window (5090's 32 GB BAR1 against most laptop-integrated TB controllers).
3. A cold-boot WITHOUT the device attached, followed by hot-plug attach.

The fake-5090 v1 substrate cannot reproduce F30 because the bridge-window state is on the bridge silicon, not on the device-side substrate. Demonstrating the under-sized-window failure requires the kernel to actually walk the bridge hierarchy against a real bridge whose `PCI_MEMORY_LIMIT` is too tight.

F30 routes to real-hardware reproduction (apnex rig with 5090 hot-plug across various host TB controllers; Barlow Ridge bridge is the documented apex case).

## Expected post-patch behaviour

**No injector patch addresses F30.** Operator-level mitigations:

1. **`pci=hpmemsize=64G`** (cmdline; pre-sizes every hot-plug-capable downstream port to 64 GB — covers 5090's 32 GB BAR1 with headroom; may not be feasible on systems where multiple hot-plug ports exist and total bridge headroom is limited).
2. **`pci=hpmemsize=32G,pci=hpiosize=4K`** (more targeted; matches 5090's specific requirements).
3. **Per-board bridge-window programming** via `setpci` in an early-boot service before TB controller initialises (e.g. `aorus-egpu-bridge-link-cap`-style service per F14's predecessor pattern).
4. **BIOS/firmware updates** that ship larger default hot-plug-reserve windows for TB downstream ports (out of operator control unless vendor publishes an update).

The injector's relationship to F30 is observational: A4's close-path-telemetry surfaces the BAR-unassigned condition (same surface as F29) and allows operator tooling to distinguish F29 vs. F30 vs. F31 by checking bridge-window register state at observation time.

## Assertion shape

**fake-5090 v1 cannot assert F30 directly.** The failure-index entry exists for documentation parity with F29 and to point real-hardware operators at the bridge-window mitigations.

**Real-hardware-only assertions (apnex rig):**

- Cold-boot baseline: enumerate the TB controller; record bridge-window sizes (`lspci -vvv -s <TB_bridge_BDF>` shows memory-window range).
- Hot-plug attempt: hot-plug the 5090; check `lspci -vvv -s <GPU_BDF>` for BAR1 assignment.
  - Pre-mitigation: BAR1 likely unassigned (F30 fires).
  - With `pci=hpmemsize=64G` cmdline: BAR1 SHOULD be assigned.
  - With `setpci`-driven pre-sizing: BAR1 SHOULD be assigned.
- Distinguish F30 from F29: check bridge-window size pre-vs-post hot-plug. If the window grew (via realloc), F29 was the issue. If the window did not grow (because parent has no headroom), F30 was the issue.

**Negative assertion (what F30 is NOT):**

- F30 is NOT F29 (allocator retry budget — F29 is about retry count; F30 is about cold-boot window pre-sizing).
- F30 is NOT F31 (cold-boot allocation failure — F30 is hot-plug-specific).
- F30 is NOT F32 (BAR1/3 boundary DMA — device-side).

## fake-5090 mechanism mapping

- **Bridge-class only.** The substrate exposes its BAR-requirement profile; the bridge-window state is on the bridge silicon. The substrate cannot drive bridge-window register writes.
- **Boundary-documentation role.** F30 sits alongside F29 to make the distinction explicit. Operators triaging "GPU enumerated but BAR1 unassigned" need to know whether their case is allocator-retry (F29 — `pci=realloc=on` helps) or window-pre-sizing (F30 — `pci=hpmemsize=` helps).
- **Future-work hook.** A fake-5090 v2 with bridge-window emulation (modelling a `PCI_MEMORY_LIMIT` register and rejecting BAR assignments that exceed it) could exercise F30; out of scope for v1.

## Source citations

- aorus-5090-egpu PM #6 (Barlow Ridge bridge investigation)
- aorus-5090-egpu PM #7 (Lever H9a service tightening — bridge-side mitigation pattern)
- Linux kernel cmdline documentation — `pci=hpiosize=` and `pci=hpmemsize=`
- Linux kernel `drivers/pci/setup-bus.c` — bridge-window resize logic (`pbus_size_mem`)
- F14 entry (related Barlow Ridge investigation — different cause but same bridge silicon)
- F29 entry (allocator-retry asymmetry — sibling kernel-class issue with same driver-visible symptom)
- F31 entry (cold-boot BAR failure — alternate cause for same symptom)
- **F41 entry** — chip-side ReBAR Control register reset on TB hot-add. SAME observable symptom (BAR1 unassigned / undersized) but DIFFERENT MECHANISM. F30's bridge-window-stickiness mitigations (`pci=hpmemsize=64G`) do NOT help F41 because the chip itself is advertising a smaller BAR size; the kernel sizes the bridge for what the chip asks for. On the apnex rig (NUC 15 Pro+ + AORUS RTX5090 AI BOX over TB4) the empirically dominant mechanism is F41, not F30 — confirmed via passive ReBAR Control register read 2026-05-28. Operators hitting broken-BAR1 should diagnose chip CTRL register first (per F41) before assuming bridge-side stickiness (F30).
