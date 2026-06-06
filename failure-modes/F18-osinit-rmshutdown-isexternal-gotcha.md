# F18 — `osinit.c:2462` `RmShutdownAdapter` post-`gpuStateDestroy` assert + `is_external_gpu ||` gotcha

| Field | Value |
|---|---|
| **Sources** | C5 (v4 site sweep) — site #11 in the cascade-scope audit |
| **Confidence** | field-bug |
| **Predecessor evidence** | MISSION-1 E07 Run 3 forensic record (2026-05-26); cascade-scope-audit site #11 + the `is_external_gpu` misread analysis at `osinit.c:2413` |
| **fake-5090 mechanism** | Surprise-removal A primitive + source-audit gate (sweep methodology that catches the `is_external_gpu` boolean-short-circuit misread class) |

## Symptom (driver view)

`RmShutdownAdapter` (`osinit.c` ~lines 2400–2470) has a `NV_ASSERT` shortly after the `gpuStateDestroy` call. Site #11 (`osinit.c:2462`) fires on every device removal where the destroy returns `NV_ERR_GPU_IS_LOST`. This is the same teardown chain F15 documents; this entry covers the **specific gotcha** at this site that nearly mis-classified the cascade as eGPU-only during the v3 audit.

The conditional immediately preceding the teardown block at `osinit.c:2413` reads:

```c
if (!nv->is_external_gpu || serverLockAllClients(&g_resServ) == NV_OK)
```

with a leading comment "LOCK: lock all clients in case of eGPU hot unplug, which will not wait for all existing RM clients to stop using the GPU." This is easy to misread as "the whole shutdown block is eGPU-gated." It is NOT. The `||` short-circuits: on a non-eGPU (discrete RTX 5090 in a regular PCIe slot, OCuLink 5090, ARM64 3060), `!nv->is_external_gpu` evaluates TRUE, the condition is satisfied without ever calling `serverLockAllClients`, and the teardown block proceeds. The assertion at line 2462 fires identically for discrete-card surprise removal and for eGPU surprise removal.

The failure mode therefore has two heads:
1. **The site itself** — bare `NV_ASSERT` that trips on `NV_ERR_GPU_IS_LOST` post-`gpuStateDestroy` (this is F15's general pattern at a specific site).
2. **The classification trap** — a future audit that hits this site might mis-classify it as transport-specific and skip it from the Option-1 Core cascade-class fix, leaving discrete-card users (issue #1134 RTX 3090, #916 RTX 4090, #1045 RTX 5080, #888 RTX 5090, #461 ARM64 3060, #776 PRIME laptop A2000, #900 OCuLink 5090) unfixed.

## Trigger sequence

1. Daemon brings the device to a healthy bound state. The substrate's `is_external_gpu` reporting can be either `0` (discrete-like) or `1` (eGPU-like) — the failure reproduces on both.
2. Test issues surprise-removal A. Driver detects bus loss; `PDB_PROP_GPU_IS_LOST` set; status propagation begins.
3. `nv_pci_remove` runs; `rm_shutdown_adapter` runs; `RmShutdownAdapter` reaches the `gpuStateDestroy` call which returns `NV_ERR_GPU_IS_LOST`.
4. The bare `NV_ASSERT` at line 2462 trips. Cascade descendants fire.

The reproduction MUST be run **twice** — once with the substrate reporting `is_external_gpu == 0` and once with `1` — to demonstrate the cascade is transport-agnostic. A test that only runs with `1` would not catch a regression where someone re-introduced an `is_external_gpu` gate.

## Expected post-patch behaviour

The site itself gets G1-style treatment: either converted to `NV_ASSERT_OR_GPU_LOST` (preferred, consistent with F15) or fronted by a top-of-`RmShutdownAdapter` `osIsGpuBusDead` early-return for the post-`gpuStateDestroy` block. The cascade-class v4 architecture's funnel set covers this site via the existing C5 v1 marker propagation.

The **audit methodology gate** (the documentation half of F18) is the requirement that any future per-site audit MUST verify the `is_external_gpu` checks it encounters are actually transport-class gates — not lock-strategy selectors, not log-string selectors, not other-purpose selectors — before classifying a site as transport-specific. The cascade-scope-audit document encodes this discipline; the audit gate in CI is the test that this discipline holds (a regression where someone re-introduces a real `is_external_gpu` gate on the teardown path MUST be flagged).

## Assertion shape

**Per-site assertion absence (after surprise-removal A, both `is_external_gpu` polarities):**

- `dmesg | grep -cE 'NV_ASSERT_FAILED.*osinit\.c:2462'` MUST be 0 in both polarities.

**Transport-polarity reproducibility:**

- Run the surprise-removal A scenario with substrate `is_external_gpu == 0`. Cleanup completes; no `NV_ASSERT_FAILED` at line 2462.
- Run the same scenario with substrate `is_external_gpu == 1`. Cleanup completes; no `NV_ASSERT_FAILED` at line 2462.
- The two runs MUST be observationally identical in cascade behaviour. Any divergence indicates a regression toward transport-specific gating.

**Audit-methodology gate (source-side, CI):**

- `grep -nE 'is_external_gpu' kernel-open/nvidia/.../osinit.c` MUST be reviewed; each hit classified as either "transport gate" (load-bearing), "lock-strategy selector" (cosmetic, like the `||` site at 2413), or "log-string selector" (cosmetic).
- Any NEW `is_external_gpu` check introduced on the `nv_pci_remove → rm_shutdown_adapter → RmShutdownAdapter` chain that fences out the teardown block MUST be flagged as a regression — the audit confirmed the chain is universally reached by both discrete and eGPU surprise removals.

**Cleanup completion:**

- After surprise-removal A in either polarity, `nv_pci_remove` completes within bounded time (target: ≤500 ms).
- Subsequent `modprobe -r nvidia ; modprobe nvidia` succeeds without operator intervention in both polarities.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15/F16/F17.
- **`is_external_gpu` polarity control.** The substrate must expose a configuration toggle that sets the kernel-visible `nv->is_external_gpu` field. In practice this is driven by the topology the substrate presents (whether the device sits behind a TB/USB4 bridge in the PCI hierarchy, whether `pdev->untrusted` is set, etc. — see F07 for the detection machinery). Test runs the same cascade in both polarities; assertion verifies no behavioural difference.
- **Source-audit gate (CI).** A `git grep` review of `is_external_gpu` occurrences on the teardown chain. This is a substrate-independent assertion run on the patched source tree.

## Source citations

- `cascade-scope-audit.md` §"Per-site classification table" row 11 and §"Site 11 (`osinit.c:2462`) — the misleading `is_external_gpu` check"
- `cascade-class-design-v4.md` §"Per-issue design implications" and §"ENTRY-POINT GUARDS"
- MISSION-1 E07 Run 3 forensic record (2026-05-26) — site #11 discovery
- Issue tracker survey: #1134 (RTX 3090 discrete), #916 (RTX 4090 discrete), #1045 (RTX 5080 discrete), #888 (RTX 5090 discrete), #461 (ARM64 3060), #776 (PRIME laptop A2000), #900 (OCuLink 5090) — all reach the same site, none filtered by `is_external_gpu`
- `osinit.c:2413` (the `||` gotcha conditional); `osinit.c:2462` (the assertion site)
