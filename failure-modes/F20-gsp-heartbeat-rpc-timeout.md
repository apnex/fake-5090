# F20 — GSP heartbeat / RPC timeout (Xid 119) — firmware wedged, not bus-dead

| Field | Value |
|---|---|
| **Sources** | C5 (v4 detector class [c]); cascade-class-design-v4 §"Detection inputs" |
| **Confidence** | field-bug |
| **Predecessor evidence** | NVIDIA issue #1045 (RTX 5080: Xid 62 → 45 → 119 → 154); #461 (RTX 3060 ARM64: crash-dump RPC triggers Xid 119 cascade) |
| **fake-5090 mechanism** | Wedged-firmware substrate state: MMIO completions succeed but `_kgspRpcRecvPoll` never observes a response within the timeout window |

## Symptom (driver view)

The MMIO funnel (F4) detects bus-dead when reads return the sentinel value. But there is a structurally different failure mode where the chip is electrically present, MMIO transactions complete successfully (the bus is alive), but the GSP firmware is wedged — it doesn't respond to RPCs, doesn't update the heartbeat counter, doesn't process queued commands.

`_kgspRpcRecvPoll` (`kernel_gsp.c` ~2868) is the polling loop that waits for GSP RPC responses. When the GSP firmware is wedged, this loop spins until the fatal timeout (typically 45 seconds) and then classifies the timeout via `_kgspClassifyGspTimeout`. The classifier produces `bIsFatalTimeout == NV_TRUE`, which currently emits Xid 119 ("GSP RPC timeout") but does NOT trip the per-GPU sink primitive — `PDB_PROP_GPU_IS_LOST` and `pci_dev_is_disconnected` are both untouched by the GSP-fatal-timeout path.

Consequence: the cascade-class v4 funnels (F4 MMIO, F16 large-RPC, etc.) all rely on sink-state being set; if sink-state is never set because the only detection signal is Xid 119 (not a bus-dead MMIO read), the funnels never fire. Downstream paths continue to issue RPCs that will time out at 45 s each; the cascade walks for minutes before the user notices.

The 45 s gap between channel error (Xid 62) and GSP timeout (Xid 119) is itself a structural feature: it provides a natural debounce window for one bounded retry per the C3 patch, but only if the GSP timeout is wired to sink-state so the retry's outcome is observable.

## Trigger sequence

1. Daemon brings device to healthy bound state; GSP firmware loaded and responsive (heartbeat counter incrementing, RPC round-trips completing).
2. Test transitions substrate to "GSP wedged" state: MMIO reads to the heartbeat register return a stale value (no longer incrementing); the RPC submission ring accepts new commands but the completion ring never gets new entries.
3. Driver issues an RPC (any operation that goes through `_kgspRpcRecvPoll` — Alloc, Control, Free).
4. `_kgspRpcRecvPoll` spins waiting for the response; the 45-second fatal-timeout fires; `_kgspClassifyGspTimeout` returns `bIsFatalTimeout == NV_TRUE`.
5. Driver emits Xid 119. Without the v4 detector hook, `PDB_PROP_GPU_IS_LOST` is NOT set; subsequent RPCs continue to enter `_kgspRpcRecvPoll`; each runs the 45 s timeout; the wedge walks for ~45 s × N pending RPCs.

## Expected post-patch behaviour

C5 v4 G3 (also referred to as detector class [c]) hooks at the existing fatal-classification site in `_kgspRpcRecvPoll`. When `bIsFatalTimeout == NV_TRUE`, the hook calls `cleanupGpuLostStateAtomic(pGpu, DETECTOR_GSP_HEARTBEAT_TIMEOUT)`. The sink-state primitive sets both markers; subsequent RPCs short-circuit via F4/F16 funnels; the 45 s timeout fires once, not N times.

The hook is ~2 lines of code added at the existing classification branch — no new exposed entry point needed.

## Assertion shape

**Wedge detection (after the first Xid 119 fires):**

- `dmesg | grep -cE 'NVRM: Xid.*119'` SHOULD be exactly 1 per episode (the first GSP timeout sets sink; subsequent RPCs short-circuit and don't reach the timeout path).
- Without the patch: `grep -c` would be N (one per pending RPC, each taking ~45 s).

**Sink-state propagation:**

- After the first Xid 119, the per-GPU sink primitive MUST be set. Verifiable via:
  - `cat /sys/module/nvidia/parameters/...` (if a debug sysfs probe of `PDB_PROP_GPU_IS_LOST` is exposed by an injector patch).
  - Substrate-side: subsequent MMIO reads from the test harness MUST observe the driver issuing zero new MMIO transactions after the sink is set (driver short-circuits at funnel-1).
  - Behavioural: the next `nvidia-smi -q` call MUST return the "GPU is lost" error message within bounded time (≤500 ms) rather than hanging for 45 s.

**Bounded recovery:**

- Total wall-clock time from substrate "wedge" transition to driver settled in lost-GPU state MUST be ≤ 60 seconds (one 45 s GSP timeout + sink propagation + cleanup unwind).

**Telemetry presence:**

- One canonical log line: `"GSP RPC fatal timeout; setting sink (DETECTOR_GSP_HEARTBEAT_TIMEOUT)"` (or the project's chosen canonical-log convention for the G3 detector class) SHOULD fire once.

## fake-5090 mechanism mapping

- **Phase 2 wedged-firmware substrate state.** The substrate must support a configuration transition: from "GSP responsive" (MMIO reads OK, heartbeat counter increments, RPC ring round-trips work) to "GSP wedged" (MMIO reads OK, heartbeat counter stuck, RPC submission accepted but completions never delivered). This is distinct from the surprise-removal A primitive — the bus is alive throughout.
- **Heartbeat-register backing.** Substrate must expose the GSP heartbeat register region; the test harness configures whether the register is "ticking" or "stuck."
- **RPC ring backing.** Substrate must expose the GSP RPC submission/completion ring memory. The test harness configures whether the substrate processes submission-ring writes into completion-ring entries.
- The substrate does NOT need real GSP firmware; only the kernel-visible behaviour of "wedged firmware" matters.

## Source citations

- `cascade-class-design-v4.md` §"DETECTION INPUTS" entry [c] (GSP heartbeat timeout); §"Open questions / risks" item 1 (revised hook location)
- NVIDIA issue #1045 — RTX 5080 desktop, Xid 62 → 45 → 119 → 154 cascade (30 s + 45 s timing pattern)
- NVIDIA issue #461 — RTX 3060 ARM64, crash-dump RPC fn 78 `DUMP_PROTOBUF_COMPONENT` blocks 5 s waiting for dead GPU, triggers Xid 119 (cross-references F25)
- `kernel_gsp.c` ~line 2868 (`_kgspRpcRecvPoll` fatal-timeout branch); `_kgspClassifyGspTimeout` classification helper
- `cascade-scope-audit.md` §"Per-cluster narrative" §"Falcon/GSP cluster"

**Related (root cause):** [[F47]] — the per-poll dead-bus marker coverage gap. The 2026-06-06 #292 A13
live-FAIL showed this heartbeat/RPC timeout as the **un-terminable storm** form: on a dead bus with only
the Linux `os_pci` marker set, `_kgspRpcRecvPoll` (honors only `PDB_PROP_GPU_IS_LOST`) never short-circuits
→ re-storms the `done:`-label heartbeat prints. F47 is the marker-coverage root cause; this F20 timeout is
the visible symptom. Fix = C7 (`osIsGpuBusLost` poll-reader).
