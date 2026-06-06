# F25 — Crash-dump RPC self-DoS — `DUMP_PROTOBUF_COMPONENT` fn 78 blocks 5 s waiting for dead GPU

| Field | Value |
|---|---|
| **Sources** | C5 (v4 detector class for crash-dump path) |
| **Confidence** | field-bug |
| **Predecessor evidence** | NVIDIA issue #461 — RTX 3060 ARM64 (Tegra Orin AGX), crash-dump RPC fn 78 blocks 5 s waiting for dead GPU during teardown |
| **fake-5090 mechanism** | Surprise-removal A primitive + crash-dump RPC scenario (substrate-side, the RPC fn 78 path must be exercisable; the substrate-injected sink prevents the RPC from completing and the host-side hook must short-circuit) |

## Symptom (driver view)

When the driver detects a cascade-triggering condition (Xid 79, MMIO sentinel reads, GSP timeout — any of F4/F20/F22/F23), it attempts to capture a diagnostic crash dump via `DUMP_PROTOBUF_COMPONENT` RPC function 78 (an RPC to GSP requesting GSP's view of internal state for forensics). This RPC takes the standard RPC path through `_issueRpcAndWait` (the same funnel F16 documents for the large-RPC variant).

When the GPU is already known-lost, the RPC submission still gets queued; the RPC waits for a GSP response that will never come; the wait blocks until the RPC's own timeout (5 seconds per NVIDIA's default for diagnostic RPCs). Multiplied across multiple cascade-detection paths firing in close succession (each attempting its own crash dump), the cumulative wait can be tens of seconds — during which the cascade-class teardown is held up, locks are held, and operator-visible "GPU lost" propagation is delayed.

The #461 reporter's case: RTX 3060 ARM64 (Tegra Orin AGX) eGPU scenario. Cascade triggered; crash-dump RPC blocks for 5 s per attempt; multiple attempts during the cascade window; total teardown takes 15-30 s where 1-2 s would be sufficient. The user-visible symptom is "the host appears wedged for 30 s after I pull the cable"; the actual cause is the diagnostic-collection path attempting impossible work.

The pattern is structurally similar to F16 (`_issueRpcAndWaitLarge` doesn't check sink) but specific to the diagnostic-collection class. The fix is a per-RPC-fn check at the dump-RPC submission site: if `osIsGpuBusDead(pGpu)`, skip the RPC entirely and emit a single canonical log line "crash-dump suppressed; GPU bus dead."

## Trigger sequence

1. Daemon brings device to a healthy bound state.
2. Test fires surprise-removal A. Driver-side cascade detector fires (F4 MMIO funnel, F20 GSP timeout, F22 forward-progress, or F23 cold-boot equivalent).
3. The cascade-handling code path attempts crash-dump collection via `DUMP_PROTOBUF_COMPONENT` RPC fn 78.
4. RPC submission queued; `_issueRpcAndWait` (or `_issueRpcAndWaitLarge` for large crash dumps) enters its wait loop.
5. Pre-patch: the wait blocks for 5 s (per-RPC default timeout) before returning `NV_ERR_TIMEOUT`. If multiple cascade detectors fire and each attempts crash-dump collection, multiple 5-second waits accumulate.
6. During the waits, the GPU lock is held; teardown progress is blocked.

## Expected post-patch behaviour

The dump-RPC submission site is fronted by `osIsGpuBusDead` check (or equivalent sink-state probe). When sink-state is set:

1. The RPC is NOT submitted.
2. A single canonical log line `"crash-dump RPC fn %d suppressed; GPU bus dead at $T"` fires once per suppression event.
3. The cascade-handling path continues immediately to the next teardown step.

The diagnostic value of crash-dump collection is preserved for the normal-failure path (where the GPU is alive enough to respond to the diagnostic RPC, but some specific subsystem failed). It is sacrificed only when the bus itself is dead — in which case the GSP cannot respond anyway, and the dump would be empty or invalid.

This is C5 v4 territory: a detector-class hook at the dump-RPC site, gated on sink-state, with single-fire canonical logging. Implementation: ~3-5 lines at the dump-RPC submission helper.

## Assertion shape

**Cascade-teardown timing (after surprise-removal A):**

- Total wall-clock from surprise-removal A to `nv_pci_remove` completion MUST be ≤ 1 s (target; the cascade-class teardown should be fast once sink is set).
- Pre-patch reference (regression-witness): without the F25 fix, the same scenario takes ≥ 15 s (3 × 5 s RPC timeouts) — used to demonstrate the patch's impact.

**Dump-RPC suppression verification:**

- `dmesg | grep -cE 'crash-dump.*suppressed.*bus dead'` (or the project's chosen canonical log) MUST be ≥ 1 if the cascade-handling path would have attempted a crash dump.
- Substrate-side: the substrate's RPC submission counter MUST NOT increment for `DUMP_PROTOBUF_COMPONENT` (fn 78) submissions after sink-state is set. Verifiable via substrate's RPC trace export.

**Per-attempt timing:**

- Each cascade-detector firing MUST NOT add a 5 s wait. Timing assertion: `(detector_N_fire_time - detector_N-1_fire_time) < 100 ms` for consecutive detector fires within the cascade window.

**Diagnostic-path preservation (negative — F25 fix MUST NOT break the normal case):**

- Run a scenario where the GPU is alive but a specific subsystem (e.g. a CE engine) reports a fault. The crash-dump RPC MUST execute normally — sink-state is not set, the `osIsGpuBusDead` check returns false, the RPC proceeds.
- The diagnostic value of the dump MUST be preserved when the GPU is alive. Verifiable via the substrate's RPC response trace (the substrate returns a valid dump-format response when not in sink state).

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15/F16/F17/F18.
- **Crash-dump RPC scenario.** The substrate must accept RPC fn 78 (`DUMP_PROTOBUF_COMPONENT`) in normal state and produce a plausible response (a minimal valid protobuf-shaped response; the content does not need to be meaningful, only well-formed enough for the host-side parser to not crash). In sink state, the substrate does NOT respond (and does NOT need to — the F25 fix prevents the submission entirely).
- **RPC-submission tracing.** The substrate exposes a per-fn-id submission counter. The test harness reads counters before and after the cascade window; the F25 assertion is "fn 78 counter MUST NOT increment after sink-state is set."
- **No new primitive** — F25 reuses surprise-removal A + the RPC-ring backing F16 already requires. The new substrate capability is the per-fn-id counter export.

## Source citations

- NVIDIA issue #461 — RTX 3060 ARM64 (Tegra Orin AGX), crash-dump RPC fn 78 blocks 5 s waiting for dead GPU
- `_community-signal.md` §"#461 — RTX 3060 ARM64: crash-dump RPC self-DoS"
- `cascade-class-design-v4.md` §"DETECTION INPUTS" entry [j] (dump-RPC site gating); §"Implementation site list"
- `cascade-scope-audit.md` §"Crash-dump RPC submission helper" — site location reference
- F16 entry (sibling RPC-funnel gap for large-RPC class)
- F20 entry (the GSP-timeout class whose detector firing often triggers the cascade window in which F25's dump-RPC submission gets attempted)
