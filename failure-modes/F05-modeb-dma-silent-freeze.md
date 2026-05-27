# F5 — Mode B DMA-path silent freeze not reachable by passive polling

| Field | Value |
|---|---|
| **Sources** | A2-bus-loss-watchdog |
| **Confidence** | hypothesis (design-time hardening; the *underlying* Mode B freeze is field-observed but A2's coverage gap fix is preventive) |
| **Predecessor evidence** | aorus-5090-egpu Lever Q patches 0010–0015 (the original Q-watchdog series) |
| **fake-5090 mechanism** | BAR-MMIO state machine (watchdog-detected transition) |

## Symptom (driver view)

NVIDIA issue #979 / aorus-5090 PM #2 ("Mode B silent freeze") happens between user-facing IO operations. The driver is idle from the kernel-VFS perspective but in-flight DMA can still freeze the bus. A passive kernel-side poller that only inspects MMIO when user-space asks (`nvidia-smi -L` open/close, etc.) cannot see the freeze in the window between operations — the GPU is wedged, but nothing on the host *tries* to read it. By the time the next user-space request arrives, the freeze has either spontaneously recovered (rare) or has progressed to a more catastrophic state.

The driver's perceived problem: *we have no way to know the device is dead until someone touches it*, and by then it's too late to localise the failure to a narrow time window for forensics.

## Trigger sequence

1. Daemon brings device to healthy bound state.
2. Driver attaches; A2's `tb_egpu_qwd_init` runs from probe.
3. *No user-space traffic.* Test asserts the system is idle (no `nvidia-smi` polling).
4. Daemon transitions BAR0 to "dead-bus" state at time T0 (MMIO reads return `0xFFFFFFFF`). Config space remains live (Mode B signature: PCIe link still up at the bridge level, but the device's internal fabric is hung).
5. Test waits for `T0 + 1.5 × Q-watchdog-poll-interval`.
6. Test reads sysfs to determine whether the watchdog detected the event without any user-space trigger.

## Expected post-patch behaviour

A2's `tb_egpu_qwd_*` kthread runs *independently* of user-space activity. It polls `PMC_BOOT_0` at the configured interval (one kthread per probed eGPU). When the poll sees the dead-bus signature (`0xFFFFFFFF`), it:

1. Latches the episode (idempotent counter increment, single mandatory log line, sysfs state transition).
2. Propagates the disconnect marker to BOTH the RM internal and Linux `pdev`-side markers (F9 fix).
3. Records `last_detection_jiffies` so forensics can correlate with other timeline sources.

Net effect: the Mode B silent freeze is converted from "undetected indefinitely" to "detected within ≤ 1 × poll-interval of occurrence." This is the load-bearing pre-condition for *all* the recovery work in A3 — A3 cannot trigger if A2 doesn't detect.

## Assertion shape

**Without-user-space detection:**

- Within `1.5 × poll_interval_jiffies` of step 4, the following all become true *without any user-space activity*:
  - `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_detections` ≥ 1.
  - `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_last_detection_jiffies` is within the trigger window.
  - `dmesg` contains exactly one `dead-bus detected` line (per A2 intent doc format).

**Cycle counter as liveness proof:**

- Before step 4 (device healthy), `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_cycles` is monotonically increasing over a 5-second observation — proves the kthread is genuinely running.
- The cycles counter must show ≥ `(observation_seconds × HZ) / poll_interval_jiffies` increments.

**Tunability sanity:**

- Change `/sys/module/nvidia/parameters/NVreg_TbEgpuQwdPollIntervalMs` (or the actual param name per A5 intent doc) to a smaller value; verify detection latency in step 5 drops correspondingly.

**Graceful-degrade check (kthread allocation failure):**

- Inject a kthread allocation failure at probe time (test infrastructure on the kernel side, not the daemon). Driver MUST continue to bind the device (graceful degrade — the A2 capability is absent but the driver works without it). Assertion: `readlink /sys/bus/pci/devices/<BDF>/driver` still resolves to `nvidia`; `tb_egpu_qwd_*` sysfs attributes are absent.

## fake-5090 mechanism mapping

- **Phase 1** BAR-MMIO state machine is sufficient.
- The test fixture must isolate "no user-space activity" — easiest path is to not run `nvidia-smi` or any client during steps 4–6, and instrument the daemon to count MMIO reads from non-kthread PIDs (must be 0 in the window).
- No AER, no topology, no bridge needed for F5.

## Source citations

- `A2-bus-loss-watchdog.md` §"Why passive polling is insufficient" (preamble + Mode B reference), §"Q-watchdog kthread design" (~50–110), §"Sysfs surface" (~195–250), §"Graceful degrade on kthread alloc failure" (line ~200 in extracts JSON; intent doc §"Failure modes of the kthread itself").
- aorus-5090-egpu patches 0010–0015 (Lever Q — the predecessor kthread series; A2 generalises them into a single rate-limited watchdog with the five sysfs attributes).
- aorus-5090-egpu `docs/failure-modes-index.md` row 2 (Mode B silent freeze).
