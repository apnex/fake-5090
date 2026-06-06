# F43 — GSP RM-unload RPC latency exceeds a naive teardown timeout budget

> **Split provenance (2026-05-30).** This entry was carved out of the former "F40 shutdown arm." It is filed separately because **it is not a wedge** — `rm_shutdown_adapter` completes normally in ~600 ms; the "hangs every teardown" reading was *iatrogenic* (a too-tight A7 timeout budget cutting off a healthy operation). F40 is now the open-arm re-init wedge only; the genuine residual lifecycle risk on this path is the leaked-worker UAF, catalogued as [[F42]].

| Field | Value |
|---|---|
| **Sources** | **A7** (f40b-bounded-wait-shutdown — the wrapper whose budget must accommodate this latency; default `NVreg_TbEgpuShutdownTimeoutMs = 1200`); SH-1 (latency), SH-2 (RPC identity via PMU sampling), SH-3 (rmmod-path + UAF guard) |
| **Confidence** | field-bug for the *latency* (~600 ms directly measured, n≥4); the *failure* it produces is iatrogenic (premature GPU-lost) when a bounded-wait budget is set below it. This is a **timing-contract** entry, not a chip disease. |
| **Predecessor evidence** | `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/shutdown-hang-ledger.md`; `experiments/SH-1-rm-shutdown-latency-poll-vs-block.md`; `experiments/SH-2-eBPF-register-identity.md`; A7 Test A validation `design/A7-test-A-validation-2026-05-29.md` |
| **fake-5090 mechanism** | Substrate models a configurable GSP RM-unload latency on the shutdown path (default ~600 ms, chip alive throughout: AER clean, `PMC_BOOT_0 = 0x1b2000a1`, worker in R-state). The test asserts that the deployed bounded-wait budget exceeds the modelled latency — a regression witness against a too-tight budget re-declaring a healthy GPU lost. |

## Symptom (driver view)

On **every healthy teardown** of a Thunderbolt-attached GPU — both the non-persistent `LAST-CLOSE` path (`nv_stop_device → nv_shutdown_adapter`) and the `rmmod` path (`nv_pci_remove_helper → nv_shutdown_adapter`) — the `rm_shutdown_adapter` step busy-runs for ~600 ms before completing **successfully**. The worker sits in R-state with one CPU core pegged; AER stays clean; `PMC_BOOT_0` reads the chip-identity value throughout. The chip is alive and the operation is making progress, not hung.

