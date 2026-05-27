# F12 — `/dev/nvidia0` close-path wedge (Problem 2)

| Field | Value |
|---|---|
| **Sources** | A4-close-path-telemetry |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu PM #3 (Problem 2 — second open hung host); patch 0029 |
| **fake-5090 mechanism** | BAR-MMIO state (LAST-CLOSE hardware snapshot — WPR2 captured on close) |

## Symptom (driver view)

The original PM #3 / Problem 2: on a `/dev/nvidia0` close that brought the usage count to zero (LAST-CLOSE), the close-path interacted badly with something on the bus. The *next* `open()` of `/dev/nvidia0` would hang the host. Empirically this does not reproduce on the current build (H22 in the predecessor ledger). A4 instruments the close path *prophylactically* so that if Problem 2 ever re-emerges, the smoking-gun signature is captured at the moment of fault, not reconstructed after the fact.

The driver's perceived problem now: we don't know *what state the GPU was in at the last close before the next bad open*. A4 makes that state observable in `dmesg` for every close, with explicit `(LAST-CLOSE)` annotation on the transition.

## Trigger sequence

F12 is fundamentally a *telemetry-presence* test, not a trigger-and-detect test (the underlying bug is empirically gone). But the telemetry itself has assertions:

1. Daemon brings device to healthy bound state.
2. User-space test opens `/dev/nvidia0` (usage count 0 → 1). A4 emits a close-site marker on entry-bookkeeping (per A4 intent doc).
3. Open a second FD (usage count 1 → 2). A4 emits an open marker.
4. Close the second FD (usage count 2 → 1). A4 emits a close marker with `usage_count=1` (NOT LAST-CLOSE — there's still one FD open).
5. Close the first FD (usage count 1 → 0). A4 emits a close marker with `usage_count=1 (LAST-CLOSE)` AND a hardware snapshot line capturing `PMC_BOOT_0` and `WPR2` state.
6. Optionally: trigger the shutdown path. A4 emits a `post-shutdown` marker with the hardware snapshot (independent of LAST-CLOSE).

The smoking-gun shape (if Problem 2 re-emerges): step 5's snapshot line shows `wpr2_up:YES`. The exact handle that links F12 to F6.

## Expected post-patch behaviour

A4 (the RM-side, distinct from UVM-side which is F13) emits, for every close-path site, the marker:

```
tb_egpu [CLOSE]: site=<site>     usage_count=<N>[<sp>(LAST-CLOSE)]
```

(Verbatim format from A4 intent doc line 62; `<site>` is one of `close-entry`, `close-exit`, `post-shutdown` per the intent doc's site catalogue.)

On LAST-CLOSE (and on `post-shutdown` always), an additional hardware-snapshot line:

```
tb_egpu [CLOSE]: site=<site>     pdev=<BDF> bar0=0x<bar0> PMC_BOOT_0=<prefix><hex> WPR2=<prefix><hex> wpr2_up:<YES|no>
```

(Format from A4 intent doc line 136. `<prefix>` is `0x` on success or `MAPFAIL:` if the snapshot read failed — the snapshot line itself MUST be emitted even on `ioremap` failure, with the `MAPFAIL:` sentinel.)

Defensive paths:
- `nvl == NULL`: emit `tb_egpu [CLOSE]: site=<site> pdev=NULL — cannot read` (A4 intent doc line 367; clean no-op).
- `bar0 == 0`: emit `tb_egpu [CLOSE]: site=<site> bar0=0 — skipping` (line 368).
- `ioremap` failure on PMC_BOOT_0: emit snapshot with `PMC_BOOT_0=MAPFAIL:deadbeef` (line 180); do NOT crash.

## Assertion shape

**Marker emission completeness:**

For trigger steps 2–5:

- After step 2 (first open): `dmesg | grep -cE 'tb_egpu \[CLOSE\]: site=open-entry.*usage_count=1'` ≥ 1 *(if A4 instruments opens; check A4 intent doc — if not, skip)*.
- After step 4 (non-last close): `dmesg | grep -cE 'tb_egpu \[CLOSE\]: site=close-entry\s+usage_count=1$'` ≥ 1 (no LAST-CLOSE annotation).
- After step 5 (LAST-CLOSE):
  - `dmesg | grep -cE 'tb_egpu \[CLOSE\]: site=close-entry\s+usage_count=1 \(LAST-CLOSE\)'` == 1.
  - `dmesg | grep -cE 'tb_egpu \[CLOSE\]: site=close-entry\s+pdev=<BDF> bar0=0x[0-9a-f]+ PMC_BOOT_0=0x[0-9a-f]{8} WPR2=0x[0-9a-f]{8} wpr2_up:(YES|no)'` == 1.

**Hardware-snapshot shape:**

- The snapshot line MUST contain BOTH `PMC_BOOT_0=` and `WPR2=` fields.
- Each field MUST be prefixed by either `0x` (successful read) or `MAPFAIL:` (read failure) — never bare hex.
- `wpr2_up:` value MUST be `YES` or `no` (exact strings per line 366), never `1`/`0` or `true`/`false`.

**Defensive paths:**

- Test variant with daemon advertising `bar0 = 0` for the device: assert `bar0=0 — skipping` line emitted instead of snapshot.
- Test variant with kernel-side `ioremap` failure injection (test infra, not daemon): assert `PMC_BOOT_0=MAPFAIL:deadbeef` (or equivalent fixed sentinel) line emitted; driver does NOT crash.

**Post-shutdown coverage:**

- Trigger the driver's shutdown path (module unload, or kernel shutdown notifier). Assert `tb_egpu [CLOSE]: site=post-shutdown ...` line emitted with hardware snapshot — independent of whether there was a LAST-CLOSE at the same time (line 158).

**Smoking-gun shape (if Problem 2 ever re-emerges):**

- Documented but not asserted in routine test runs: a LAST-CLOSE snapshot with `wpr2_up:YES` is the canonical Problem 2 signature. Couples to F6 recovery trigger (extracts line 108).

## fake-5090 mechanism mapping

- **Phase 1 + small extension.** Requires BAR0 state machine that can be set to specific values (so the snapshot reads return predictable test values) and the daemon's MMIO callback for `0x88a828` (WPR2_ADDR_HI) returns the scripted state.
- The `ioremap` failure variant is *kernel-side* test infrastructure, not daemon work — defer to host-side test harness.
- F12 is a low-effort fixture once Phase 1 substrate exists; the assertions are mostly `grep` on `dmesg`.

## Source citations

- `A4-close-path-telemetry.md` §"Requirement: [CLOSE] marker at every RM-side site" (intent doc line 40), marker format string (line 62), snapshot format string (line 136), `MAPFAIL` rule (line 138, 147, 180), defensive `pdev=NULL` (line 367), defensive `bar0=0` (line 368), `post-shutdown` always-fires rule (line 158), logging table (lines ~364–371).
- aorus-5090-egpu `docs/failure-modes-index.md` row 3 (Problem 2); patch 0029.
- Empirical "does not reproduce on current build" attribution: H22 in aorus-5090-egpu reliability ledger.
