# F17 — kfsp arithmetic invariant on dead-bus reads — funnel sentinel cannot satisfy

| Field | Value |
|---|---|
| **Sources** | C5 (v4 site-local fix G9) |
| **Confidence** | field-bug |
| **Predecessor evidence** | MISSION-1 E07 Run 3 forensic record (2026-05-26) — site #12 discovered during forensics, NOT in the v3 sweep |
| **fake-5090 mechanism** | Surprise-removal A primitive + FSP EMEM register backend (substrate exposes `PFSP_EMEMC` register class; reads return `0xFFFFFFFF` when sink is active) |

## Symptom (driver view)

`_kfspWriteToEmem_GH100` (`kern_fsp_gh100.c:649`) asserts an arithmetic invariant over two register reads:

```c
NV_ASSERT(
    (DRF_VAL(_PFSP, _EMEMC, _OFFS, ememOffsetEnd) -
     DRF_VAL(_PFSP, _EMEMC, _OFFS, ememOffsetStart)) == wordsWritten
);
```

On a dead bus, BOTH reads return `0xFFFFFFFF`. The `DRF_VAL` bitfield extraction extracts the `_OFFS` bitfield from each value; both extractions produce the same result (whatever bit-range the field spans of all-1s). The arithmetic difference is therefore 0; `wordsWritten` is non-zero (the caller wrote N words to EMEM); `0 != N`; the invariant fails.

This is a **NEW pattern class** distinct from every status-check site C5 v3 swept for. The class is: "an arithmetic invariant computed over multiple dead-bus reads, where the funnel-1 sentinel (`0xFFFFFFFF`) cannot satisfy the invariant." The C5 v3 status-pattern sweep regex looked for `NV_ASSERT.*status.*==.*NV_OK`-shaped predicates; it did not look for `NV_ASSERT.*GPU_REG_RD.*==.*GPU_REG_RD`-shaped invariants. Site #12 is the only confirmed instance in the audited tree; sweep-for-siblings is required.

Blackwell reachability is confirmed via the HAL dispatch: `kfspSendPacket_GB100` (`kern_fsp_gb100.c:276-295`) tail-calls `kfspSendPacket_GH100` when `PDB_PROP_KFSP_USE_MNOC_CPU` is unset, which is the default on consumer Blackwell parts including the 5090.

## Trigger sequence

1. Daemon brings device to a healthy bound state; FSP-driven operation in progress (e.g. early init, control packet send via `kfspSendPacket`).
2. Test fires surprise-removal A mid-`kfspSendPacket`.
3. `_kfspWriteToEmem_GH100` reads `NV_PFSP_EMEMC(idx)` to compute `ememOffsetStart`, performs the write loop, then reads `NV_PFSP_EMEMC(idx)` again to compute `ememOffsetEnd`. Both reads return `0xFFFFFFFF` (sink active).
4. The invariant `(end - start) == wordsWritten` fails because `end == start` (both `0xFFFFFFFF` produce identical bitfield extracts) and `wordsWritten > 0`. `NV_ASSERT_FAILED` fires at `kern_fsp_gh100.c:649`. Cascade descendants of this assert (FSP error-handling path) then run against the dead bus.

## Expected post-patch behaviour

Site-local fix per C5 v4 G9. Two viable shapes:

1. **Top-of-function `osIsGpuBusDead` check** returning `NV_ERR_GPU_IS_LOST` before any register read — symmetric with how `_issueRpcAndWait` (Funnel 3) gates RPCs. ~5 lines added at function entry.
2. **Early-out on first dead-bus read** at line 621 (the existing first `ememOffsetStart` read): if the read returns `0xFFFFFFFF`, return `NV_ERR_GPU_IS_LOST` immediately instead of proceeding to the write loop and final check. ~3 lines at the existing read site.

The sweep for sibling sites uses a regex like `NV_ASSERT.*GPU_REG_RD.*==.*GPU_REG_RD` (or simpler: `NV_ASSERT.*GPU_REG_RD` and audit each hit for the invariant-on-multiple-reads pattern). Any match without an `osIsGpuBusDead` precondition is a candidate G9-like site.

## Assertion shape

**Single-site (after surprise-removal A during FSP packet-send):**

- `dmesg | grep -cE 'NV_ASSERT_FAILED.*kern_fsp_gh100\.c:649'` MUST be 0.
- `_kfspWriteToEmem_GH100` MUST return `NV_ERR_GPU_IS_LOST` and the status MUST propagate up the call chain without further assertions.

**Telemetry presence (canonical sink log):**

- One log line per detection: `"_kfspWriteToEmem_GH100: dead bus detected pre-invariant"` (or the project's chosen canonical-log convention for G9) SHOULD fire once per episode.

**Pattern-class audit (source-side, CI):**

- `grep -rnE 'NV_ASSERT\b.*GPU_REG_RD' src/nvidia/` MUST be reviewed; each hit audited for the "arithmetic-on-dead-bus-reads" pattern.
- Any matching site WITHOUT an `osIsGpuBusDead` precondition (or equivalent funnel-1 short-circuit) MUST be flagged as a candidate G9-like site requiring its own fix.
- The current single-site identification (line 649) MUST hold; new occurrences from driver-version updates trigger the audit.

**Cascade absence:**

- The descendant FSP error-handling path MUST NOT fire any additional `NV_ASSERT_FAILED` lines attributable to the same surprise-removal window.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same primitive used by F03/F04/F15/F16.
- **FSP EMEM register backend.** The substrate must expose at minimum the `NV_PFSP_EMEMC(idx)` register (and adjacent EMEM control registers used by `kfspSendPacket`). When the substrate is in the "sink active" state, reads to these registers return `0xFFFFFFFF`. The test harness exercises an FSP packet-send path that reaches `_kfspWriteToEmem_GH100` (which on Blackwell requires routing through `kfspSendPacket_GB100` with `PDB_PROP_KFSP_USE_MNOC_CPU` unset — the substrate need not model MNOC at all; absence of the property is the default).
- The substrate does NOT need real FSP firmware. The path is exercisable as long as the EMEM register reads complete with the dead-bus sentinel and the kernel-side write loop runs to completion before the second read.

## Source citations

- `cascade-scope-audit.md` §"Site 12 (`kern_fsp_gh100.c:649`) — Blackwell EMEM path" and §"Funnel reachability audit" §"Gap at site 12"
- `cascade-class-design-v4.md` §"ENTRY-POINT GUARDS" G9 "kfsp arithmetic invariants"
- MISSION-1 E07 Run 3 forensic record (2026-05-26) — discovery site
- `g_kern_fsp_nvoc.c:343-356` (HAL dispatch for Blackwell); `kern_fsp_gb100.c:276-295` (Blackwell wrapper tail-call)
- `kern_fsp_gh100.c:621` (first read site, candidate early-out location); `kern_fsp_gh100.c:649` (invariant assertion site)