If a bounded-wait wrapper guards this step with a budget **below** the ~600 ms latency (A7's original 200 ms — ~3× too tight), the wrapper times out and calls `rm_cleanup_gpu_lost_state`, **declaring the GPU lost on a perfectly healthy teardown**. That premature GPU-lost is the failure — and it is iatrogenic: the chip was fine, the call would have completed, and the wrapper manufactured the loss.

## Mechanism (SH-2 — PMU sampling, kprobe was IBT-blocked)

The ~600 ms is a **GSP RM-unload RPC round-trip**, not a stuck register:

```
rm_shutdown_adapter
  → kgspUnloadRm_IMPL
    → _issueRpcAndWait
      → _kgspRpcRecvPoll       ← busy-polls the GSP mailbox / heartbeat until the
                                 GSP firmware reports the RM unload complete (~600 ms over TB4)
```

`rm_disable_adapter` — the teardown step immediately before — completes within budget every time; the latency is specific to the final GSP-unload step. The ~600 ms is a property of the TB4 RPC round-trip and the GSP firmware unload sequence; it is **not reducible by the host**. (Note: the *init* path uses the same `_kgspRpcRecvPoll` primitive — that is the lead into the [[F40]] open-arm wedge, where the GSP-side poll never completes.)

## Distinction from neighbours

- **Not [[F40]].** F40 is the open-arm re-init wedge: a chip *dead for init* where `RmInitAdapter`'s GSP lockdown-release never completes → multi-second lock-holding busy-poll → host deadlock (n=13 reboots). F43's chip is alive and the RPC completes.
- **Not [[F42]].** F42 is the leaked-worker UAF: if `rmmod` completes while the ~600 ms worker is still in flight, the worker executes in freed `nvidia.ko` `.text`/`nvl`. That is the genuine residual lifecycle bug on this path; A7's SH-3 `flush_work` guard joins the worker before teardown. F43 is purely the latency-vs-budget contract.
- **Not [[F20]].** F20 is a GSP heartbeat/RPC *timeout* where the firmware is actually wedged (Xid 119). F43's RPC completes; nothing is wedged.

## Trigger sequence

1. Cold boot; deploy the injector → `nvidia.ko` bound + persistence engaged.
2. (Optionally) run any healthy workload.
3. `kubectl exec <injector-pod> -- /entrypoint.sh uninstall` (rmmod path) **or** a non-persistent `LAST-CLOSE`.
4. `rm_shutdown_adapter` runs the GSP-unload RPC for ~600 ms, then returns.

No F40-precondition is required — this fires on any healthy teardown.

## Expected post-patch behaviour

A7's bounded-wait wrapper uses `NVreg_TbEgpuShutdownTimeoutMs` (default **1200 ms**, ≥2× the measured ~600 ms). The call completes within budget — wrapper logs `rm_shutdown_adapter completed within budget` — and teardown proceeds normally with no premature GPU-lost. SH-3 Rung-1 measured the rmmod-path at **~649 ms < 1200 ms** (n=1, apnex.22 uninstall; host clean).

**Scope / open question (v5).** Because the RPC reliably completes on a healthy chip, A7's *shutdown-arm* bounded-wait may be **deletable** — its only remaining job is to bound the pathological case (a chip that genuinely dies mid-unload), which is [[F40]]/[[F42]] territory, not this entry. Whether A7-shutdown earns its keep versus simply widening the budget is a strategic-review (v5) question. What is NOT optional on this path is the leaked-worker join — see [[F42]].

## Assertion shape

- **Regression-witness (the iatrogenic failure):** with the substrate's GSP-unload latency set to ~600 ms and the bounded-wait budget set **below** it (e.g. 200 ms), the wrapper MUST emit `rm_shutdown_adapter timed out after <budget> ms — declaring GPU lost` and set the C5 sink. This reproduces the iatrogenic premature-GPU-lost.
- **Correct behaviour (budget ≥ latency):** with the deployed budget (1200 ms), the wrapper MUST emit `rm_shutdown_adapter completed within budget` and teardown MUST continue with no GPU-lost classification and no Xid.
- **Negative assertions during a healthy teardown:** `dmesg | grep -cE 'NVRM: Xid'` MUST be 0; `dmesg | grep -cE 'AER:'` (excluding the C4 probe-time hardening line) MUST be 0; `PMC_BOOT_0` MUST remain readable (`0x1b2000a1`) throughout — the chip is alive.

## fake-5090 mechanism mapping

Phase-1 substrate plus a shutdown-path RPC-latency parameter. The ~600 ms figure is TB4-RPC-round-trip-specific (real-HW-corroborated); the substrate models it as a configurable latency so the test can sweep budget-vs-latency without real hardware.

## Source citations

- `shutdown-hang-ledger.md` — SHUTDOWN ARM RESOLVED 2026-05-30; the iatrogenic-budget headline; budget 200 → 1200 ms.
- `experiments/SH-1-rm-shutdown-latency-poll-vs-block.md` — ~600 ms latency, busy-poll, n=3 close-path.
- `experiments/SH-2-eBPF-register-identity.md` — the `kgspUnloadRm_IMPL → _issueRpcAndWait → _kgspRpcRecvPoll` RPC identity (PMU sampling; kprobe IBT-blocked).
- `patches/addon/A7-f40b-bounded-wait-shutdown.patch` — `NVreg_TbEgpuShutdownTimeoutMs = 1200`.
- `design/A7-test-A-validation-2026-05-29.md` — the original n=2 "fires on every healthy rmmod" observation (later reinterpreted as the 200 ms guillotine, not a hang).

## Cross-references

- [[F40]] — open-arm re-init wedge (the genuine disease). Shares the `_kgspRpcRecvPoll` poll primitive — on the *init* side, where the GSP never completes the poll.
- [[F42]] — leaked bounded-wait worker UAF — the genuine residual lifecycle risk on this same shutdown path (A7's `flush_work` guard).
- [[F20]] — GSP heartbeat / RPC timeout (Xid 119) — firmware actually wedged; F43's RPC completes.
