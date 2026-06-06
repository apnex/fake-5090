# F34 — Spurious plug-event burst — TB controller emits rapid plug/unplug events from cable noise

| Field | Value |
|---|---|
| **Sources** | A3 (recovery — H1 cap + H3 persistent-kill prevent burst-induced storm); A4 (telemetry surfaces burst-detection event) |
| **Confidence** | community-evidence |
| **Predecessor evidence** | aorus-5090-egpu PM #11 (observer-induced failure investigation; surfaced the burst-event class while debugging); TB4 specification §hot-plug-event debounce expectations; community reports of "GPU keeps redetecting itself" on marginal cables |
| **fake-5090 mechanism** | Hot-plug-event burst primitive: substrate exposes a "fire N plug-events in M milliseconds" test primitive; A3's debounce / H1 cap should suppress the burst from triggering N independent recovery attempts |

## Symptom (driver view)

A marginal TB/USB4 cable, a flaky retimer, or an EMI source can cause the TB controller to emit rapid hot-plug events — alternating "device arrived" / "device removed" within milliseconds, possibly tens of times before the situation stabilises. Each event normally triggers a kernel-side PCI re-enumeration cycle: device removed → `nv_pci_remove` runs → device arrived → `nv_pci_probe` runs → `rm_init_adapter` runs.

Under a burst of N events in rapid succession, the naïve kernel-side behaviour would spawn N concurrent enumeration cycles, exhaust the workqueue, fill `dmesg` with re-enum messages, and likely produce one or more wedged probe attempts when the next event fires mid-probe. The driver-visible symptom is recovery storm — same surface as F10 — but caused by burst events rather than per-failure A3 retries.

The injector's existing defences:

1. **A3's H1 cap (3 attempts max)** caps the recovery cycles per-probe-failure. If the burst causes 3 probe failures, A3 marks persistent and stops attempting; the next plug-event after persistent is rejected without entering `rm_init_adapter`.
2. **A3's H3 persistent-kill** combined with A4's telemetry surface means operator tooling can detect "this device is in burst-event-storm state" and respond (e.g. disable the port via `tbtools` until the operator investigates the cable).
3. **Kernel-side hot-plug debounce** (configurable via TB-controller firmware settings; the kernel typically has its own per-port debounce window of ~100 ms) provides some upstream protection.

F34's failure mode is specifically the case where the burst is fast enough to evade the kernel debounce (e.g. 5 events in 50 ms with the kernel debounce at 100 ms — the kernel sees only the first event, but the substrate's emit-rate is what the operator should be able to investigate via telemetry). The injector's responsibility is bounded handling and telemetry exposure, not burst prevention.

## Trigger sequence

1. Daemon brings device to a healthy bound state.
2. Test fires the burst primitive: substrate emits a configurable sequence of plug/unplug events — e.g. 5 events at 10 ms intervals (faster than the kernel debounce) or 10 events at 200 ms intervals (each event passes the debounce; the kernel sees all 10).
3. Driver-side enumeration cycles fire (modulo the debounce). For each, A3's recovery attempt may trigger; the H1 cap holds the total attempts to 3.
4. After 3 failed re-enumerations, A3 marks persistent; subsequent burst events do not re-trigger enumeration.
5. A4's telemetry surface reports a burst-detection event with the count of plug-events received within the burst-window.

## Expected post-patch behaviour

The injector's existing layers (A3 H1 cap, A3 H3 persistent-kill, A4 telemetry) provide the bounded-handling contract. F34's specific addition over the F10 baseline is the **burst-detection telemetry** — A4 maintains a per-port event-counter with timestamps; when more than N events arrive within a window of M milliseconds (defaults: N=5, M=1000), A4 raises a burst-detection event surfaced via sysfs.

Operator tooling reads the surface, identifies the burst-state, and can:

1. Surface a clear "burst-event-storm detected on port <X> — cable may be marginal" message.
2. Optionally disable the port via `tbtools` to stop the storm.
3. Page the user / log to a central monitoring system.

The kernel's own debounce window remains the primary line of defence; F34's role is to make burst-state observable so operator tooling can respond intelligently.

## Assertion shape

**Bounded handling (same as F10):**

- After the burst primitive fires N events, A3's recovery attempts MUST be ≤ 3 (H1 cap).
- A3 MUST mark persistent after the 3rd failed re-enumeration.
- Subsequent burst events MUST NOT trigger further `rm_init_adapter` invocations.

**Burst-detection telemetry:**

- After a burst of N events within M ms (with N > 5, M ≤ 1000), A4's burst-detection sysfs surface MUST report:
  - Event count: N.
  - Window duration: ≤ M ms.
  - First-event-timestamp and last-event-timestamp.
- `cat /sys/bus/pci/devices/<BDF>/<A4-burst-state-path>` MUST contain the burst state when active.

**Bounded total time:**

- Total wall-clock from first burst event to host settled in persistent-kill state MUST be ≤ 30 s (consistent with F23/F27 — A3's bounded retry window).

**Negative — single events MUST NOT register as burst:**

- A single plug-event (no burst) MUST NOT trigger the burst-detection surface.
- A slow series of plug-events (N events spread over more than M ms) MUST NOT trigger the burst-detection surface.

**Negative — burst-detection MUST NOT prevent legitimate hot-plug:**

- After a burst causes persistent-kill, an operator-driven port reset (e.g. `echo 1 > /sys/bus/pci/devices/<BDF>/remove` followed by re-enumeration) MUST allow the device to come back if the underlying cause has been fixed. The persistent-kill marker MUST be clearable on operator action.

## fake-5090 mechanism mapping

- **Phase 3 hot-plug-event burst primitive.** Substrate must support emitting a configurable sequence of plug/unplug events. This is a Phase 3 primitive (more complex than Phase 2 surprise-removal A) because it requires the substrate to participate in the host's hot-plug enumeration framework (signalling "device arrived" and "device removed" events through the substrate's transport adapter).
- **Burst configuration:** N events, M ms inter-event interval, optional jitter. Test harness fires the burst; substrate emits the events.
- **Event-counter exposure (A4 dependency).** A4 maintains the per-port event counter; the substrate's burst-emission contributes events that A4 counts; the substrate does not itself need to track the burst state — only emit the events.

## Source citations

- aorus-5090-egpu PM #11 (observer-induced failure investigation; burst-event class discovered)
- TB4/USB4 specification §hot-plug-event debounce expectations
- Linux kernel `drivers/thunderbolt/` — kernel-side hot-plug debounce
- A3 intent doc — H1 cap + H3 persistent-kill (already covers F10's bounded-retry surface)
- A4 intent doc — telemetry-surface for burst-detection (F34-specific addition)
- F10 entry (recovery storm — F34's failure-class predecessor; F10 was unbounded per-failure, F34 is unbounded per-burst)
- F33 entry (signal-loss vs CC-pin — adjacent transport-classification issue)
