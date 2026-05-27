# F13 — UVM close-path wedge (Problem 4 — misattributed; benign)

| Field | Value |
|---|---|
| **Sources** | A4-close-path-telemetry (UVM-side instrumentation only) |
| **Confidence** | doc-only (the underlying "bug" was misattributed; A4's UVM markers are prophylactic) |
| **Predecessor evidence** | aorus-5090-egpu PM #4 (admitted misattribution) |
| **fake-5090 mechanism** | *Not reproducible* — assert that A4 UVM marker emission is correct |

## Symptom (driver view)

Predecessor PM #4 ("Problem 4") originally claimed that the *last* close of `/dev/nvidia-uvm` by a CUDA process exit caused future opens to hang the host. The pattern was inferred from Problem 2 (F12) and assumed to share the same mechanism. Subsequent investigation found:

- UVM's `va_space_destroy` is *internal-state-only* on the current build.
- It does not touch the bus in any way that could destabilise the link.
- The empirical "next open hangs" pattern attributed to UVM never actually reproduced under controlled conditions — the reports were either confused-with-Problem-2 events or transient state from concurrent RM-side activity.

So PM #4 is **classified resolved as a misattribution**. A4 still instruments the UVM close path with markers — not because there's a known bug to catch, but because:
1. If a *real* UVM close-path bug ever emerges, the instrumentation is already in place to characterise it.
2. The UVM-side `[CLOSE]` markers + UVM-local fd-count tracking give post-mortem analysts a cleaner timeline than dmesg without them.

## Trigger sequence

Like F11, F13 is a telemetry-presence test, not a trigger-and-detect test. The "trigger" is normal UVM lifecycle activity from a user-space client.

1. Daemon brings device to healthy bound state; driver loaded with UVM module.
2. Test opens `/dev/nvidia-uvm` from a user-space helper (UVM fd_count 0 → 1). A4 emits a UVM open marker.
3. Open a second UVM fd (1 → 2). A4 emits another marker.
4. Close the second fd (2 → 1). A4 emits a UVM close marker.
5. Close the first fd (1 → 0). A4 emits a UVM close marker with `(LAST-CLOSE)` annotation AND triggers a hardware snapshot if an NVIDIA pdev is bound (via the cross-module pdev lookup A4 exposes — extracts line 112).
6. Repeat with no NVIDIA pdev bound (e.g. before the RM-side device is probed): assert the "no NVIDIA pdev bound; skipping snapshot" path fires cleanly.

## Expected post-patch behaviour

A4's UVM-side (distinct from RM-side / F12) emits the marker:

```
tb_egpu UVM [CLOSE]: site=<site>     fd_count=<N>[<sp>(LAST-CLOSE)]
```

(Verbatim from A4 intent doc line 215.) Sites include `uvm-open-entry`, `uvm-release-entry`, `uvm-release-exit`, plus the lifecycle hooks per the intent doc's UVM site catalogue (line ~233).

UVM-local fd_count is tracked separately from the RM-side `/dev/nvidia0` usage count (UVM has its own char device, its own opens).

On UVM open that follows a LAST-CLOSE (the "first open after last close" transition — line 233), A4 triggers a hardware snapshot via the cross-module pdev lookup. If no NVIDIA pdev is currently bound (lifetime ordering edge case), emit:

```
tb_egpu UVM [CLOSE]: site=<site> — no NVIDIA pdev bound; skipping snapshot
```

(Verbatim from line 255.) This is the *clean no-op* path, not an error.

On UVM release that brings fd_count to zero, snapshot fires (captures-on-resolution telemetry — extracts line 111).

## Assertion shape

**Marker emission completeness:**

For trigger steps 2–5:

- After step 2 (first UVM open): `dmesg | grep -cE 'tb_egpu UVM \[CLOSE\]: site=uvm-open-entry\s+fd_count=1.*'` ≥ 1.
- Step 3 (second open): fd_count=2 marker.
- Step 4 (non-last release): fd_count=1, no LAST-CLOSE annotation.
- Step 5 (LAST-CLOSE):
  - `dmesg | grep -cE 'tb_egpu UVM \[CLOSE\]: site=uvm-release-entry\s+fd_count=1 \(LAST-CLOSE\)'` == 1.
  - If NVIDIA pdev bound: hardware snapshot line emitted (same format as F12's, since the snapshot helper is shared via the cross-module exposure).

**Lifecycle-ordering edge case:**

- Test variant: open UVM *before* the RM-side device is probed (this is unusual but legal — UVM has its own enumeration).
- Assert: `dmesg | grep -cE 'tb_egpu UVM \[CLOSE\]: site=.* — no NVIDIA pdev bound; skipping snapshot'` ≥ 1 on the trigger.
- Assert: driver does NOT crash.

**Cross-module coupling minimality:**

- The pdev-lookup helper exposed for UVM use MUST be a single function call, not a deep coupling. Static-analysis assertion (or code review checklist item): UVM source file references exactly one RM-side symbol for the snapshot helper, no others.

**UVM-vs-RM site name distinctness:**

- All UVM marker lines start with `tb_egpu UVM [CLOSE]:` (note `UVM` prefix, line 215).
- All RM marker lines start with `tb_egpu [CLOSE]:` (no `UVM` prefix, line 62).
- This is the disambiguation that lets `grep` cleanly separate the two streams (intent doc preamble explicitly notes this).

## fake-5090 mechanism mapping

- **Not reproducible in the substrate.** The UVM char device is purely a kernel/user-space concern; vfio-user doesn't model it. The fake-5090 device only needs to be *present* (so the cross-module pdev lookup succeeds) for the snapshot path to fire on LAST-CLOSE.
- F13's value is in the test fixture asserting the telemetry shape, not in driving the underlying interaction.

## Source citations

- `A4-close-path-telemetry.md` §"Requirement: UVM-side [CLOSE] marker and fd-count tracking" (intent doc line 188), UVM marker format string (line 215), `uvm-open-entry` first-open-after-LAST-CLOSE example (line 233), `no NVIDIA pdev bound` defensive path (line 255), cross-module pdev exposure rationale (extracts line 112), `pr_info` log-level for UVM lines (line 369), site-name disambiguation rationale (line 453).
- aorus-5090-egpu `docs/failure-modes-index.md` row 4 (Problem 4 — RESOLVED as misattribution).
- Extracts JSON line 103: "UVM close-path bug was MISATTRIBUTED — pattern-matched from Problem 2; UVM va_space_destroy is internal-state-only; benign."
