# F39 — Xid 154 compute-disconnect race — multi-second window between Xid 79 and Xid 154

| Field | Value |
|---|---|
| **Sources** | C5 (sink-state primitive must propagate to UVM/compute path within bounded time); A4 (telemetry exposes the timing) |
| **Confidence** | field-bug |
| **Predecessor evidence** | NVIDIA issue #1045 (RTX 5080 desktop, Xid 62 → 45 → 119 → 154 cascade timing — Xid 154 fires ~30 s after Xid 119); issue #979 (jciolek 5090 → cross-GPU Xid 154 with timing details); UVM channel-cleanup path analysis |
| **fake-5090 mechanism** | Surprise-removal A primitive + UVM-channel scenario (test harness establishes UVM channels, fires surprise-removal A, measures the latency from sink-set to Xid 154 emission) |

## Symptom (driver view)

Xid 154 is "GPU disconnect from compute applications" — emitted when UVM's cleanup of compute channels on a lost GPU completes. The driver-internal cascade fires (Xid 79 or Xid 119 for the initial bus-loss detection or GSP timeout); the UVM-side teardown runs through its channel-cleanup state machine; Xid 154 fires when UVM declares "all compute work on the dead GPU is unmoored, applications should see CUDA errors."

The timing between Xid 79 and Xid 154 is a known characteristic: typically several seconds to tens of seconds, depending on:

- Number of active UVM channels at sink-set time.
- Number of pages migrated / faulted into the dead GPU's memory.
- Depth of the channel-cleanup state machine (each channel goes through ~5-10 cleanup states, each potentially involving a bounded wait).
- Whether `nvGpuOpsReportFatalError` fires (F19 path) — adds the system-global-flag propagation latency.

The #1045 reporter's case: Xid 119 fires at T+0; Xid 154 fires at T+30 s. During the 30 s gap, CUDA applications calling against the dead GPU are in an indeterminate state — some calls return errors quickly (via funnel-1), some block on UVM channels that haven't been torn down yet, some succeed against stale state and produce wrong results.

F39 is the failure mode at the boundary: even when the driver-internal cascade is well-contained (per F4/F15-F18/F22), the user-visible "GPU is unavailable for compute" signal can lag the bus-loss event by tens of seconds. Workloads written assuming "Xid 79 means stop immediately" can run for an additional 30 s of garbage output before Xid 154 forces them to error.

This is structurally distinct from F22 (silent hard hang — F22 is the case where NO Xid fires; F39 is the case where Xid 79 fires promptly but Xid 154 lags) and from F19 (cross-GPU contamination — F19 is the case where Xid 154 fires on innocent GPUs; F39 is the case where Xid 154 fires correctly on the lost GPU but takes too long).

## Trigger sequence

1. Daemon brings device to a healthy bound state.
2. Test harness establishes UVM channels — runs a CUDA program that allocates managed memory, touches it from CPU and GPU (forcing UVM page-migration), and starts a long-running CUDA kernel using the managed memory.
3. Test fires surprise-removal A. Driver-side cascade detector fires; sink-state set; Xid 79 emitted at T+0.
4. UVM-side teardown begins. The channel-cleanup state machine walks through:
   - Detect lost-state on each active channel.
   - Cancel pending DMA migrations.
   - Tear down per-channel resources.
   - Decrement per-process channel-count.
   - When the last channel on the lost GPU is torn down, emit Xid 154.
5. Measure the wall-clock delta from Xid 79 emission to Xid 154 emission. Pre-patch / under load: 5-30 s. Post-patch: target ≤ 1 s.

## Expected post-patch behaviour

The UVM-side cleanup path's per-channel teardown is fronted by the same sink-state short-circuit as the RM-side paths. When sink-state is set:

1. The per-channel teardown short-circuits the bounded waits (cancel-DMA-migration becomes an immediate cancel-acknowledged-locally; tear-down-resources skips the device-side dereference and reclaims the host-side state immediately).
2. Xid 154 is emitted as soon as the per-channel state-machine reaches the "all channels torn down" point — which under the short-circuited path takes hundreds of milliseconds rather than tens of seconds.

