# F8 — AER Internal-Error mask demotes Internal errors to silent-drop

| Field | Value |
|---|---|
| **Sources** | C2-aer-internal-unmask |
| **Confidence** | hypothesis (firmware-dependent; some platform firmware sets this mask and some doesn't — the patch is defensive and idempotent) |
| **Predecessor evidence** | aorus-5090-egpu G3-H UncMaskClear (within Lever-stack) |
| **fake-5090 mechanism** | Config-space — writable AER Uncorrectable Mask register |

## Symptom (driver view)

Some platform firmware initialises the device's AER Uncorrectable Mask register with the Internal Error bit *set*. Per PCIe r6.0 §6.2.3.2.4, masked uncorrectable errors do NOT escalate to the AER framework — the severity register is *irrelevant* once an error class is masked. Consequence: AER Internal Errors silently drop before the kernel sees them. The driver's downstream recovery pipeline (C3 retry → C4 dispatch → A3 reinit) has nothing to trigger on, because no `.error_detected` callback ever fires. This is a *cause-of-no-cause*: the bug is invisible from the driver's perspective because the symptom is suppression.

Edge cases the patch must handle:
- SR-IOV Virtual Functions: AER cap is PF-only on this device class — VFs must be skipped (no-op, no error).
- Device without AER extended cap (config addr `0x100` absent): unmask is a no-op, no error.
- Firmware that *didn't* set the bit: unmask is idempotent.

## Trigger sequence

1. Daemon presents device with:
   - AER extended capability present at config offset `0x100` (or later in the extended-cap chain).
   - Uncorrectable Mask register has the Internal Error bit (bit 22 per PCIe spec) **set** at driver-bind time.
2. Driver probes; C2 runs at probe and clears the bit.
3. Test injects an AER Internal Error (write to Uncorrectable Status register + assert the message).
4. Observe whether the kernel's AER framework receives and decodes the error.

Negative-case sweeps:
- **5a.** Device has no AER cap → C2 should no-op without logging an error.
- **5b.** Device is an SR-IOV VF → C2 should skip without logging an error.
- **5c.** Device has AER cap but Internal Error bit was already clear at bind → C2's clear-and-reissue is idempotent.

## Expected post-patch behaviour

C2 reads the AER Uncorrectable Mask at probe time. If the Internal Error bit is set, clear it and re-issue the write (verify the write took). If the cap is absent, return clean. If this is a VF, return clean. Emit one log line per device naming whether the unmask was needed.

After C2, AER Internal Errors injected on the device DO escalate to the kernel's AER framework; the kernel decodes and dispatches per `pci_error_handlers` (which C4 ensures exists), and the recovery pipeline (A3) can engage.

## Assertion shape

**Probe-time behaviour:**

- After driver probe, `setpci -s <BDF> ECAP_AER+0x08.L` (Uncorrectable Mask register, offset 0x08 within the AER cap) returns a value with bit 22 **clear**.
- `dmesg | grep -E 'tb_egpu aer-unmask: pdev=<BDF>.*internal_error_bit=(was-set|already-clear)'` — exactly one line per probe (format per C2 intent doc).

**Functional escalation (post-patch):**

- After step 3 (Internal Error injection), kernel AER framework receives the event: `dmesg | grep -E 'AER:.*Internal Error'` matches at least one line.
- The driver's `.error_detected` callback fires (proves the C2 → C4 → driver chain is intact): `dmesg | grep -E 'tb_egpu .*error_detected.*state='` matches at least one line attributable to this event.

**Negative cases:**

- **5a (no AER cap):** `dmesg | grep -E 'tb_egpu aer-unmask: pdev=<BDF>.*no_aer_cap'` matches one line; no error logged.
- **5b (VF):** `dmesg | grep -E 'tb_egpu aer-unmask: pdev=<BDF>.*skip_vf'` matches one line; no error.
- **5c (already clear):** `internal_error_bit=already-clear` in the log line; subsequent AER Internal Error injection still works (proves the no-op path didn't disable anything).

**Idempotence proof:**

- Unbind and re-bind the driver to the same device twice. C2 runs both times; the second run logs `already-clear` (because the first run cleared it and the daemon kept it cleared).

## fake-5090 mechanism mapping

- **Phase 1 + small extension.** Requires writable AER extended capability at config offset `0x100+`, with:
  - Uncorrectable Mask register (offset `0x08` within the AER cap) — read/write.
  - Uncorrectable Status register (offset `0x04`) — write to set, kernel reads to decode.
  - Ability for the daemon to assert the AER message to the kernel's root-port handler when status is written.
- For negative-case 5a (no AER cap), the daemon variant simply omits the extended-cap chain past `0x100`.
- For 5b (VF), requires SR-IOV capability advertisement on a parent PF — heavier; defer to a later phase if too costly.

## Source citations

- `C2-aer-internal-unmask.md` §"Why platform firmware masks Internal Errors" (preamble), §"PCIe r6.0 §6.2.3.2.4 reference" (line ~17 in extracts JSON), §"Idempotent clear-and-reissue" (~30–60), §"VF skip rule" (line ~19), §"No-AER-cap no-op rule" (line ~20).
- aorus-5090-egpu Lever G3-H (UncMaskClear) — predecessor proof that firmware-set masks happen in the wild on this platform.
