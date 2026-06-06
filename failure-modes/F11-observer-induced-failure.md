# F11 — Observability tool triggers the failure it monitors

| Field | Value |
|---|---|
| **Sources** | A2-bus-loss-watchdog (sysfs surface), A4-close-path-telemetry (diagnostic markers) |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #11 (the nvidia-smi 17 s recovery cycle) |
| **fake-5090 mechanism** | *Not reproducible by design* — assert that A2's sysfs surface exists and is sufficient for passive observability |

## Symptom (driver view)

The original `aorus-egpu-observability-watchdog` polled `nvidia-smi -L` every 10 seconds for liveness. Each poll opened and closed `/dev/nvidia0`. The close-path destabilised the link (F12 territory); the Q-watchdog detected the resulting bus loss; Lever M-recover re-inited the GPU. Net effect: the monitoring tool was *creating* the failures it was supposed to detect, on a ~17-second cycle. The driver had no defence — from its perspective, real bus losses were happening; the fact that the tool was triggering them was an external invariant.

This is a class of bug that *cannot* be fixed inside the driver. The fix is to ensure observability tools have a *passive* surface (read-only sysfs files that don't touch `/dev/nvidia*`), and to document that operators must use that surface.

## Trigger sequence

F11 is not directly reproducible in `fake-5090` (we don't model `nvidia-smi`'s open/close behaviour against a real GPU, and reproducing it would require modelling the close-path destabilisation that is itself unpredictable on the real hardware). Instead, the fixture asserts the *prevention* surface exists and is sufficient.

1. Daemon brings device to healthy bound state.
2. Test enumerates the sysfs surface available to an observability tool.
3. Test confirms that a passive observability loop (sysfs reads only, NO `open("/dev/nvidia*")`) can determine all the state an observer needs:
   - Is the device enumerated? (`vendor`, `device`).
   - Is a driver bound? (`driver` symlink).
   - Is the device in a recovery episode? (`tb_egpu_qwd_state`, `tb_egpu_qwd_detections`).
   - Has recovery exhausted? (`tb_egpu_recover_state`, `tb_egpu_recover_surrenders`).
   - Last episode timestamp? (`tb_egpu_qwd_last_detection_jiffies`).

## Expected post-patch behaviour

A2 publishes five `tb_egpu_qwd_*` sysfs attributes specifically called out as the "predecessor PM #11 feedback loop" fix (extracts line 78). A3 publishes five recovery-state sysfs attributes. A4 emits dmesg markers but documents that A4 is *diagnostic, not mitigation* — the mitigation is the A2 sysfs surface plus an observer that uses only sysfs (extracts line 113 explicitly notes this).

So the F11 "fix" is structural:
- Mandatory: A2 publishes the sysfs surface (verified by F4/F5 already).
- Mandatory: A3 publishes the recovery sysfs surface (verified by F6/F10 already).
- Documentation: operators MUST use the sysfs surface for liveness polling, NOT `nvidia-smi -L`.

## Assertion shape

**Sysfs-surface completeness:**

- After probe, the following files MUST all exist and be readable as the unprivileged `nobody` user:
  - `/sys/bus/pci/devices/<BDF>/vendor`
  - `/sys/bus/pci/devices/<BDF>/device`
  - `/sys/bus/pci/devices/<BDF>/driver` (symlink)
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_qwd_state`
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_qwd_cycles`
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_qwd_detections`
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_qwd_last_detection_jiffies`
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_qwd_episode_count` *(or whatever the fifth attribute is called; check A2 intent doc §195–210)*
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_recover_attempt_count`
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_recover_state`
  - `/sys/bus/pci/devices/<BDF>/tb_egpu_recover_surrenders`
  - *(+ remaining A3 attributes per A3 intent doc lines ~312–345)*

**Passive-observer simulation:**

- Run a 60-second loop that reads ALL the above files every 100 ms. During the loop, the daemon does NOT inject any faults.
- Assert: no entries appear in `dmesg` for `dead-bus detected`, `recover`, or `[CLOSE]`. (Proves the sysfs reads themselves don't perturb device state.)
- Assert: the `tb_egpu_qwd_cycles` counter increments monotonically during the loop (proves the watchdog kthread is alive and the sysfs reads aren't blocking it).

**Anti-pattern documentation guard:**

- Run a 60-second loop that calls `nvidia-smi -L` every 10 s. Assert that this is documented as the anti-pattern in the operator runbook (look for a string in the repo's docs that explicitly warns against this). This is a *documentation* assertion — the only enforcement available.

## fake-5090 mechanism mapping

- **Not directly reproducible.** F11 documents the *fixture's role* in the larger ecosystem: by providing the sysfs surface, the driver prevents the observer pattern that caused PM #11.
- Treat F11 as a "presence-of-surface" test, not a "trigger-and-detect" test.

## Source citations

- `A2-bus-loss-watchdog.md` §"Sysfs surface and observer-feedback-loop prevention" (intent doc preamble + lines ~195–250); extracts line 78 ("predecessor PM #11 feedback loop").
- `A4-close-path-telemetry.md` §"A4 is diagnostic NOT mitigation; mitigation is A2 sysfs" (extracts line 113; intent doc preamble paragraph on PM #11 attribution).
- aorus-5090-egpu `docs/failure-modes-index.md` row 11 ("`nvidia-smi` triggers ~17 s recovery cycle") and the predecessor `_predecessor.md` §183–190 explaining the redesigned observability-watchdog.

## Update 2026-06-06 (A13 live-FAIL)

The #292 A13 live-FAIL (apnex.31, capture `netcon3-2026-06-06-A13-live-FAIL-rpcpoll-wedge.log`) adds a NEW, distinct observer-induced-failure datapoint: not the PM #11 open/close feedback loop, but a **synchronous-printk amplifier** that may *convert* a bounded stall into a hard wedge. Cross-ref **F10**, **F16**, and **F47** (the in-flight re-open GSP RPC-poll deadlock the storm rides on).

**The netcon3 datapoint.** During the diverged re-open, the GSP RPC poll `_kgspRpcRecvPoll` storms the dead bus emitting hundreds of `GSP RM / LibOS heartbeat timed out` + `_kgspIsHeartbeatTimedOut … timeout 5200` + `tmrGetTimeEx_GH100: Consistently Bad TimeLo value ffffffff` lines (~3 lines/iteration, every ~0.1 ms). The synchronous **netconsole printk-storm at `console_loglevel=8`** — saturating the storming CPU in the netpoll path — is a **candidate proximate CONVERTER** that turns an otherwise *bounded* per-RPC heartbeat-timeout into the observed host hard-wedge.

**Why the underlying stall is bounded (so the amplifier is suspect).** `_kgspIsHeartbeatTimedOut` computes elapsed time from the GPU **PTIMER** (`tmrGetTime`, kernel_gsp.c:2294), not wall-clock; on the dead bus `tmrGetTimeEx_GH100` returns 0 ("Consistently Bad TimeLo ffffffff") so `diff > 5200ms` is always TRUE — **but this boolean is ONLY LOGGED** (the `done:` block, kernel_gsp.c:2974-2981); it never breaks the loop. The per-call OSTIMER `gpuCheckTimeout` is re-armed every call (GSP-client default `GPU_TIMEOUT_FLAGS_OSTIMER`), so each `_kgspRpcRecvPoll` returns in ~0.3-3 ms and is re-invoked: the worker is **bounded per-call**, the outer re-dispatch effectively unbounded within the host's surviving lifetime. With a printk-storm absent, the worker might stall-but-survive; with the synchronous netconsole storm it does not.

**UNRESOLVED — does the host recover netconsole-OFF?** Both captures died early (netcon2 silent at +1.067 s; netcon3 storm EOF at +1.075 s, ~1.9 s before A12's +3000 ms foreground arm could fire), so no bound was ever testable. The grounding findings internally disagree: the post-`INIT_DONE` API-lock reacquisition (kernel_gsp.c:4818-4820) implies A12's `rm_cleanup` rescue **also** COND_ACQUIRE-defers (i.e. no recovery), which the source read favors — but the F44 substrate (foreground parked in `wait_for_completion_timeout` holding `nvl->ldata_lock` across `flush_work`, nv.c:2037) is a genuine wedge independent of any printk. So whether the storm is *load-bearing* or merely *amplifying* is **UNRESOLVED**.

**Mandate — dual-loglevel n≥3 validation.** Because the amplifier-vs-genuine-wedge question cannot be answered from the dead captures, the apnex.32 (C7 + A13' + A14) validation MUST run the primary repro at **BOTH** `console_loglevel=8` (netconsole armed) **AND** minimised observability (`echo 4 4 1 7 > /proc/sys/kernel/printk`, async `/dev/kmsg`, external 100 ms ping/ssh liveness probe, fsync'd heartbeat file, passive sysrq-w/t/l), n≥3 each. **SURVIVAL at BOTH loglevels** is the load-bearing proof that the storm is removed *at source* (C7-e3 pre-loop return suppressing the RPC poll re-entry + C7-e5 gating the heartbeat prints behind `!osIsGpuBusLost`), not merely hidden by lowering the loglevel. Passive channels only — consistent with this fixture's thesis and the standing "observability perturbs the eGPU bug" finding (IBT blocks kprobe on closed RM; kdump/drgn cannot capture the live wedge).

**Fixture role.** Like the PM #11 case, this is *not* directly reproducible in fake-5090 (we don't model netconsole netpoll CPU saturation). Treat it as a methodology assertion: any harness exercising the #292 re-open repro MUST capture at two loglevels and prove survival is loglevel-independent before claiming a fix. Source: `/root/nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/finding-2026-06-06-A13-live-FAIL-rpcpoll-wedge.md` and `.../design-2026-06-06-292-redesign-C7-A13prime-A14.md` (§1.6, §1.7, §5a, GAP-4).
