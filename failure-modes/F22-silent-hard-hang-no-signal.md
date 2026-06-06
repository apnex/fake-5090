# F22 — Silent hard hang under sustained inference — no Xid, no AER, no dmesg signal

| Field | Value |
|---|---|
| **Sources** | A2 (bus-loss-watchdog forward-progress detector — closest existing coverage) |
| **Confidence** | community-evidence |
| **Predecessor evidence** | NVIDIA issue #1111 (2026-04-17; RTX PRO 6000 Blackwell, vanilla 595.71.05 Fedora 44, kernel 6.18 stable; sustained llama.cpp inference → silent hard hang, **no Xid, no AER, dmesg silent**) |
| **fake-5090 mechanism** | Wedge-without-signal substrate state: MMIO completions return stale-but-plausible values, RPC ring accepts but does not complete, AER subsystem sees no errors, the only forward-progress signal is the absence of work-completion |

## Symptom (driver view)

The driver sees nothing wrong. There is no Xid. There is no AER signal at the PCIe layer. `dmesg` carries zero events attributable to the failure. The chip presents as healthy by every passive probe (MMIO reads return plausible values; `lspci` is clean; `nvidia-smi` may even succeed if its probe doesn't traverse the wedged path). But the workload is hung — no kernels are completing, no memory transfers are progressing, the GPU is "alive but not making progress."

This is structurally distinct from F05 (Mode B DMA-path silent freeze, which has a similar surface but is reachable via the BAR1/BAR3 mapping path AND is detectable by the Q-watchdog's forward-progress check on DMA completions). F22 is the broader class: any silent wedge under sustained load where the only observable signal is "the workload's progress counter stopped advancing." The Q-watchdog (A2) is the closest existing detector — it polls a forward-progress signal at 4 Hz and can catch DMA-path wedges. But for an in-firmware silent hang, the Q-watchdog's check may itself succeed (the firmware responds to the watchdog ping) while the application-visible work is hung.

The #1111 reporter's case: sustained llama.cpp inference (continuous GPU kernels at high throughput) for unspecified duration, then everything stops. No reboot warning. No clear root cause. No recovery without a host reboot.

The cascade-class-design-v4 analysis identifies this as part of the "forward-progress class" of detector inputs (alongside F05 Q-watchdog DMA wedge and F31 boot-time BAR failure). The v4 design includes Q-watchdog (A2) as a sink-trigger input, which catches forward-progress-class failures and routes them through `cleanupGpuLostStateAtomic` to set per-GPU sink-state. F22 is the part of this class that the Q-watchdog catches; F22's existence as a separate entry is justified because the observability-distinct case (truly nothing in `dmesg`) requires distinct test methodology and assertion shape — the substrate must demonstrate "no signal" rather than "signal X."

## Trigger sequence

1. Daemon brings device to healthy bound state.
2. Test launches a sustained-compute workload (synthetic llama.cpp-equivalent or a long-running CUDA loop that issues continuous kernels with continuous memory transfers).
3. After some bounded interval (the test harness controls when), substrate transitions to "silent-wedge" state:
   - MMIO reads continue to return plausible values (NOT the dead-bus sentinel `0xFFFFFFFF`).
   - The GSP heartbeat register continues to tick (or is held at a value that doesn't trip the firmware-side watchdog).
   - The RPC ring accepts submissions; the substrate's RPC handler responds to "are you alive?" pings with plausible responses.
   - But the engine-work completion ring stops delivering completions for the workload's kernels.
4. Application-side: GPU utilisation stays at 100 % (per nvidia-smi), but the work-counter (tokens-per-second, batches-completed, whatever metric the workload exposes) stops advancing.

## Expected post-patch behaviour

A2's Q-watchdog detects the forward-progress stall by polling a substrate-exposed completion-counter at 4 Hz. After the configurable "N consecutive cycles with no progress" threshold (default 5 cycles = 1.25 s at 4 Hz), the watchdog calls `cleanupGpuLostStateAtomic(pGpu, DETECTOR_QWATCHDOG_DMA_WEDGE)` (per the C5 v4 architecture's renaming of A2's direct `os_pci_set_disconnected` call to route through the sink primitive).

Sink-state set → F4/F16/F20 funnels short-circuit subsequent driver operations → the cascade is contained → A4's close-path telemetry observes the sink-state transition and emits a canonical "GPU wedged via forward-progress detector at $T" log line → operator triage has a single timeline.

The host does NOT recover the GPU automatically — F22 is unrecoverable without chip reset. The patch's job is to convert "host wedged silently" into "host cleanly reports GPU lost and continues running on remaining GPUs (if any)."

## Assertion shape

**Wedge detection (after silent-wedge transition):**

- The Q-watchdog MUST emit its detection event within bounded time: ≤ 5 × `NV_TB_EGPU_BUS_LOSS_WATCHDOG_HZ` × 1000 ms = 1250 ms at 4 Hz default cadence (per A2's tunable cadence).
- `dmesg | grep -cE 'tb_egpu_qwd.*dead bus detected'` (or the project's canonical Q-watchdog detection log) MUST be ≥ 1 within the 1250 ms window.

**Sink-state propagation:**

- Per-GPU sink primitive MUST be set after the Q-watchdog detection. Verifiable via the same path as F20:
  - Substrate-side: subsequent MMIO transactions from the driver MUST be zero (or only "is sink set?" debug reads).
  - Behavioural: `nvidia-smi -q` MUST return "GPU is lost" within ≤500 ms.

**Telemetry presence (absence-of-signal pattern recognition):**

- Before the wedge: `dmesg` has steady-state nvidia logs from normal operation.
- During the wedge BEFORE Q-watchdog fires: `dmesg` has ZERO new nvidia log lines for the duration of the wedge. This "no signal" gap is the diagnostic signature.
- After the Q-watchdog fires: ONE detection log + standard sink-state transition logs.

**Coverage-boundary assertion (negative — F22 is NOT F20 / F21):**

- `dmesg | grep -cE 'NVRM: Xid'` during the wedge MUST be 0 (no Xid fires; this distinguishes F22 from F20 / F21 / F23).
- `dmesg | grep -cE 'AER:'` during the wedge MUST be 0 (no AER signal; this distinguishes F22 from any AER-triggered class).

**Workload-side regression-witness:**

- The test workload's progress counter MUST report zero progress for the duration of the wedge.
- After Q-watchdog fires and sink-state is set, the workload's next CUDA call MUST return an error (not hang) within ≤500 ms.

## fake-5090 mechanism mapping

- **Phase 2 wedge-without-signal substrate state.** This is the most "modelled" of the wedge states — the substrate must keep enough liveness signals to pass passive probes (MMIO reads return plausible values, RPC pings get responses) while withholding the work-completion signal. This is a configurable substrate state, not a single primitive; the test harness composes it from:
  - "Hold heartbeat ticking" mode (heartbeat register increments normally)
  - "Stale-completion ring" mode (engine completion ring stops getting new entries)
  - "RPC probe-friendly" mode (a curated subset of RPCs succeed normally — at minimum, the watchdog's own RPC if it issues one)
- **Forward-progress counter exposure.** The substrate must expose a counter that the A2 watchdog can poll. This counter is exposed via a designated BAR0 register and reflects the substrate's "did I do any work?" semantics. The test harness drives it: counter increments under normal operation; counter freezes when the substrate enters silent-wedge mode.
- **No new substrate primitive** for F22 versus F05 — both use the forward-progress detector route. F22's distinct identity is in the assertion shape (absence-of-signal patterns) and the test workload (sustained compute rather than DMA-path operations).

## Source citations

- NVIDIA issue #1111 — RTX PRO 6000 Blackwell, vanilla 595.71.05, Fedora 44 kernel 6.18, sustained llama.cpp inference → silent hard hang, no Xid, no AER, dmesg silent
- `_community-signal.md` §"#1111 — GSP firmware halt on sm_120 … sustained zero-gap llama.cpp inference — silent hard hang" (tagged as A2's strongest single-match)
- `cascade-class-design-v4.md` §"DETECTION INPUTS" entry [f] (Q-watchdog DMA wedge; A2 routes to sink)
- `A2-bus-loss-watchdog.md` §"Triangulation sources" and the polling-cadence verification at I8
- F05 entry (Mode B DMA-path silent freeze — sibling failure in the same forward-progress class)
