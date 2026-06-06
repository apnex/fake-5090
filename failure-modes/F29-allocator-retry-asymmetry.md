# F29 — PCIe allocator retry asymmetry — kernel allocator gives up faster on hot-plug than cold-boot

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — kernel-side root cause; documented as known-mitigated-elsewhere)* |
| **Confidence** | community-evidence |
| **Predecessor evidence** | NVIDIA issue #888 (RTX 5090 hot-plug: BAR allocation fails on first hot-plug; same allocation pattern succeeds at cold-boot); kernel mailing-list discussions of `pci_assign_unassigned_bridge_resources` retry behaviour |
| **fake-5090 mechanism** | **Kernel-class only.** The substrate cannot drive the kernel's PCIe resource allocator to demonstrate F29; failure-index entry exists to document the boundary between fake-5090's scope and kernel-side root causes |

## Symptom (driver view)

The Linux kernel's PCIe resource allocator (`pci_assign_resources`, `pci_assign_unassigned_bridge_resources`, etc.) has different retry behaviour for cold-boot enumeration vs. hot-plug-attach enumeration. At cold-boot, the allocator runs multiple passes with progressively-relaxed constraints (e.g. assign-must-have → assign-prefer → assign-best-effort), and if all fail, the kernel logs the failure and proceeds with whatever resources were successfully assigned.

At hot-plug (TB/USB4 device arrival, OCuLink connect, SR-IOV VF instantiation), the allocator runs a smaller number of passes — typically only the strict-constraint pass. When the strict pass fails for the new device's BAR requirements (most commonly BAR1's 32GB requirement on the 5090 colliding with bridge-window aperture limits set at cold-boot time), the kernel logs "BAR N: failed to assign" and the device enumerates with no valid BAR1.

The driver-visible symptom: `nv_pci_probe` runs against a device with `pci_resource_start(pdev, 1) == 0` (BAR1 unassigned). Driver attempts to ioremap → fails → `rm_init_adapter` fails → user sees "GPU detected but won't initialise."

This is structurally distinct from F30 (bridge-window stickiness — bridge windows set at cold-boot time may be too small for hot-plug-attached BAR requirements) and F31 (genuine BAR-allocation failure at cold-boot — different cause class). F29 is specifically the "kernel allocator's retry budget differs between cold-boot and hot-plug" asymmetry.

The injector cannot fix this directly — the allocator is kernel-side code. Operator-level mitigations include `pci=realloc=on` kernel cmdline (forces hot-plug-attach to run the full cold-boot-style retry sequence) and manual `setpci`-driven bridge-window pre-sizing before hot-plug attach.

## Trigger sequence

**Kernel-class only.** Reproducing F29 requires:

