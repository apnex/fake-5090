# F33 — Thunderbolt signal-loss vs CC-pin disconnect — link-down has two physical causes

| Field | Value |
|---|---|
| **Sources** | A2 (bus-loss-watchdog) — same detection path covers both; E1 (egpu-detection) tags the transport |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #2 (Mode B silent freeze investigation); aorus-5090-egpu PM #11 (observer-induced failure investigation); TB4/USB4 specification §physical-layer disconnect classes |
| **fake-5090 mechanism** | Substrate's transport-classification state: substrate exposes an "extra-info" channel for "how did the disconnect happen?" — signal-loss (link went down due to noise/marginal cable) vs CC-pin (user pulled the connector cleanly); test harness configures which mode the surprise-removal A primitive uses |

## Symptom (driver view)

A Thunderbolt 4 / USB4 link can go down for two physically-distinct reasons:

1. **Signal-loss (Mode B-class).** The cable is still mechanically connected but the physical-layer link has failed (cable degradation, EMI, marginal connection, retimer fault, host-port issue). The driver sees: MMIO reads start returning `0xFFFFFFFF`; the TB controller's link-status register reports "link down"; the CC-pin (connector-detect pin) reports STILL CONNECTED. This is the canonical "the cable looks fine but the link is dead" case.
2. **CC-pin disconnect (clean removal).** The user physically pulled the connector. The driver sees: MMIO reads start returning `0xFFFFFFFF`; the TB controller's link-status register reports "link down"; the CC-pin reports DISCONNECTED. This is the canonical "user yanked the cable" case.

The driver's bus-loss detection (via the funnel-1 sentinel read or A2's Q-watchdog) catches both cases identically — both produce sink-state being set and the cascade-class teardown running. The DIFFERENCE matters for:

- **Recovery semantics.** Signal-loss might be recoverable on link renegotiation (TB controller re-trains); CC-pin disconnect is not recoverable without user re-plugging.
- **Operator-visible messaging.** A4's telemetry can surface "GPU lost due to link signal-loss (try re-seating cable)" vs. "GPU lost due to clean disconnect (re-plug to recover)" — distinct operator actions.
- **A3's recovery retry policy.** A3's H1 cap currently treats both identically (3 attempts then persist-kill); a future refinement could retry more aggressively on signal-loss (it's a transient physical-layer issue) and persist-kill immediately on CC-pin (user intent is clear).

The current injector handles both as the same failure mode (per the v1 design). F33 documents the distinction so the test catalogue exercises both scenarios — primarily to ensure A4's telemetry exposes the correct transport-class tag for each.

## Trigger sequence

**Signal-loss scenario:**

1. Daemon brings device to a healthy bound state via TB/USB4 transport (substrate's `is_external_gpu` set, topology includes a TB downstream port).
2. Test fires surprise-removal A with the "signal-loss" sub-mode: substrate transitions to MMIO-returns-sentinel + link-status register reports "link down" + CC-pin register reports CONNECTED.
3. Driver detects bus loss; cascade fires; sink-state set; teardown runs.
4. A4's telemetry surfaces: transport-class = TB; disconnect-reason = SIGNAL_LOSS.

**CC-pin scenario:**

1. Same setup.
2. Test fires surprise-removal A with the "CC-pin" sub-mode: substrate transitions to MMIO-returns-sentinel + link-status register reports "link down" + CC-pin register reports DISCONNECTED.
3. Driver detects bus loss; cascade fires; sink-state set; teardown runs.
4. A4's telemetry surfaces: transport-class = TB; disconnect-reason = CC_PIN.

## Expected post-patch behaviour

The injector's cascade-class teardown runs identically for both sub-modes (current v1 behaviour — the distinction does not affect the cascade itself). What the patches DO provide:

1. **A2's watchdog detects both** (already covered by F22's mechanism).
2. **E1's transport-classification** tags the device as TB/USB4 (already covered by F07).
3. **A4's telemetry surfaces the disconnect-reason** by reading the CC-pin register state captured at sink-set time. This requires:
   - Substrate exposes a "CC-pin state at last sink event" debug register.
   - A4's sink-set capture hook reads the CC-pin state and stores it in the per-device sysfs surface.
   - Operator tooling reads the surface to render the disconnect-reason to the user.

This is the F33-specific addition — an A4 enhancement, ~5-10 lines.

## Assertion shape

**Both sub-modes — common assertions (same as F22):**

- A2's Q-watchdog OR funnel-1 MMIO detection MUST fire within bounded time.
- Sink-state MUST be set; cascade-class teardown MUST run to completion within ≤ 1 s.

**Signal-loss-specific:**

- After teardown, A4's telemetry surface MUST report disconnect-reason = SIGNAL_LOSS.
- `cat /sys/bus/pci/devices/<BDF>/<A4-disconnect-reason-path>` MUST contain the string "signal_loss" (or the project's canonical-name).

**CC-pin-specific:**

- After teardown, A4's telemetry surface MUST report disconnect-reason = CC_PIN.
- `cat /sys/bus/pci/devices/<BDF>/<A4-disconnect-reason-path>` MUST contain "cc_pin" (or the project's canonical-name).

**Transport-class verification (E1 — already F07's territory but worth re-checking on TB-specific scenarios):**

- `cat /sys/bus/pci/devices/<BDF>/<E1-transport-path>` MUST contain "thunderbolt" or "usb4" (depending on substrate configuration).
- This MUST be set BEFORE the disconnect-reason is read.

**Negative — F33 fix MUST NOT misclassify non-TB transports:**

- Run the same sub-mode tests against a substrate configured as PCIe-discrete (`is_external_gpu == 0`, no TB downstream port). The disconnect-reason surface SHOULD report "n/a" or "not_applicable" — the CC-pin concept doesn't apply to discrete cards.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25-F28.
- **TB-transport substrate state.** Substrate must be configured with a topology that includes a TB downstream port; the host-side `pci_is_thunderbolt_attached` returns true; `nv->is_external_gpu` is set.
- **Sub-mode configuration.** Surprise-removal A has two sub-modes: signal-loss (CC-pin stays CONNECTED) and CC-pin (CC-pin transitions to DISCONNECTED). The test harness selects the sub-mode via a substrate-config flag.
- **CC-pin register exposure.** Substrate exposes a debug-register representing the CC-pin state. Real TB controllers expose this state via TBT firmware; the substrate emulates the exposed surface. The A4 patch reads this register at sink-set time.

## Source citations

- aorus-5090-egpu PM #2 (Mode B silent freeze — signal-loss scenario evidence)
- aorus-5090-egpu PM #11 (observer-induced failure — overlapping investigation that surfaced the signal-loss-vs-CC-pin distinction)
- TB4/USB4 specification §physical-layer disconnect classes
- A2 intent doc — bus-loss detection (transport-agnostic)
- A4 intent doc — telemetry-surface for disconnect-reason classification
- E1 intent doc — transport classification (already covers F07)
- F22 entry (silent hard hang — F33's signal-loss sub-mode is a subset)
- F07 entry (TB4/USB4 misclassified-as-internal — adjacent transport-classification issue)
