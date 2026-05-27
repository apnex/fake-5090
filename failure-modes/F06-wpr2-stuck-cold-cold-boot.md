# F6 — WPR2 stuck after `rm_init_adapter` failure (cold-cold-boot wedge)

| Field | Value |
|---|---|
| **Sources** | A3-recovery |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #1; Lever M-recover patches 0024–0028 |
| **fake-5090 mechanism** | BAR-MMIO state machine (WPR2 stuck-up) + surprise-removal B/C (root-port `pci_reset_bus`) |

## Symptom (driver view)

On a cold-cold-boot (eGPU enclosure power-cycled), the first `rm_init_adapter` call sometimes fails. As a side effect, the `WPR2` register (`NV_HUBMMU_PRI_MMU_WPR2_ADDR_HI`, BAR0 + `0x88a828`) is left in a "stuck-up" state. Subsequent retries of `rm_init_adapter` hit the GSP boot path and trip `_kgspBootGspRm: unexpected WPR2 already up` — the firmware refuses to proceed because it sees state from the prior failed attempt. The GPU enumerates on PCI but is functionally unusable; `nvidia-smi -L` returns "No devices found" or hangs.

Critically: the predecessor analysis initially blamed "WPR2 persists across boots" — falsified 2026-05-06 by diagnostic telemetry. The actual mechanism is **WPR2 set during a failed first `rm_init`, not across boots.** This distinction matters for F6's trigger sequence.

## Trigger sequence

1. Daemon brings device to healthy bound state.
2. Test arms: "fail the next `rm_init_adapter` path AND leave `WPR2` stuck-up." Mechanically: BAR0 `0x88a828` reads non-zero (e.g. `0x00FFFFFF`); the daemon-side `rm_init` simulation returns a failure code on the GSP boot register sequence.
3. Driver retries `rm_init_adapter` (the natural NVIDIA retry inside probe).
4. The retry sees WPR2 already up; trips the GSP firmware guard; second `rm_init` also fails.
5. **Unpatched path:** driver surrenders; `nvidia-smi -L` returns "No devices found."
6. **A3 path:** post-rmInit-FAIL trigger fires; recovery state machine engages; work handler calls `pci_reset_bus` on the upstream Thunderbolt bridge (NOT on the GPU pdev — TB topology requires bridge-level reset). After bridge reset, device re-emerges, WPR2 is clear, driver re-attempts rmInit successfully.

## Expected post-patch behaviour

**A3 trigger logic:**

- "Post-rmInit-FAIL with WPR2 stuck-up" is the matching condition that calls `tb_egpu_recover_trigger`.
- "WPR2 clear after rmInit failure" is a DIFFERENT failure mode (different root cause) and explicitly does NOT trigger recovery — let the upper layer report it. (This negative case is part of the F6 spec.)

**Re-entry guard:**

- The workqueue may schedule the trigger twice (race during fault detection). Re-entry guard ensures only one invocation drives the state machine.

**Work handler:**

- Reads pre-flight `PMC_BOOT_0` and WPR2 for telemetry.
- Calls `pci_reset_bus` on the upstream bridge (looked up via A1's topology walker).
- `pci_reset_bus` failure → surrender cleanly (no infinite retry within a single attempt; H1 cap governs the burst).

**Slot-reset callback:**

- After bridge reset, kernel dispatches `.slot_reset`. Handler reads `PMC_BOOT_0`:
  - Bus still down (`0xFFFFFFFF`) → return `DISCONNECT` + increment `surrender_count` + emit `PERMANENT_FAIL` uevent.
  - Healthy read → return `RECOVERED`; `.resume` fires.

## Assertion shape

**Trigger correctness:**

- After step 4 (two consecutive rmInit failures, WPR2 stuck), within bounded time:
  - `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_attempt_count` increments to 1.
  - `dmesg | grep -E 'tb_egpu recover: trigger fired.*reason=post-rminit-fail.*wpr2_up:YES'` matches exactly one line (or per the A3 intent doc's exact format).

**Negative case (WPR2 clear after rmInit fail):**

- Re-run with daemon serving WPR2 = 0 but rmInit still failing. `tb_egpu_recover_attempt_count` MUST NOT increment. A3 does not own this failure mode.

**Bridge-targeted reset (load-bearing topology assertion):**

- `dmesg | grep -E 'pcieport [0-9a-f:.]+: .*reset.*bus'` — the BDF in the matched line MUST equal the upstream TB bridge BDF, NOT the GPU's BDF. (If reset targets the GPU pdev directly, the topology walker is wrong and the test fails.)

**Recovery outcome:**

- On successful recovery: `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_state` → `READY`; `TB_EGPU_GPU_STATE=READY` uevent fired; subsequent `rm_init_adapter` succeeds (driver actually usable).
- On bus-still-down after reset: `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_surrender_count` increments; `TB_EGPU_GPU_STATE=PERMANENT_FAIL` uevent fired; `attempt_count` NOT reset (so H1 cap can engage on next burst).

**Re-entry guard:**

- Fire the trigger twice in rapid succession (within the same recovery window). `tb_egpu_recover_attempt_count` increments by exactly 1, not 2. (The second trigger sees the guard and bails.)

## fake-5090 mechanism mapping

- **Phase 2** BAR-MMIO state machine with named-register state (WPR2 at `0x88a828`).
- **Surprise-removal B or C primitive** for the bridge reset: vfio-user device must accept a `pci_reset_bus` from the kernel and re-emerge with WPR2 cleared. This is the topology-aware reset path (distinct from F3's silent dead-bus mechanism A).
- The daemon must model an upstream bridge (even minimally — just enough that `pci_reset_bus` can target it and the kernel sees a credible reset cycle).
- F6 is the most topology-demanding entry; it's the canonical fixture for "we've gotten the bridge modelling right."

## Source citations

- `A3-recovery.md` §"Trigger conditions" (post-rmInit-FAIL with WPR2 stuck-up, line ~60–90), §"Re-entry guard" (~95–120), §"Work handler — bridge-level reset" (~120–145), §"`.slot_reset` callback" (~145–185), §"Surrender path emits PERMANENT_FAIL uevent" (~145–180, ~218, ~274).
- aorus-5090-egpu `docs/failure-modes-index.md` row 1 (WPR2 cold-cold-boot wedge); patches 0024–0028 are the predecessor Lever M-recover stack that A3 consolidates.
- The 2026-05-06 diagnostic-telemetry archive (referenced in `_predecessor.md` line ~34) that falsified the "WPR2 persists across boots" hypothesis.
