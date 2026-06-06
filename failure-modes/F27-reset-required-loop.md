# F27 — `Reset Required` loop — `pci_reset_function` returns success, GPU re-enters lost state

| Field | Value |
|---|---|
| **Sources** | A3 (recovery — explicitly covers via H1 cap + H3 persistent-kill); C5 (sink-state primitive must catch the immediate re-loss) |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #8 (recovery storm investigation); A3 design's identification of "reset succeeds at the bus level but the device returns to lost state before driver re-init completes" as a distinct trigger |
| **fake-5090 mechanism** | Reset-behaviour configuration: substrate accepts `pci_reset_function`-equivalent invocations and returns "reset successful" at the bus level, but the post-reset MMIO read returns the dead-bus sentinel within a configurable delay (default: immediately, modelling a chip that resets cleanly but then re-fails) |

## Symptom (driver view)

When the driver detects bus loss and A3's recovery layer fires, A3 issues a `pci_reset_function` (or equivalent reset primitive) against the device. The kernel's PCI framework reports the reset as successful — config-space is re-established, BARs are re-mapped, the device responds to enumeration. A3 then re-attempts `rm_init_adapter`.

`rm_init_adapter`'s first MMIO read returns `0xFFFFFFFF` (the device is back in bus-loss state — either it never genuinely recovered, or it recovered briefly and immediately re-failed). The sink primitive's funnel-1 fires; `rm_init_adapter` returns failure; A3 sees the failure; A3's H1 cap counts this as one of its 3 attempts; A3 retries; the loop repeats up to 3 times.

This is a **distinct failure mode from F23** because F23 is the cold-boot case (chip broken before first usable state); F27 is the steady-state-then-cascade case (chip was usable, then failed, then reset succeeded mechanically but the chip remains broken). The signature in `dmesg` is different:
- F23: instant Xid 79 at boot, no preceding healthy operation.
- F27: workload runs successfully for some time, then Xid 79, then A3 reset cycle, then 2-3 more Xid 79 fires within the H1 window, then persistent-kill.

A3's H1 cap is the load-bearing mitigation: without it, the loop runs unbounded (this is the F10 recovery-storm class). With H1: bounded to 3 attempts. With H3 persistent-kill: subsequent re-probe events (from udev, nvidia-persistenced, operator scripts) do not re-trigger the cycle.

F27 specifically requires that the sink primitive correctly identifies "reset succeeded mechanically but the GPU is still lost" — i.e. the funnel-1 fire on the first post-reset MMIO read MUST set sink-state, propagate up, signal A3's recovery layer that the attempt failed, and let A3 decide whether to retry or persist-kill.

## Trigger sequence

1. Daemon brings device to a healthy bound state.
2. Test fires surprise-removal A. Cascade detector fires; sink-state set; A3's recovery layer triggers.
3. A3 issues `pci_reset_function` against the device. Substrate accepts the reset request and returns "reset successful" at the bus level (config-space is re-established, BARs are valid, enumeration succeeds).
4. A3 re-attempts `rm_init_adapter`. First MMIO read goes to the substrate.
5. Substrate is in "post-reset-still-dead" mode (configurable): returns the dead-bus sentinel for MMIO reads despite the reset having succeeded mechanically.
6. Funnel-1 fires; `rm_init_adapter` returns failure with `NV_ERR_GPU_IS_LOST`.
7. A3 sees the failure; increments its retry counter; retries up to H1 cap (3); each retry follows steps 3-6.
8. After 3 failures, A3 marks the device persistent-killed (H3); subsequent re-probe events are suppressed.

## Expected post-patch behaviour

The sink primitive correctly catches the "reset succeeded but device still dead" case at the first post-reset MMIO read. A3's bounded retry cap holds (≤ 3 attempts). A3's persistent-kill marker fires after the 3rd failed attempt. The host converges to a clean "this GPU is dead" state within bounded time (target: ≤ 30 s — 3 × A3-retry-window + cleanup).

