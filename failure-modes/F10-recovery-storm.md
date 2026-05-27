# F10 — Recovery storm (21 attempts in 9 min, no cap / rate-limit / persistent kill)

| Field | Value |
|---|---|
| **Sources** | A3-recovery |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #8 (2026-05-06 incident); Lever M-recover patches 0024 + 0026 + 0027 + 0028 (H1/H2/H3/H4 hardening) |
| **fake-5090 mechanism** | AER sustained fault injection (exercise H1 cap, H2 rate-limit, H3 persistence) |

## Symptom (driver view)

The 2026-05-06 incident: the *first* Lever M-recover Commit 3 attempted recovery 21 times in ~9 minutes, driving the GPU into a worse state than the original fault. Root causes (per the predecessor's post-mortem):

- **No cap (H1):** no upper bound on retry attempts per fault burst.
- **No rate-limit (H2):** no minimum interval between attempts; back-to-back retries hammered the bridge.
- **No persistent kill-switch (H3):** `echo 0 > /sys/.../Enable` was reset on every `modprobe -r` round-trip, so operator attempts to disable recovery were silently undone.
- **Naive `.error_detected` (H4):** the handler returned `DISCONNECT` *while its own recovery was in flight*, conflicting with the in-progress reset and causing the kernel to escalate further.

## Trigger sequence

A3 must defend against all four root causes. The fixture stresses each axis:

1. Daemon brings device to healthy bound state; A3 is active with `NVreg_TbEgpuRecoverMaxAttempts = N` (default per A3 intent doc, e.g. 3).
2. Test arms sustained fault: AER fatal injection that re-fires on every recovery attempt (the daemon's `slot_reset` simulation leaves the device in the same fault state instead of clearing it).
3. Drive the recovery state machine to fire repeatedly. Count attempts in `tb_egpu_recover_attempt_count`.
4. **H1 sweep:** verify cap exhaustion at attempt `N`.
5. **H2 sweep:** within a single burst, measure inter-attempt interval; verify ≥ configured minimum.
6. **H3 sweep:** persistent kill — write `0` to the enable sysfs, unload and reload the driver module, verify recovery stays disabled across the cycle.
7. **H4 sweep:** during an in-flight recovery, inject an additional AER fatal; verify the second `.error_detected` does NOT return `DISCONNECT` (which would cause a kernel-side escalation conflict).
8. **Idle-burst-boundary sweep:** let an idle period elapse after a burst ends; trigger a new genuine event; verify `attempt_count` was reset so the new event isn't gated by ancient history.

## Expected post-patch behaviour

**H1 (MaxAttempts cap):**
- `tb_egpu_recover_attempt_count` increments to `NVreg_TbEgpuRecoverMaxAttempts`; the next trigger is gated (no actual recovery action) and emits a `PERMANENT_FAIL` uevent.

**H2 (per-device rate limit):**
- Triggers within `last_fire_jiffies + min_interval` are deferred (logged as "gated; rate-limited"). `attempt_count` NOT incremented (rate-limit gate is *before* the counter).

**H3 (persistent kill-switch):**
- A kill-switch file (intent doc names the path; typically `/etc/aorus/recovery-kill-switch` or similar) overrides the cmdline `Enable=1` flag. Survives `modprobe -r` round-trip. Once present, ALL triggers gate out with `disable engaged`; sysfs reads return `0\n` per scenario at line ~326 in A3 intent doc.

**H4 (smarter error_detected during recovery):**
- While own recovery is in flight (state is `RECOVERING`), `.error_detected` returns `CAN_RECOVER` or `NEED_RESET` consistent with the in-flight recovery — NOT `DISCONNECT`. Avoids the 2026-05-06 conflict.

**Idle-burst-boundary reset:**
- After an idle period (no triggers for some configurable duration), the *next* trigger checks `last_fire_jiffies` and resets `attempt_count` to 0 before doing anything else. So a genuine new event isn't gated by ancient history.

**Recovery-success counter reset:**
- `attempt_count` is reset to 0 ONLY on verified end-to-end recovery (`rmInit` success after the storm) — NOT on cold-boot rmInit success (which is a no-op reset).

## Assertion shape

**H1 cap:**

- After `N` attempts in step 4: `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_attempt_count` returns `N`. The `(N+1)`-th trigger does NOT increment (still `N`).
- `cat /sys/bus/pci/devices/<BDF>/tb_egpu_recover_surrenders` (per A3 intent doc line 35) increments by 1.
- `dmesg | grep -E 'tb_egpu recover: trigger gated.*max-attempts.*emitting PERMANENT_FAIL'` matches one line per A3's logging table (line ~480).
- `TB_EGPU_GPU_STATE=PERMANENT_FAIL` uevent fired (capture with `udevadm monitor --kernel`).

**H2 rate-limit:**

- In step 5: inter-attempt interval as measured by daemon-side timestamp deltas ≥ `min_interval`. Triggers within the deferral window log `rate-limited` and do NOT increment `attempt_count`.

**H3 persistent kill-switch:**

- Write `0` to the recovery-enable sysfs (path per A3 intent doc / A5 toggle doc). `rmmod nvidia && modprobe nvidia`. Read enable sysfs back: returns `0`. Trigger an event: NO recovery fires; `tb_egpu_recover_attempt_count` does NOT increment; `dmesg` shows the trigger was gated with `disable engaged`.
- Reading the five sysfs counters (per scenario at A3 intent doc line ~326) returns `0\n` placeholders while kill-switch engaged (proves the sysfs surface honours the kill-switch state).

**H4 smarter error_detected:**

- During in-flight recovery (state `RECOVERING`), inject a second AER fatal. Capture the `.error_detected` callback's return value. MUST NOT be `pci_channel_io_perm_failure` (`DISCONNECT`). Log line should reflect "in-flight recovery; deferring" or equivalent.

**Idle-burst-boundary reset:**

- After H1 exhausts at attempt `N`, wait the idle-burst threshold. Trigger a new event. `tb_egpu_recover_attempt_count` resets to `0` then increments to `1` (per A3 scenario "Gate reset on idle burst boundary clears attempt_count", line ~247).

**Recovery-success reset:**

- Run a successful recovery cycle from a non-storm fault. After `tb_egpu_recover_state == READY`, verify `attempt_count` reset to `0`. (Verified end-to-end recovery resets; cold-boot success does not.)

**Sysfs `force_trigger`:**

- `echo 1 > /sys/bus/pci/devices/<BDF>/tb_egpu_recover_force_trigger` invokes the trigger function as if from post-rmInit-FAIL (per A3 intent doc line 345). Useful as a test infrastructure shortcut so the test can drive the state machine without needing real AER injection for every H-class assertion.

## fake-5090 mechanism mapping

- **Phase 2 AER injection** with sustained-fault capability: the daemon must be able to re-arm the fault automatically after each recovery attempt, so the storm reliably produces the storm shape.
- **Bridge model** that accepts `pci_reset_bus` and either clears the fault (one-shot test) or leaves it set (storm test) per daemon scenario.
- The H3 sweep needs cooperation between fake-5090 and the test harness on the host side (driver module load/unload around the kill-switch write); the daemon's role is just to keep the device present across the module cycle.
- F10 is the integration test for the whole A3 stack — passing F10 means H1, H2, H3, H4, and the counter-reset logic are all wired correctly.

## Source citations

- `A3-recovery.md` §"H1 / H2 / H3 / H4 pre-schedule gates" (~91–95 in extracts JSON; intent doc §"Recovery state machine gates"), §"H1 cap exhaustion emits PERMANENT_FAIL uevent" (extracts line 94, intent doc §267), §"H2 rate-limit defers a too-quick re-trigger" (intent doc line ~260), §"attempt_count reset on idle-burst boundary" (extracts line 93, intent doc §247), §"H3 kill-switch persistence" (extracts line 96, intent doc), §"H4 smarter error_detected" (extracts line 92), §"attempt_count reset only on verified end-to-end recovery" (extracts line 95), §"sysfs force_trigger" (intent doc line ~345), §"recovery counter sysfs attributes" (intent doc lines ~312–345), logging table (intent doc lines ~480–510).
- aorus-5090-egpu `docs/failure-modes-index.md` row 8 (Recovery storm 2026-05-06); `archive/commit3-recovery-loop-2026-05-06-161429/` for the original telemetry capture.
- aorus-5090-egpu reliability ledger entry H15 (RESOLVED — the umbrella for H1/H2/H3/H4).
