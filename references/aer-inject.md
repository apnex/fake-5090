# aer-inject reference

`aer-inject` is a kernel module + userspace tool that synthesises
PCIe AER (Advanced Error Reporting) events against a real device.
Useful for exercising the injector's error-handler patches (C2, C3,
C4, A2) against the real 5090 when attached.

## Host status on obpc

```
/lib/modules/7.0.9-204.fc44.x86_64/kernel/drivers/pci/pcie/aer_inject.ko.xz
```

Module **ships in the running kernel**. Not loaded by default —
`modprobe aer_inject` when needed.

## How it works

1. `modprobe aer_inject` registers a debugfs interface under
   `/sys/kernel/debug/aer_inject/`.
2. Userspace tool (`aer-inject`, separate package, or a small custom
   script) writes a binary record describing the error: target BDF,
   error type (correctable / non-fatal / fatal), header log,
   capability mask, etc.
3. Kernel routes the synthesised error to the AER handler chain as
   if the hardware had raised it.
4. PCIe driver's `pci_error_handlers` callbacks fire normally.

## What we'd validate with it (real 5090 attached)

| Patch                  | Inject what                          | Expect                                              |
|------------------------|--------------------------------------|-----------------------------------------------------|
| C2-aer-internal-unmask | Correctable + non-fatal AER          | Driver no longer swallows; unmask path runs         |
| C3-gpu-lost-retry      | Fatal AER then surprise removal      | Retry logic engages, gives up gracefully            |
| C4-err-handlers-scaffold | Non-fatal AER                      | Scaffold callbacks invoked, telemetry recorded      |
| A2-bus-loss-watchdog   | Fatal AER (loss-of-link class)       | Watchdog detects bus-loss within configured SLA     |

## Companion tools

- **`setpci`** — direct config-space writes. Useful for forcing link
  retraining, masking/unmasking capabilities mid-test.
- **APEI EINJ** (`/sys/kernel/debug/apei/einj/`) — ACPI Error
  Injection. Platform-level errors at the root complex. Requires
  BIOS/firmware support; check with `ls /sys/kernel/debug/apei/einj/`
  on `obpc` when needed.
- **Thunderbolt detach** —
  `echo 0 > /sys/bus/thunderbolt/devices/<dev>/authorized` for
  controlled detach; `echo 1 > ...` for re-attach. Closest software
  analogue to physical unplug for the A4 close-path telemetry tests.

## Recommended Phase-0 harness

A small bash wrapper in `fake-5090/scripts/`:

```
scripts/aer-harness.sh inject correctable <BDF>
scripts/aer-harness.sh inject fatal <BDF>
scripts/aer-harness.sh tb-detach <tb-device>
scripts/aer-harness.sh tb-attach <tb-device>
```

That's the highest-confidence validation we can run today, against
the actual fork on the actual card, with zero new infrastructure.
Build it before fake-5090 itself.

## Pitfalls

- `aer_inject` ignores root-port-level errors by default. Use the
  `force` knob (debugfs `force_capable`) to override for testing.
- A fatal-class injection will often wedge the device; expect to
  rmmod/reload the driver between runs.
- EINJ requires ACPI support compiled in *and* BIOS/firmware support
  *and* root. Not all platforms expose it. The Meteor Lake-P
  consumer BIOS on `obpc` may or may not — TBC.