The C5 v4 architecture covers part of this surface (the per-RM-call funnels are in place). F39 documents the UVM-specific extension required: a UVM-side sink-state probe that gates the per-channel teardown's bounded waits.

Implementation lives in the UVM kernel module (`kernel-open/nvidia-uvm/`) rather than the RM core. The probe is `nv_uvm_is_gpu_bus_dead(pUvmGpu)`, analogous to the F26 DRM-side wrapper. The audit identifies the bounded-wait sites in UVM channel-cleanup; per-site gating.

## Assertion shape

**Bounded Xid 154 latency (after surprise-removal A with UVM-active workload):**

- Wall-clock delta from Xid 79 emission to Xid 154 emission MUST be ≤ 1 s post-patch.
- `dmesg --time-format=iso | grep -E 'NVRM: Xid.*(79|154)'` — parse timestamps; compute delta.
- Pre-patch reference (regression-witness): without the F39 fix, delta is 5-30 s.

**Per-channel teardown verification (substrate-side):**

- The substrate exposes per-channel DMA-cancel counters and per-channel resource-tear-down counters.
- After Xid 79 fires, the substrate-side counters MUST reach their final values within bounded time (≤ 500 ms).
- The host-side UVM-internal counters (exposed via debug interface, if available) MUST also reach final values within the same window.

**Application-side fast-fail:**

- The test harness's CUDA program issues a CUDA call (e.g. `cudaStreamSynchronize()`) every 100 ms during the cascade window.
- Pre-Xid-79: calls succeed.
- Between Xid 79 and Xid 154 (pre-patch — the lag window): calls may succeed (with stale data), fail (with CUDA error), or hang (depending on the specific call).
- Post-Xid-154: calls MUST fail with CUDA error.
- Post-patch target: Xid 154 fires within ≤ 1 s of Xid 79, so the "indeterminate window" is ≤ 1 s.

**UVM-source audit (CI):**

- The cascade-scope-audit MUST be extended to cover `kernel-open/nvidia-uvm/`. Per-bounded-wait sites identified and gated.
- New bounded-wait sites introduced in UVM code MUST be reviewed against the sink-state-gate requirement.

**Negative — F39 fix MUST NOT break normal teardown:**

- A clean `cudaDeviceReset()` call (no surprise removal) MUST complete normally — sink-state is not set, the UVM short-circuits do not fire, the standard teardown path runs with the standard latency.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25-F28/F32-F34.
- **UVM-channel-active test scenario.** The test harness must include a CUDA program that establishes UVM channels with active workload at surprise-removal time. The substrate accepts CUDA-related ioctls during the healthy phase (these go through the standard nvidia-uvm.ko paths against the substrate's memory model) and does not need special handling during the cascade.
- **Substrate-side counter exposure.** Substrate exposes per-channel DMA-cancel and resource-tear-down counters via debug registers. The test harness reads these to verify UVM's short-circuit behaviour reaches the substrate-side endpoints.
- **No new substrate primitive** beyond what F22 and F32 already require. The new substrate state is the per-channel counter set.

## Source citations

- NVIDIA issue #1045 (RTX 5080 desktop — Xid 119 → Xid 154 timing measurement)
- NVIDIA issue #979 (jciolek 5090 cross-GPU — Xid 154 timing in cross-contamination case)
- `cascade-scope-audit.md` §"Audit scope boundary — `nvidia-uvm.ko` is partially in scope" (the audit traversed RM-side calls into UVM but did not audit UVM-internal bounded-wait sites)
- `cascade-class-design-v4.md` §"DETECTION INPUTS" §"UVM channel-cleanup latency"
- C5 v4 design — RM-side funnels (covers part of the path; F39 is the UVM-side extension)
- F19 entry (`nvGpuOpsReportFatalError` system-global — related UVM-path failure but different mechanism)
- F22 entry (silent hard hang — F39's no-Xid sibling)
- F26 entry (nvidia_drm teardown hang — analogous cross-module audit-extension pattern; F39 is the UVM equivalent of F26's DRM-extension)
