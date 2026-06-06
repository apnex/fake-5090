# F24 — Sanity-check noise — diagnostic `NV_ASSERT` lines reachable on lost-GPU paths

| Field | Value |
|---|---|
| **Sources** | C5 (v3 sanity-check class — non-load-bearing observability) |
| **Confidence** | hypothesis (observability degradation; no field incident attributed) |
| **Predecessor evidence** | `cascade-scope-audit.md` §"Non-load-bearing sites (C5 v3 sanity-check class)" — sites that emit `NV_PRINTF(LEVEL_ERROR, ...)` from cleanup paths reachable with lost-GPU status |
| **fake-5090 mechanism** | Surprise-removal A primitive + dmesg-noise-budget assertion (substrate-driven cascade; CI gate counts noise lines and enforces a budget) |

## Symptom (driver view)

A class of `NV_ASSERT_FAILED` and `NV_PRINTF(LEVEL_ERROR, ...)` log lines from the cleanup chain that ARE technically reachable on the lost-GPU path but are NOT load-bearing for cascade correctness. They are diagnostic-print sanity checks — the code paths still produce correct behaviour after the print fires (status propagates correctly, locks unwind correctly, no follow-on damage), but the prints themselves are noise in operator triage.

Examples per the cascade-scope-audit §"Non-load-bearing sites" classification:

- `NV_ASSERT(status != NV_ERR_BUSY)` in cleanup paths where the status WILL be `NV_ERR_GPU_IS_LOST` instead — assertion passes (LOST != BUSY) but adjacent `NV_PRINTF` lines fire with confusing messages like "expected to free %d resources but found %d" when the difference is "GPU is gone."
- Resserv refcount diagnostics that print "unexpected refcount %d during teardown" when the path is the lost-GPU shutdown rather than steady-state free.
- UVM-shim diagnostic prints that fire on first-touch-after-lost rather than only on genuine UVM-internal inconsistency.

Each individual line is harmless. The aggregate effect under a cascade is multi-thousand-line dmesg explosions that obscure the load-bearing signals (Xid 79, A3's recovery-attempt logs, A4's persistent-kill marker, sink-primitive canonical logs). Operators triaging the cascade spend significant time filtering noise to find the real diagnostic timeline.

This is distinct from F15 (load-bearing bare `NV_ASSERT` sites that actually cascade — failing those assertions causes downstream damage) and F25 (the dump-RPC self-DoS where the diagnostic action itself is the failure). F24 is "the prints are correct in steady-state but reachable on the lost path; their reachability is intentional diagnostic value EXCEPT during a cascade."

## Trigger sequence

1. Daemon brings device to a healthy bound state.
2. Test fires surprise-removal A.
3. Driver cascade runs (load-bearing assertions handled by F15/F16/F17/F18/F20 — these MUST NOT fire post-patch).
4. As the teardown chain unwinds, the sanity-check class of `NV_PRINTF(LEVEL_ERROR, ...)` lines fire from sites the cascade-scope-audit classified as non-load-bearing.
5. Count the new error-class lines in `dmesg` attributable to the surprise-removal window.

## Expected post-patch behaviour

C5 v3+v4 makes a **principled non-fix** decision for this class: the prints are kept because their diagnostic value in steady-state is non-zero, but their reachability under the cascade is gated by:

1. **Sink-state suppression.** A canonical idiom: `if (osIsGpuBusDead(pGpu)) return early-without-print;` at the top of the most-noisy diagnostic sites. Sites where the steady-state diagnostic value is high and the post-sink reachability is the only noise source get this treatment.
2. **Log-level demotion.** Sites where the print might genuinely add value during cascade triage (rare) get demoted from `LEVEL_ERROR` to `LEVEL_INFO`/`LEVEL_NOTICE` so they don't trip dmesg-rate-limit and don't trip operator-tool ERROR-level alerting.
3. **Budget enforcement.** A CI assertion that a surprise-removal A scenario produces ≤ N total `NV_PRINTF(LEVEL_ERROR, ...)` lines attributable to the cascade window, where N is small (target: ≤ 10 per cascade event).

The cascade-scope-audit identifies specific sites that fall in the per-site fix class vs. the budget-only class. Per-site fixes are applied where the cost is low (one-line `osIsGpuBusDead` guard); budget-only enforcement covers the long tail where individual fixes don't carry their weight.

## Assertion shape

**Dmesg-noise budget (CI gate):**

- After surprise-removal A in the substrate, count `dmesg` lines matching `NVRM:.*LEVEL_ERROR|NV_PRINTF.*LEVEL_ERROR|NV_ASSERT_FAILED` attributable to the cascade window.
- Total MUST be ≤ N (configurable budget; v1 target: N = 15 — covers load-bearing detector logs + canonical sink log + 1-2 unavoidable diagnostic lines per teardown phase).
- The budget MUST be ratcheted: each release sets a tighter budget if the previous N had headroom.

**Per-site suppression verification (specific high-frequency sites identified by audit):**

- The cascade-scope-audit names specific noisy sites (TBD in the audit document's per-site noise budget table). For each named site, the `osIsGpuBusDead` early-return idiom is applied. CI source-audit confirms each site has the guard.
- Post-patch dmesg count for each named site under surprise-removal A MUST be 0.

**Canonical-signal preservation:**

- The load-bearing signals MUST still appear:
  - The sink-primitive canonical log MUST fire exactly once.
  - A3's recovery-attempt logs (if A3 triggers) MUST fire per H1's cap.
  - Xid 79 / Xid 119 (whichever applies) MUST fire per the detector class that triggered.
- The budget assertion MUST exclude these canonical signals from the count.

**Operator-triage smoke test (informal):**

- After a cascade run, a triage operator scanning the dmesg output should see ≤ ~20 lines of nvidia-attributable signal, all of which carry diagnostic value. No "wall of noise" effect.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15/F16/F17/F18.
- **Dmesg-noise observability.** The substrate does not need anything new; the assertion is host-side log counting on the cascade-window dmesg snapshot. The cascade-scope-audit document is the source of truth for which sites are budget-class vs. per-site-fix class.
- **CI infrastructure.** The CI gate is a shell-level grep + line-count against `dmesg --time-format=iso` output captured around the surprise-removal A invocation. No new substrate state is required.

## Source citations

- `cascade-scope-audit.md` §"Non-load-bearing sites (C5 v3 sanity-check class)" and §"Per-site noise-budget table"
- `cascade-class-design-v4.md` §"Cleanup-path noise vs. cascade asserts — orthogonality"
- `C5-crash-safety.md` §"v3 amendment lineage — sanity-check class identification"
- F15 entry (load-bearing assertion sites — the orthogonal class)
- F25 entry (dump-RPC self-DoS — the active-damage diagnostic class)