1. A real hot-pluggable device with BAR requirements that fit at cold-boot but stress the hot-plug allocator (the 5090's 32 GB BAR1 is the canonical case).
2. A real Linux kernel with the asymmetric retry behaviour (most kernel versions since ~5.x).
3. A real cold-boot followed by a real hot-plug-attach event (cable connection, OCuLink lever, TB4 enumeration trigger).

The fake-5090 v1 substrate cannot reproduce F29 because:
- The kernel's PCIe allocator runs against root-port silicon and bridge-window registers; the substrate only exposes device-side state, not bridge-side state.
- Demonstrating the asymmetry requires comparing cold-boot enumeration outcomes vs. hot-plug enumeration outcomes; the substrate runs in a stable post-boot environment and does not participate in cold-boot enumeration at all.

F29 routes to real-hardware reproduction (apnex rig with 5090 hot-plug; OCuLink lever cycles).

## Expected post-patch behaviour

**No injector patch addresses F29.** Three operator-level mitigations are documented for completeness:

1. **`pci=realloc=on` kernel cmdline.** Forces hot-plug-attach to run the full cold-boot-style retry sequence. Resolves F29 at the cost of slightly longer hot-plug latency.
2. **Bridge-window pre-sizing via `setpci`.** Set the bridge windows at cold-boot time to be large enough for any expected hot-plug attach. Requires knowing the BAR requirements in advance.
3. **Kernel-cmdline parameters specific to TB/USB4 hot-plug retry.** Distribution-specific; documented in the injector's operator-guide if applicable.

The injector's relationship to F29 is observational: A4's close-path-telemetry surfaces the BAR-unassigned condition via sysfs so operator tooling can detect F29 and surface a clear error message ("GPU enumerated but BAR1 unassigned — likely F29; try `pci=realloc=on`") rather than the generic `rm_init_adapter` failure.

## Assertion shape

**fake-5090 v1 cannot assert F29 directly.** The failure-index entry exists to:

1. Document the coverage boundary.
2. Provide a cross-reference for operators triaging real-hardware "GPU enumerated but won't initialise" reports.
3. Surface the operator-mitigation cmdline parameters in a discoverable form.

**Real-hardware-only assertions (if a hot-plug-attach soak suite is ever spun up):**

- Cold-boot baseline: enumerate the device at cold-boot; verify all BARs assigned (`lspci -vvv -s <BDF>` shows resource ranges for BAR0/BAR1/BAR3).
- Hot-plug case: cold-boot WITHOUT the device attached; hot-plug attach the device; verify all BARs assigned (likely fails — this IS F29).
- Mitigation case: cold-boot with `pci=realloc=on` WITHOUT the device attached; hot-plug attach; verify all BARs assigned (F29 mitigated).

**Observability assertion (what fake-5090 CAN check — substrate-side):**

- The substrate exposes its BAR-requirement profile (e.g. "I claim 32 GB at BAR1, 256 MB at BAR0, 32 MB at BAR3"). The host-side `lspci -vvv -s <BDF>` MUST report the substrate's claimed sizes. If the host shows smaller-than-claimed sizes, that indicates kernel-side allocation issues — but the substrate cannot distinguish F29 from F30 from F31 from this alone.

**Negative assertion (what F29 is NOT):**

- F29 is NOT F30 (bridge-window stickiness — a related but distinct kernel-side issue).
- F29 is NOT F31 (boot-time BAR failure — different trigger: cold-boot allocation fails, no hot-plug involved).
- F29 is NOT F32 (BAR1/3 boundary DMA failure — device-side, not allocation-side).

## fake-5090 mechanism mapping

- **Kernel-class only.** The substrate exposes BAR-requirement registers (BAR-size capability, BAR-prefetchable bit, etc.) and the kernel-side allocator drives against the bridge silicon. Reproducing F29 requires the kernel to actually run its allocator with the asymmetric retry behaviour, which requires a real cold-boot-then-hot-plug sequence the substrate cannot drive.
- **Boundary-documentation role.** F29 is in the index to make explicit that BAR-allocation failures at hot-plug time are a kernel-class issue, not a driver-class issue — operators triaging "GPU enumerated but won't initialise" should not look in the injector for a fix.
- **Future-work hook.** A fake-5090 v2 that includes a bridge-window-pressure scenario could exercise the kernel allocator's retry path in CI; explicitly out of scope for v1.

## Source citations

- NVIDIA issue #888 — RTX 5090 hot-plug: BAR allocation fails on first hot-plug; same allocation pattern succeeds at cold-boot
- Linux kernel `drivers/pci/setup-bus.c` — `pci_assign_resources`, `pci_assign_unassigned_bridge_resources` (retry-budget asymmetry)
- Kernel cmdline documentation — `pci=realloc=on` parameter
- `_community-signal.md` §"#888 — 5090 BAR1 hot-plug allocation asymmetry"
- F30 entry (bridge-window stickiness — sibling kernel-class issue)
- F31 entry (cold-boot BAR failure — alternate cause for same symptom)
- F32 entry (BAR1/3 boundary DMA — device-side issue with similar BAR-related surface)
