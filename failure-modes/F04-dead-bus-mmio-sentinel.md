# F4 — Dead-bus MMIO readers loop returning 0xFFFFFFFF

| Field | Value |
|---|---|
| **Sources** | C5-crash-safety (sentinel + opaque primitives), A2-bus-loss-watchdog (active detection) |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu Lever I patch 0001 (original `osHandleGpuLost` minimal patch) |
| **fake-5090 mechanism** | BAR-MMIO state machine + surprise-removal A (silent dead-bus) |

## Symptom (driver view)

Distinct from F3 (cleanup asserts): F4 is about *steady-state* MMIO readers on the hot/cold paths. When the bus is dead, every MMIO read returns `0xFFFFFFFF`. Unpatched readers don't recognise this as a sentinel — they treat it as a real register value, often re-read it, often kick off downstream work based on the bogus value (e.g. "interrupt status register says all 32 bits asserted, let me service every IRQ source"). This wastes CPU, fills logs, and produces false work that crosses the dead bus again, snowballing.

F4 is the *MMIO-observable* dead-bus class; the forward-progress family extends into two siblings that present the same surface but resist passive detection: F22 (silent hard hang — no Xid, no AER, no sentinel return because the device is alive but wedged) and F32 (BAR1/BAR3 boundary DMA failure — completions silently dropped at the boundary). F4 is the prerequisite class; F22 and F32 are why A2's active forward-progress watchdog exists in addition to the C5 sentinel-check primitives.

## Trigger sequence

1. Daemon brings device to healthy bound state; driver attaches IRQ handler and runs normal traffic.
2. Daemon transitions to surprise-removal A (silent dead-bus): all MMIO returns `0xFFFFFFFF`, no `.remove()` fires.
3. Spin a workload that exercises MMIO readers:
   - Hot path: a CUDA-equivalent or any user-space client that causes register polling.
   - Cold path: the runtime PM resume path (daemon can simulate by re-triggering whatever the driver uses to detect activity).
4. Observe the MMIO-read rate on the daemon side.

## Expected post-patch behaviour

**C5 layer:**
- `os_pci_read_dword` (and the family `os_pci_*` opaque primitives) check the dead-bus sentinel BEFORE issuing the bus access. If `pdev` is marked disconnected (the Linux marker, propagated by F9 fix), the primitives return the sentinel directly without crossing the bus.
- Bus access count after disconnect should drop to *zero* per reader per episode.

**A2 layer:**
- The Q-watchdog kthread polls `PMC_BOOT_0` at a runtime-tunable interval (default in module param). The *first* dead-bus read in an episode mandatorily logs, latches the episode, and propagates the disconnect to BOTH markers (RM internal + Linux `pdev`).
- Subsequent dead-bus reads in the same episode are silent but still propagate — no log storm.
- An already-disconnected GPU is skipped without re-firing the latch.

## Assertion shape

**Loop-bounding (C5 sentinel):**

- After trigger step 2, MMIO-read count served by daemon to the driver MUST stop growing within `2 × Q-watchdog-poll-interval`. (A2 takes ≤ 1 poll period to detect; C5 short-circuits readers as soon as the marker is propagated.)
- Specifically: instrument the daemon to count reads-per-second; after the latency window, reads-per-second from `nvidia*` PIDs MUST be 0.

**A2 detection:**

- Exactly one mandatory log line per episode: `dmesg | grep -cE 'tb_egpu_qwd.*dead-bus detected'` returns exactly 1 (format string per A2 intent doc — verify).
- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_detections` increments by exactly 1.
- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_last_detection_jiffies` is non-zero and within the trigger window.
- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_state` returns the "lost" / "episode active" value (exact string per A2 intent doc).

**Idempotence check:**

- Trigger step 2 a second time after the first detection — `tb_egpu_qwd_detections` does NOT increment a second time (already-disconnected skip path).
- No additional `dead-bus detected` log line.

**Cycles counter sanity:**

- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_cycles` is monotonically increasing while the kthread is alive (proves kthread is actually running, not deadlocked).

## fake-5090 mechanism mapping

- **Phase 1** substrate (BAR-MMIO with scriptable per-read returns) plus surprise-removal A primitive.
- The daemon's MMIO read counter is itself test infrastructure — assertion #1 requires it.
- No AER, no topology, no bridge needed for F4.

## Source citations

- `C5-crash-safety.md` §"Opaque os_pci_* primitives" and §"Dead-bus sentinel short-circuit" (intent doc lines ~340–400).
- `A2-bus-loss-watchdog.md` §"Requirement: Q-watchdog kthread polls PMC_BOOT_0" (lines ~50–100), §"Five `tb_egpu_qwd_*` sysfs attributes" (lines ~195–250), §"Mandatory log-once on first dead-bus read per episode" (lines ~120–160).
- aorus-5090-egpu patch 0001 (Lever I `osHandleGpuLost` — the minimal predecessor that gave the driver any disconnect awareness at all; C5 + A2 are the production version).
