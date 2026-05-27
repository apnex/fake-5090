# F7 — TB4/USB4 GPU misclassified as internal

| Field | Value |
|---|---|
| **Sources** | E1-egpu-detection |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu `docs/iommu-gsp-lockdown-analysis.md` (untrusted-port signal), implied across H10/H16/H17 ledger entries |
| **fake-5090 mechanism** | Topology modelling (`pci_is_thunderbolt_attached`, untrusted/external-facing-port signal) |

## Symptom (driver view)

NVIDIA's upstream "is this GPU external?" classifier uses `pci_pcie_type()` plus a bridge walk to guess at externality. The heuristic happens to work for TB3-attached GPUs (TB3 enumerates with patterns the upstream code recognises) but misclassifies TB4/USB4-tunneled GPUs as *internal*. Once misclassified, downstream policy paths that should engage only on external GPUs (the A2 Mode B watchdog being the load-bearing example) don't engage, and the driver runs with the wrong assumptions about hot-removability, link-stability, and reset-permissions.

Specific downstream consequences observed:
- A2 watchdog kthread not spawned on TB4 GPUs (under the policy "watchdog only on external GPUs").
- IOMMU + GSP_LOCKDOWN cascade interactions (F14) — the misclassification affects whether the driver applies the bridge-link-cap policy.

## Trigger sequence

1. Daemon presents the device with a topology that mimics TB4/USB4 attachment:
   - Upstream chain includes a bridge with the kernel's Thunderbolt-attached signal asserted (whatever vfio-user wires to `pci_is_thunderbolt_attached()` — typically a PCIe extended cap or sysfs marker the kernel computes from the bridge tree).
   - OR: the bridge advertises the "untrusted" / "external-facing port" attribute that the kernel exposes for non-TB external ports (OCP form factors).
2. Driver probes the device.
3. E1 classification logic runs and decides "external."
4. Re-run with the topology mimicking PCH-rooted internal attachment (no TB signal, no external-facing-port signal). Driver MUST decide "internal."
5. Re-run with the topology mimicking TB3 (legacy, the working-by-accident case). Driver MUST decide "external" — regression guard, the patch must not break TB3.

## Expected post-patch behaviour

E1 replaces the upstream heuristic with: `external := pci_is_thunderbolt_attached(pdev) || pdev_or_bridge_has_external_facing_port_signal(pdev)`. The OR is load-bearing — TB covers TB3/TB4/USB4, the external-facing-port signal covers non-TB external (OCP). One log line per probe naming which marker(s) fired so post-mortems can trace the decision.

False-positive guard: purely internal PCH-rooted GPU MUST NOT be classified external (lest the watchdog spawn on every iGPU and waste cycles).

## Assertion shape

**Per-topology classification:**

For each topology variant (TB4, TB3, OCP-external, PCH-internal), after probe:

- `dmesg | grep -E 'tb_egpu egpu-detect.*pdev=<BDF>.*classification=(external|internal).*markers=\[.*\]'` — matches exactly one line.
- The `classification=` field matches the expected value for the variant.
- The `markers=` list contains:
  - TB4/TB3: `tb_attached`
  - OCP-external: `external_facing_port`
  - PCH-internal: empty `[]`.
- Sysfs proof of downstream policy engagement: presence/absence of `/sys/bus/pci/devices/<BDF>/tb_egpu_qwd_*` correlates with classification (external → present; internal → absent), gated by the A2/E1 integration policy.

**Regression guard (TB3):**

- The TB3 variant test exists and PASSES. Without it, an E1 implementation that accidentally hardcodes "TB4 only" would silently regress the already-working TB3 case.

**False-positive guard (PCH-internal):**

- The PCH-internal variant correctly classifies as `internal`; no watchdog spawned. Without this assertion, an E1 implementation that ORs in any "is this a bridge?" check would explode to every onboard GPU.

## fake-5090 mechanism mapping

- **Phase 2 topology modelling.** Requires the daemon to construct a multi-device vfio-user topology (host bridge → TB controller → device, or host bridge → root port → device) and present the appropriate capability bits / sysfs hints so the kernel's `pci_is_thunderbolt_attached()` returns true.
- Of the 14 failure modes, F7 is the topology-purity test — it doesn't need MMIO, AER, or BAR work, only correct enumeration and bridge attribute presentation.
- Can be the *first* phase-2 fixture: minimal cognitive load on the rest of the device model.

## Source citations

- `E1-egpu-detection.md` §"Upstream classifier shortcomings" (preamble), §"`pci_is_thunderbolt_attached` is the correct signal" (~50–100), §"OR with external-facing-port for non-TB external" (~100–130), §"One log line per probe" (~150–180), §"Regression guards: TB3 preserved, PCH-internal not false-positive" (~180–220).
- aorus-5090-egpu `docs/iommu-gsp-lockdown-analysis.md` — the analysis chain that surfaced the misclassification's downstream impact on GSP_LOCKDOWN cascades.
- Reliability hypothesis ledger H10/H16/H17 in `aorus-5090-egpu/docs/reliability-hypothesis-ledger.md` for the field-incident chain.