No retry-storm; no Xid 79 flood; A4's telemetry surfaces the persistent-kill state via sysfs for operator tooling.

## Assertion shape

**Bounded retry verification:**

- After substrate enters "post-reset-still-dead" mode and surprise-removal A fires, count A3 retry-attempt logs:
- `dmesg | grep -cE 'A3.*recovery.*attempt'` (or A3's canonical retry-attempt log) MUST be ≤ 3.
- `dmesg | grep -cE 'NVRM: Xid.*79'` SHOULD be ≤ (H1 cap + 1) = 4 (initial fire + one per retry).

**Persistent-kill marker fires:**

- `dmesg | grep -cE 'A3.*persistent.*kill'` MUST be ≥ 1 after the 3rd failed attempt.
- A4's persistent-kill sysfs path (TBD by A4 intent-doc) MUST reflect persistent state within bounded time after the marker is set.

**No re-probe re-trigger:**

- Simulate a udev-style re-probe by triggering a synthetic `pci_rescan` (test harness writes to `/sys/bus/pci/rescan` or equivalent). A3's persistent-kill MUST suppress re-entry to `rm_init_adapter`.
- `dmesg | grep -cE 'A3.*recovery.*attempt'` after the synthetic re-probe MUST NOT increase.

**Sink-state on first post-reset read:**

- Substrate-side: the first MMIO read after the reset-accept response MUST return `0xFFFFFFFF`. Verifiable via substrate's MMIO trace export.
- Host-side: funnel-1's canonical detection log MUST fire within bounded time after the substrate returns the sentinel. The detection log MUST attribute the source to "post-reset MMIO read."

**Bounded total time:**

- Total wall-clock from initial cascade trigger to host settled in persistent-kill state MUST be ≤ 30 s.

**Negative — F27 fix MUST NOT prevent genuine post-reset recovery:**

- Run the same scenario but configure the substrate to genuinely recover after reset (substrate returns the dead-bus sentinel for the FIRST post-reset read, then transitions back to healthy on the SECOND read). A3's first retry SHOULD see the failure (one Xid 79); A3's second retry SHOULD succeed (`rm_init_adapter` completes). The H1 cap MUST NOT trigger persistent-kill in this case.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25/F26.
- **Reset-behaviour configuration.** Substrate exposes a reset-callback that accepts `pci_reset_function`-equivalent invocations and reports success/failure per configuration. Multi-state machine:
  - State R0: pre-reset (substrate has accepted surprise-removal A; MMIO returns sentinel; awaiting reset).
  - State R1: reset accepted (substrate has re-established BARs, accepts enumeration; configurable: next MMIO returns sentinel OR returns plausible value).
  - State R2: post-reset-recovered (substrate is back to healthy operation; MMIO returns plausible values).
  - State R3: post-reset-still-dead (substrate has accepted the reset mechanically but stays in MMIO-returns-sentinel mode — modelling a chip that resets at the bus level but is genuinely broken).
- The test harness configures whether the substrate, after accepting a reset, transitions to R2 (genuine recovery) or R3 (mechanically-reset-but-still-dead). F27's primary scenario is R3; the negative assertion uses R2 to demonstrate the fix doesn't prevent genuine recovery.
- **A3's retry-counter observability.** A3 exposes its retry-counter and persistent-kill state via sysfs (per A4's design); the assertion shape reads these for verification.

## Source citations

- aorus-5090-egpu PM #8 (recovery storm investigation; documented the unbounded-retry root cause)
- A3-recovery intent doc §"H1 cap (3 attempts max)" and §"H3 persistent-kill marker"
- `cascade-class-design-v4.md` §"DETECTION INPUTS" §"Post-reset MMIO funnel re-fire"
- F10 entry (recovery storm — F27's unbounded sibling, demonstrates what A3 v0 looked like before H1/H3)
- F23 entry (instant Xid 79 cold-boot — contrasting failure-time-vs-cascade-time)
- A4-close-path-telemetry intent doc — persistent-kill sysfs surface
