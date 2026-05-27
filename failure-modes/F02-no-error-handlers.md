# F2 — `pci_error_handlers` absent → `pcie_do_recovery` aborts

| Field | Value |
|---|---|
| **Sources** | C4-err-handlers-scaffold (scaffold), A3-recovery (real handler bodies) |
| **Confidence** | hypothesis (no observed field incident on this exact path; defensive against the upstream gap) |
| **Predecessor evidence** | aorus-5090 Lever M-base patch 0007 (the original "register error_handlers" minimal patch) |
| **fake-5090 mechanism** | AER fatal injection + dispatch-trace observation |

## Symptom (driver view)

NVIDIA's upstream driver does not register a `pci_error_handlers` ops table. When an AER Fatal or Internal Error fires for an NVIDIA device, the kernel's `pcie_do_recovery()` walks the driver list, finds no error handlers, and aborts with the kernel-side message `"no error handlers"` — the bus is *surrendered* without the driver ever receiving an `.error_detected` callback. The driver cannot participate in recovery and cannot even log that the event happened from its own perspective.

## Trigger sequence

1. Daemon brings device to bound/healthy state; driver has called `probe()` and registered.
2. Test arms AER injection: target = device's BDF, error class = Fatal (`PCIE_FATAL`), specific bit = `DEVICE_RESET` or `UnsupportedRequest` (any fatal-severity bit serves).
3. Test triggers the injection (vfio-user AER callback fires the bit into the device's AER status register and asserts the appropriate root-port message).
4. Kernel AER code dispatches `pcie_do_recovery()` against the device.
5. **Unpatched path:** kernel logs `"no error handlers"` and surrenders.
6. **C4-only build:** `.error_detected` returns `pci_channel_io_perm_failure` → kernel logs `DISCONNECT` and surrenders cleanly (honest "we have no reinit").
7. **C4+A3 build:** `.error_detected` returns `pci_channel_io_normal` (non-fatal gated state) or `pci_channel_io_need_reset` → kernel proceeds to `.slot_reset` → A3 work-handler drives `pci_reset_bus` on the upstream Thunderbolt bridge → recovery proceeds.

The *same* AER injection must produce three distinguishable terminal states across the three build variants.

## Expected post-patch behaviour

**C4-only**: driver's `.error_detected` is called (proves registration worked). It classifies non-fatal → `CAN_RECOVER` and fatal/frozen → `DISCONNECT`. `.mmio_enabled` returns `RECOVERED` (no-op stub). `.slot_reset` returns `DISCONNECT` (the honest "this build has no reinit"). `.resume` logs and returns.

**C4+A3**: `.error_detected` returns `NEED_RESET` when recovery gates are open. `.slot_reset` reads `PMC_BOOT_0` post-reset; healthy → `RECOVERED` + `.resume` fires; still dead → `DISCONNECT` + `surrender_count` increments + `PERMANENT_FAIL` uevent.

## Assertion shape

**Registration check (build-independent):**

- `dmesg | grep -E 'nvidia .* registered.*pci_error_handlers'` — at least one line at probe time (or equivalent — exact format depends on injector logging).

**C4-only build, AER fatal injection:**

- `dmesg | grep -E 'pcieport.*AER.*Fatal'` — kernel observes the AER event.
- `dmesg` MUST NOT contain `"no error handlers"`.
- A driver-side log line proving `.error_detected` was called: `dmesg | grep -E 'tb_egpu .*error_detected.* state=(non-fatal|fatal)'` (exact format per the C4 intent doc's logging table).
- Terminal state: `DISCONNECT`. `readlink /sys/bus/pci/devices/<BDF>/driver` becomes absent (driver unbound).

**C4+A3 build, AER fatal injection on healthy device:**

- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_attempt_count` increments by 1 within the recovery window.
- Bus-reset event on the upstream bridge: `dmesg | grep -E 'pcieport [0-9a-f:.]+: .*reset.*bus'` matching the upstream TB bridge BDF (NOT the GPU's own BDF — this is load-bearing).
- On success: `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_state` returns `READY`; `TB_EGPU_GPU_STATE=READY` uevent fired.
- Driver re-bound: `readlink /sys/bus/pci/devices/<BDF>/driver` still resolves to `nvidia`.

## fake-5090 mechanism mapping

- **Phase 2 — AER injection** capability required: writable AER Uncorrectable Status register + ability to assert the corresponding root-port message to the kernel's AER framework.
- The vfio-user device must implement a credible Thunderbolt-topology *upstream bridge* so `pci_reset_bus` on the bridge does the right thing (Surprise-removal mechanism B/C territory). For F2 specifically the bridge can be a simple passthrough that accepts `pci_reset_bus` and re-emerges with the device healthy.
- The three build-variant terminal states (no-handler / C4-only DISCONNECT / C4+A3 RECOVERED) are the cleanest single fixture demonstrating the injector stack composes correctly.

## Source citations

- `C4-err-handlers-scaffold.md` §"Requirement: register pci_error_handlers" and §"State-aware error_detected classification" (lines ~40–110); logging table at lines ~150–200.
- `A3-recovery.md` §".error_detected with healthy gates returns NEED_RESET" (line ~87 in extracts JSON; intent doc §"Recovery state machine"); §"Work handler drives pci_reset_bus on UPSTREAM bridge" (intent doc lines ~120–145); sysfs surface lines ~312–345.
- aorus-5090-egpu patch 0007 (the minimal predecessor that proved the path was registrable but had no reinit body).
