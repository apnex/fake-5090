# F21 — GSP firmware-internal halt under sustained load (Xid 8) — in-firmware fault, not bus-loss

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — community-evidence class; A3 recovery is observability-adjacent but does not fix)* |
| **Confidence** | community-evidence |
| **Predecessor evidence** | NVIDIA issue #1159 (2026-05-22; RTX PRO 6000 Blackwell PCIe, vanilla 595.71.05 + Ubuntu 24.04 kernel 6.17; sustained SGLang FP8 inference → Xid 8 + GSP watchdog timeout) |
| **fake-5090 mechanism** | **Field-observation only.** Substrate cannot emulate GSP firmware-internal scheduling faults; fake-5090's failure-index entry exists to document the gap and the boundary between F20 (RPC-layer GSP wedge, reproducible) and F21 (firmware-internal halt, not reproducible without real GSP firmware) |

## Symptom (driver view)

Under sustained compute load (issue #1159 reported continuous SGLang FP8 inference at high throughput), the GSP firmware encounters an internal fault that manifests as Xid 8 ("GR scheduling error") followed by a GSP-internal watchdog timeout. The GSP firmware's own watchdog catches a scheduling deadlock or context-switch hang inside the firmware — the bus is alive, RPC submission works at the protocol level, but the firmware reports that it cannot make forward progress on the engine schedule.

This is structurally distinct from F20 (`_kgspRpcRecvPoll` host-side timeout — GSP not responding to RPCs) and from F22 (silent hard hang with no logs at all). F21 produces:
- An Xid 8 (NOT Xid 79 / NOT Xid 119 / NOT Xid 154).
- A GSP-internal watchdog timeout signature in `dmesg`.
- No AER signal (the bus is alive at the PCIe layer).
- The driver's view of the chip is "alive but the engine schedule is hung." Recovery requires either a chip reset or a host reboot; the C3/C5/A3 patches do not cover this path because their trigger conditions (`NV_ERR_GPU_IS_LOST`, AER fatal, post-`rm_init_adapter` WPR2-stuck) are not met.

The #1159 reporter sees this on a PCIe-attached card (not eGPU). The cascade-class-design-v4 analysis classifies this as "adjacent to the GSP-LOCKDOWN cascade A3 recovery handles" but NOT an exact match for A3 v1's trigger conditions. The A3 patch's `_community-signal.md` tag is "upstream-PR-rationale strengthening only" — i.e. evidence that the recovery surface is needed, not evidence of an A3 code defect.

## Trigger sequence

**Real-hardware only.** Reproducing F21 requires:

1. A real NVIDIA Blackwell GPU (issue #1159 was RTX PRO 6000; class likely covers consumer Blackwell parts).
2. A sustained compute workload at high throughput — SGLang FP8 inference is the reporter's case; the broader class is "sustained compute keeping the engine schedule maximally busy for hours."
3. Sufficient runtime for the GSP firmware's internal scheduling deadlock or context-switch hang to manifest (the reporter's case took multiple hours).

The fake-5090 v1 substrate cannot reproduce F21 because:
- The failure is internal to the GSP firmware's scheduler; emulating it would require either a real GSP firmware binary running against the substrate (out of scope for vfio-user) or a faithful simulator of the GSP scheduler's internal state machine (multi-order-of-magnitude scope expansion).
- The substrate's view of "GSP wedged" (F20) covers the host-observable symptom but not the in-firmware mechanism that produces Xid 8 specifically.

F21 routes to issue-tracker correlation and real-hardware soak testing (which we do not currently run on this class — the apnex rig has Blackwell but the sustained-FP8-inference workload is not in the current soak menu).

## Expected post-patch behaviour

**No injector patch addresses F21 directly.** Three indirect levers:

1. **Detection signal forwarded to sink-state.** If the C5 v4 architecture is extended to add a detector for "Xid 8 + GSP watchdog timeout" alongside the existing detector classes (G3 GSP heartbeat timeout, etc.), the sink primitive would fire, the cascade-class funnels would short-circuit subsequent operations, and the host would converge to a clean "GPU is lost" state without further timeouts. This would NOT recover the GPU — it would only stop the cascade from compounding.

2. **A3 recovery extension.** A3's post-rmInit-FAIL trigger does NOT fire on Xid 8 (no `rm_init_adapter` failure occurred). A3's AER NEED_RESET path does NOT fire (no AER signal). Extending A3 to accept Xid 8 + GSP watchdog as a trigger would require careful scope work — the current A3 design is explicitly bounded.

3. **Upstream-PR strengthening.** F21 is part of the broader evidence body that the recovery surface introduced by A3 is needed in vanilla even for non-eGPU users. Citation for the upstream PR body, not driver-side hardening.

## Assertion shape

**fake-5090 v1 cannot assert post-patch behaviour for F21 directly.** The substrate cannot reproduce the trigger. The failure-index entry exists to:

1. Document the coverage boundary (no fake-5090 assertion).
2. Provide a cross-reference for operators triaging real-hardware Xid 8 + GSP watchdog incidents.
3. Capture the field-evidence record so future scoping decisions about A3 extension have a starting point.

**Real-hardware-only assertions (if a soak suite is ever spun up for this class):**

- Long-soak SGLang FP8 inference on real Blackwell hardware for >24 h.
- `dmesg | grep -cE 'NVRM: Xid.*8'` attributable to GSP watchdog (NOT GR scheduling errors from other causes).
- If a future A3-extension or C5-extension patch lands targeting this class, post-patch: the count should be 0 OR the count should be ≥1 followed by clean sink-state transition (depending on which lever was chosen).

**Negative assertion (what F21 is NOT):**

- F21 is NOT F20 — `_kgspRpcRecvPoll` does not necessarily fire (the host-side RPC layer may be working; the in-firmware engine schedule is what's hung).
- F21 is NOT F22 — there ARE logs (Xid 8 + GSP watchdog); F22 is the silent case.
- F21 is NOT F18 — no `osinit.c:2462` assertion fires (no `gpuStateDestroy` invocation).

## fake-5090 mechanism mapping

- **Field-observation only.** The substrate cannot emulate the in-firmware mechanism.
- **Boundary-documentation role.** F21 is in the index to make explicit that fake-5090 v1's coverage stops at the host/firmware interface; failures originating inside the firmware (Xid 8 GSP watchdog, Xid 119 GSP RPC timeout when caused by in-firmware deadlock rather than host-side wedge, GSP-LOCKDOWN from internal causes) are out of scope.
- **Future-work hook.** A fake-5090 v2 that bundles a real GSP firmware image (or a faithful scheduler simulator) could reproduce this class — explicitly out of scope for v1.

## Source citations

- NVIDIA issue #1159 — RTX PRO 6000 Blackwell PCIe, vanilla 595.71.05, Ubuntu 24.04 kernel 6.17, sustained SGLang FP8 inference → Xid 8 + GSP watchdog timeout
- `_community-signal.md` §"#1159 — RTX PRO 6000 Blackwell: Xid 8 / GSP watchdog timeout under sustained SGLang FP8 inference"
- `A3-recovery.md` §"A3-recovery-I16 — Community-signal #1159 (Xid 8 / GSP watchdog timeout) — does it surface a v3 defect?" (verification: does not exercise A3 v1 trigger conditions; framing is upstream-PR-rationale strengthening)
- `cascade-class-design-v4.md` §"What v4 explicitly does NOT do" (firmware-internal faults are out of scope)
