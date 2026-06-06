# F31 — Boot-time BAR allocation failure — cold-boot resource pressure starves the GPU's BARs

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — kernel-side / firmware-side root cause; documented as known-mitigated-elsewhere)* |
| **Confidence** | hypothesis (documented kernel-mailing-list failure class; no specific injector field incident attributed, but contributes to the broader "GPU enumerated but won't initialise" symptom class) |
| **Predecessor evidence** | kernel mailing-list discussions of cold-boot BAR allocation failure on systems with high PCIe device count + 32-bit BAR pressure; distribution bug reports tagged "PCI: failed to assign" at cold-boot |
| **fake-5090 mechanism** | **Firmware/kernel-class only.** Substrate cannot reproduce; failure-index entry documents the third member of the BAR-allocation-failure trio (alongside F29 and F30) |

## Symptom (driver view)

At cold-boot, the kernel's PCIe enumeration walks the entire device tree and assigns BARs based on the firmware's resource-map (`E820`, `_CRS` ACPI methods, etc.) and the kernel's own allocation policy. When the system has a high PCIe device count (multi-NVMe, multi-NIC, integrated graphics, plus the discrete GPU) and the firmware's resource-map is conservatively sized, the cold-boot allocator can fail to find a suitable resource range for the GPU's BARs — even though the GPU is physically attached at cold-boot.

The failure is distinct from F29 (hot-plug-vs-cold-boot retry asymmetry — F31 is cold-boot-only) and F30 (bridge-window stickiness — F30 is hot-plug-specific). F31 is the "we ran out of 64-bit MMIO space" or "we couldn't satisfy the 32GB-aligned-to-32GB requirement of BAR1 within the firmware's reported memory map" cold-boot case.

The driver-visible symptom is the same as F29/F30: `nv_pci_probe` runs against a device with one or more BARs unassigned (`pci_resource_start(pdev, N) == 0`); `nv_pci_remap_bar()` returns failure; `rm_init_adapter` fails. `dmesg` shows kernel-side "PCI: failed to assign Mem-Pref" or "PCI: BAR N: no space for ..." messages from cold-boot enumeration.

The injector cannot fix this — the allocation runs before `nvidia.ko` ever loads. Operator mitigations are firmware-level (BIOS update with corrected resource map; UEFI settings to enable "Above 4G Decoding" / "Resizable BAR Support" / "Memory Mapped IO Above 4G") and kernel-level (`pci=use_crs` / `pci=nocrs` flips; `pci=bigroot_window` if supported).

## Trigger sequence

**Firmware/kernel-class only.** Reproducing F31 requires:

1. A real system whose firmware resource-map is too tight for the GPU's BAR requirements at cold-boot.
2. The GPU physically attached at cold-boot (this distinguishes F31 from F29/F30 which are hot-plug cases).
3. A real cold-boot.

The fake-5090 v1 substrate cannot reproduce F31:
- The substrate runs in a post-boot environment; it does not participate in cold-boot enumeration.
- The substrate cannot inject firmware-resource-map pressure.

F31 routes to real-hardware reproduction in cases where users report cold-boot BAR-allocation failures (rare on workstation/desktop boards; more common on industrial / embedded / minimalist-firmware boards).

## Expected post-patch behaviour

**No injector patch addresses F31.** Operator-level mitigations:

1. **UEFI settings:** Enable "Above 4G Decoding" / "Resizable BAR Support" / "Memory Mapped IO Above 4G" / "PCI Express Native Control" — typical settings that allow the firmware to expose a larger 64-bit MMIO range.
2. **Firmware update:** Vendor BIOS update addressing the resource-map issue (if vendor publishes one).
3. **Kernel cmdline:** `pci=use_crs` (force kernel to honour ACPI `_CRS` for resource ranges) or `pci=nocrs` (force kernel to compute its own range, ignoring `_CRS`) — opposite directions; either may help depending on the firmware bug.

The injector's relationship to F31 is observational (same as F29/F30): A4's close-path-telemetry surfaces the BAR-unassigned condition. The distinction between F29/F30/F31 is operator-side, made by inspecting the cold-boot dmesg for kernel-side allocation messages.

## Assertion shape

**fake-5090 v1 cannot assert F31 directly.** The failure-index entry exists for documentation parity and as the third member of the BAR-allocation-failure trio.

**Real-hardware-only assertions:**

- Cold-boot the system with the GPU attached.
- Capture cold-boot `dmesg` (before nvidia.ko loads) and grep for `PCI: failed to assign` or `PCI: BAR N: no space for`.
  - Pre-mitigation: failures observed for the GPU's BAR.
  - With UEFI "Above 4G Decoding" enabled (or relevant setting): failures should disappear.

**Distinguishing F31 from F29/F30 (operator triage workflow):**

- F31: failures appear in cold-boot dmesg, BEFORE nvidia.ko loads.
- F29: failures appear AFTER a hot-plug-attach event; cold-boot dmesg is clean.
- F30: failures appear AFTER a hot-plug-attach event; specifically the bridge-window grow attempt failed.

**Negative assertion:**

- F31 is NOT F29 (hot-plug-specific).
- F31 is NOT F30 (bridge-window stickiness — F31 can occur even with adequate bridge windows if the root-level resource-map is starved).
- F31 is NOT F32 (BAR1/3 boundary DMA — device-side; F31 is allocation-side).

## fake-5090 mechanism mapping

- **Firmware/kernel-class only.** The substrate cannot model firmware resource-map state.
- **Boundary-documentation role.** F31 completes the BAR-allocation-failure trio (F29/F30/F31). The three are mechanistically distinct but produce the same driver-visible symptom; the failure index keeps them separated so operator triage workflows can find each.
- **Future-work hook.** Out of scope for v2 as well — modelling firmware resource-map pressure requires running against a real kernel boot, which is outside the user-space substrate's domain.

## Source citations

- Linux kernel `drivers/pci/setup-res.c` — cold-boot allocation policy
- Linux kernel cmdline documentation — `pci=use_crs`, `pci=nocrs`, `pci=bigroot_window`
- UEFI specification — "Above 4G Decoding", "Resizable BAR Support"
- Distribution bug trackers (Fedora, Ubuntu, Debian) — class of "PCI: failed to assign" cold-boot reports
- F29 entry (allocator-retry asymmetry — hot-plug sibling)
- F30 entry (bridge-window stickiness — hot-plug sibling)
