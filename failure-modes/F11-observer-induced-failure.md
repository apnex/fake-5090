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
