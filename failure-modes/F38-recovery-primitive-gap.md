# F38 — Recovery-primitive gap — `pci_reset_function` is the only mechanism, but it's not always safe

| Field | Value |
|---|---|
| **Sources** | A3 (recovery — uses `pci_reset_function`); documented gap surfaced during A3 design review |
| **Confidence** | hypothesis |
| **Predecessor evidence** | A3-recovery intent doc §"Reset-primitive selection"; Linux kernel `drivers/pci/pci.c` notes on `pci_reset_function` vs `pci_reset_bus` vs `pci_reset_slot`; community kernel-mailing-list discussions of TB/USB4 reset semantics |
| **fake-5090 mechanism** | Substrate reset-primitive support: substrate accepts `pci_reset_function`-equivalent invocations and reports outcome per configuration (already partly covered by F27); F38 documents the additional consideration that the chosen reset primitive is not always the right one for the underlying failure |

## Symptom (driver view)

The Linux kernel exposes multiple reset primitives for PCI devices:

1. **`pci_reset_function(pdev)`** — function-level reset (FLR). Resets only the specific function (BDF). Most fine-grained; safe to call on a single device without disturbing siblings.
2. **`pci_reset_slot(pdev)`** — slot-level reset. Resets all functions in a slot. Useful when sibling functions share state but more disruptive.
3. **`pci_reset_bus(pdev)`** — bus-level reset. Resets all devices on the bus segment. Most disruptive; necessary when the bus itself is wedged.
4. **`pci_bus_reset(bus)`** — internal bus-reset helper used by the above.
5. **TB/USB4-specific resets** — controller-level link renegotiation, available via TB-subsystem APIs but not commonly exposed to PCI drivers.

A3's current design uses `pci_reset_function` as the recovery primitive. This is correct for most failure modes (F23/F27 cases — single-device reset is sufficient when the device's own state machine is wedged). It is NOT sufficient for:

- **Bus-wedge failures.** If the failure is bus-level (PCIe link itself is in an error state, or the TB downstream port is wedged), function-level reset cannot recover — the link must be re-trained, which requires bus-level or controller-level reset.
- **Multi-function devices.** Some GPUs expose multiple functions (graphics function + audio function on the same card); function-level reset of one without the other can leave them in inconsistent state.
- **Shared-bridge state.** If the PCIe bridge upstream of the device has a stuck state (e.g. AER mask register stuck, link-cap register stuck), function-level reset of the device downstream does not clear the bridge state.

F38 is the documented gap: A3's recovery surface is correct for the dominant cases but explicitly bounded. Cases that require bus-level or controller-level reset are NOT covered by A3 v1; they remain unrecoverable in software and require either operator intervention (host reboot, manual `setpci`-driven bridge reset) or upstream-kernel improvements.

## Trigger sequence

**This is a documentation-gap entry — no specific reproduction trigger.** The gap is exposed by:

1. A failure that A3's `pci_reset_function` cannot recover (e.g. F23 with the substrate configured to "post-reset-still-dead" mode where the device IS dead at the bus level — F27's R3 state).
2. A3's H1 cap fires after 3 failed function-level reset attempts; A3 marks persistent-killed.
3. An operator-driven attempt to recover via bus-level reset (e.g. `setpci -s <bridge_BDF> BRIDGE_CONTROL=0x40:0x40` toggle to trigger SBR) succeeds where `pci_reset_function` failed — demonstrating that A3 could have recovered the device had it used a different primitive.

## Expected post-patch behaviour

**A3 v1's design is correct as bounded.** F38 documents:

1. The bounded-scope decision (A3 uses `pci_reset_function`; cases requiring bus/controller-level reset are out of A3 v1's scope).
2. The future-work pathway: A3 v2 (or a separate A6 patch) could add bus/controller-level reset as a fallback after function-level reset fails the H1 cap. The decision was deferred because:
   - Bus-level reset is significantly more disruptive (resets sibling devices on the same bus).
   - Controller-level reset (TB-subsystem APIs) requires deeper integration with the TB-subsystem and is platform-specific.
   - The recovery-success rate for cases A3 v1 cannot handle is low (genuinely-broken-chip cases like F23 are not recoverable by ANY reset primitive — they need RMA).

3. The operator-mitigation path: operators who hit a case where A3's function-level reset fails can manually attempt bus-level reset via `setpci` SBR toggles. The injector's documentation includes the relevant `setpci` recipes for the supported boards.

## Assertion shape

**Documentation-gap assertion (CI):**

- A3-recovery intent doc MUST contain a §"Recovery-primitive scope" section that:
  - States the chosen primitive (`pci_reset_function`).
  - Lists the bounded cases (function-level wedge — recoverable).
  - Lists the out-of-scope cases (bus-level wedge, controller-level wedge — not recoverable by A3 v1).
  - Documents the operator-mitigation `setpci` recipes for the bus-level fallback.

**Future-work tracking assertion:**

- The project's future-work tracker (TBD location — possibly `consumer-holders-and-teardown-future-work.md` or a new `recovery-extensions-future-work.md`) MUST contain an entry for "A6 / A3 v2 — bus-level and controller-level reset fallback" with the deferred-decision rationale.

**Operator-recipe verification (apnex setup guide):**

- The apnex/nvidia-driver-injector setup guide MUST contain a §"Manual recovery for A3-bounded failures" with:
  - The `setpci` SBR-toggle recipe.
  - The board-specific bridge-BDF identification procedure.
  - The pre-conditions for safe SBR (no sibling devices in active use on the same bus).

**Negative — F38 is NOT a runtime assertion:**

- F38 has no fake-5090-side runtime assertion. The substrate cannot meaningfully exercise "the recovery primitive A3 chose is wrong for this failure" — it can only exercise the substrate behaviour, which A3 will either recover from (F27) or not (F23 with broken substrate).

## fake-5090 mechanism mapping

- **No new substrate primitive.** F38's substrate-side behaviour is already covered by F27's reset-behaviour configuration (substrate accepts resets and reports outcome).
- **Documentation-class entry.** F38 exists in the failure index to:
  - Make explicit that A3's recovery scope is bounded.
  - Provide a discoverable pointer to operator-mitigation recipes for cases A3 cannot handle.
  - Track the future-work pathway for A6 / A3 v2 extension.
- **Future-work hook.** A fake-5090 v2 or v3 could exercise bus-level / controller-level reset primitives, requiring substrate-side support for bus-level reset acceptance (currently only function-level is modelled).

## Source citations

- A3-recovery intent doc §"Reset-primitive selection" — bounded-scope decision
- Linux kernel `drivers/pci/pci.c` — `pci_reset_function`, `pci_reset_slot`, `pci_reset_bus` documentation
- Linux kernel `drivers/thunderbolt/` — TB-subsystem reset APIs (out-of-scope reference)
- `setpci(8)` documentation — SBR-toggle recipes
- apnex/nvidia-driver-injector setup guide — §"Manual recovery" (TBD; not yet written)
- F27 entry (`Reset Required` loop — F38's recoverable sibling; F38 is the out-of-scope class)
- F23 entry (instant Xid 79 cold-boot — F38 explicitly acknowledges this class is unrecoverable)
- `consumer-holders-and-teardown-future-work.md` — future-work tracking location
