# F16 — `_issueRpcAndWaitLarge` bypass — multi-chunk Control RPCs skip the GPU_IS_LOST funnel

| Field | Value |
|---|---|
| **Sources** | C5 (v4 funnel set) |
| **Confidence** | field-bug |
| **Predecessor evidence** | `cascade-scope-audit.md` §"Funnel 3 gap at `_issueRpcAndWaitLarge`"; source comparison `rpc.c:1854-1859` vs `rpc.c:2258` |
| **fake-5090 mechanism** | Surprise-removal A primitive + synthetic large-buffer Control RPC scenario |

## Symptom (driver view)

`_issueRpcAndWait` (`rpc.c:1854-1859`) correctly returns `NV_ERR_GPU_IS_LOST` when `PDB_PROP_GPU_IS_LOST` is set, before issuing the RPC. This is the C5 v1 design's primary RPC funnel.

`_issueRpcAndWaitLarge` (`rpc.c:2258`) handles the multi-chunk path when `total_size > maxRpcSize`. `rpcRmApiControl_GSP` (line 11236) routes to `_issueRpcAndWaitLarge` instead of `_issueRpcAndWait` whenever the Control RPC payload exceeds the per-chunk threshold (typical: large per-context state queries, large mapping arrays, multi-engine batch controls). This wrapper does NOT contain the `PDB_PROP_GPU_IS_LOST` short-circuit.

Consequence: a surprise-removal during a multi-chunk Control RPC bypasses the funnel and hits MMIO-completion-timeout territory with the GPU lock held. The driver wedges holding the GPU lock; subsequent driver-internal locks (`nv_lock_safe_irq`, the per-device resserv lock, the GPU mutex) all stall behind the held lock. Visible as: large RPC mid-flight → cable yank → GPU lock not released for the duration of the MMIO completion timeout (kernel default 60 s, but driver-internal poll may be longer) → cascade of unrelated callers stalling behind the lock.

The sibling primitive `_issuePteDescRpc` (also in `rpc.c`) has the same gap by inheritance.

## Trigger sequence

1. Daemon brings the device to a healthy bound state.
2. Test launches a workload that triggers a multi-chunk Control RPC — any operation whose `NVOS54_PARAMETERS.paramsSize > maxRpcSize`. Concrete examples: large per-context engine state dumps, large memory-mapping batch queries, `NV_CTRL_CMD_*` calls with multi-page argument arrays.
3. While the multi-chunk RPC is mid-flight (between chunk N and chunk N+1), test fires surprise-removal A.
4. Driver-side `PDB_PROP_GPU_IS_LOST` gets set by parallel detection paths (the MMIO funnel, the AER callback if registered, the Q-watchdog), but the in-flight `_issueRpcAndWaitLarge` does not check the marker; the next chunk's MMIO completion-poll stalls.

## Expected post-patch behaviour

`_issueRpcAndWaitLarge` mirrors the `_issueRpcAndWait` short-circuit, applied at **function entry AND between every chunk**. When sink-state fires mid-RPC, the next chunk attempt returns `NV_ERR_GPU_IS_LOST` without issuing the next MMIO transaction; the GPU lock is released; downstream callers unwind cleanly with `NV_ERR_GPU_IS_LOST`. The same treatment is applied to `_issuePteDescRpc` to close the sibling gap.

## Assertion shape

**Mid-RPC short-circuit behaviour:**

- During a multi-chunk Control RPC, fire surprise-removal A between chunk N and chunk N+1.
- `_issueRpcAndWaitLarge` MUST return `NV_ERR_GPU_IS_LOST` from the next chunk attempt within bounded time (target: ≤100 ms after sink-state is set).
- The GPU lock MUST be released within the same bounded window — verifiable by a competing call (a lightweight `nvidia-smi -q` from a separate process) succeeding immediately after.

**Telemetry absence (completion timeout MUST NOT fire):**

- `dmesg | grep -cE 'rpc\.c:.*_issueRpcAndWaitLarge.*completion timeout'` MUST be 0 after sink-state fires.
- `dmesg | grep -cE 'NVRM:.*RPC.*timeout.*RmApiControl'` MUST be 0 attributable to the surprise-removal window.

**Telemetry presence (canonical sink log fires once):**

- A single canonical log line per detection: `"_issueRpcAndWaitLarge: GPU lost, aborting multi-chunk RPC at chunk N/M"` SHOULD appear exactly once per episode (per the C5 v4 detector-class log-once discipline).

**Coverage-parity audit (source-side, CI):**

- `grep -nE 'PDB_PROP_GPU_IS_LOST' src/nvidia/.../rpc.c` MUST match in `_issueRpcAndWait`, `_issueRpcAndWaitLarge`, AND `_issuePteDescRpc`. Any single-match (only `_issueRpcAndWait` present) is a regression.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same primitive used by F03/F04/F15.
- **Synthetic multi-chunk RPC scenario.** The substrate must accept a Control RPC whose payload exceeds the per-chunk threshold and process it as multiple MMIO transactions visible in the test harness's transaction trace. The test harness orchestrates: (a) start a large RPC, (b) wait for the first chunk's completion via transaction trace, (c) fire surprise-removal A, (d) confirm the second chunk attempt short-circuits without issuing a TLP. The substrate does not need real GSP firmware to emulate this — the per-chunk MMIO write-then-poll dance is observable at the BAR0 register level.

## Source citations

- `cascade-scope-audit.md` §"Funnel 3: `_issueRpcAndWait` — C5 v1 ✅ + gap at large-buffer path" and §"Gap summary — what Option 1 v4 must add beyond C5 v1+v3"
- `cascade-class-design-v4.md` §"ENTRY-POINT GUARDS" G3 (`_issueRpcAndWaitLarge`)
- `rpc.c:1854-1859` (working funnel); `rpc.c:2258` (gap site); `rpc.c:11236` (`rpcRmApiControl_GSP` routing logic)
- C5 v4 design doc §"Cross-patch impact analysis" — C5 absorbs G3 as a +5-line addition
