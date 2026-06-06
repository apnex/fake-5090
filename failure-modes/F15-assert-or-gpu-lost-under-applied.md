# F15 — `NV_ASSERT_OR_GPU_LOST` macro defined but under-applied at teardown sites

| Field | Value |
|---|---|
| **Sources** | C5 (v3 + v4 site sweep) |
| **Confidence** | field-bug |
| **Predecessor evidence** | MISSION-1 E07 Run 2 (2026-05-26); cascade-scope-audit per-site classification table (sites 1, 2, 3, 6, 7, 8, 11, 13) |
| **fake-5090 mechanism** | Surprise-removal A primitive + source-audit gate (substrate emits `NV_ERR_GPU_IS_LOST` status without test-harness crash; CI source check enforces macro application) |

## Symptom (driver view)

The macro `NV_ASSERT_OR_GPU_LOST` is defined in `nv-gpu-lost.h:74` to accept three statuses as non-fatal: `NV_OK`, `NV_ERR_GPU_IN_FULLCHIP_RESET`, and `NV_ERR_GPU_IS_LOST`. Only a small number of call sites use it (`rs_client.c:855`, `rs_server.c:272` — the two C5 v3 sites). The remaining teardown sites use a bare `NV_ASSERT(status == NV_OK || status == NV_ERR_GPU_IN_FULLCHIP_RESET)` clause that trips when status is `NV_ERR_GPU_IS_LOST` — exactly the status produced after the cascade has begun.

The sites identified by `cascade-scope-audit.md` are: `kernel_graphics.c:2608`, `fecs_event_list.c:1623`, `fecs_event_list.c:1639`, `vaspace_api.c:573`, `mem.c:178`, `rs_server.c:1388`, `osinit.c:2462`, and `gpu_user_shared_data.c:248` — 8 sites in tree, all on the universal `nv_pci_remove → rm_shutdown_adapter → RmShutdownAdapter → engstate/resserv teardown` chain. None of these sites is `is_external_gpu`-gated; all fire equally for discrete RTX 5090, OCuLink-attached 5090, TB4 eGPU, ARM64 3060, and PRIME laptop A2000.

## Trigger sequence

1. Daemon brings the device to a healthy bound state under the substrate.
2. Test issues the surprise-removal A primitive — MMIO completions return the dead-bus sentinel (`0xFFFFFFFF` for 32-bit, `0xFFFF`/`0xFF` for 16/8-bit) AND `pci_dev_is_disconnected` is set on the kernel side.
3. The driver detects bus loss (per F4 funnel), sets `PDB_PROP_GPU_IS_LOST`, and status `NV_ERR_GPU_IS_LOST` begins propagating up the teardown chain as cleanup paths return their work products.
4. Each cleanup-path assertion that uses bare `NV_ASSERT` instead of `NV_ASSERT_OR_GPU_LOST` trips when called with the lost-GPU status. The cascade is a tree of assertion failures rooted at the per-site bare-assert calls, not a single root.

## Expected post-patch behaviour

Every cleanup-path assertion that can legitimately observe `NV_ERR_GPU_IS_LOST` accepts it as non-fatal via `NV_ASSERT_OR_GPU_LOST`. The macro is applied uniformly at the 8 audited sites; the cascade-class v4 design (G1–G10) keeps the macro family at the 10 cleanup sites as defense-in-depth (funnels short-circuit upstream but cleanup must complete; cleanup-path asserts must accept `NV_ERR_GPU_IS_LOST`). Future audits add the macro at any newly-discovered site that meets the contract (must execute on the teardown chain AND can be reached with a lost-GPU status). Cascade collapses to one clean teardown with zero spurious asserts.

## Assertion shape

**Per-site assertion absence (after surprise-removal A fires):**

- `dmesg | grep -cE 'NV_ASSERT_FAILED.*kernel_graphics\.c:2608'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*fecs_event_list\.c:1623'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*fecs_event_list\.c:1639'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*vaspace_api\.c:573'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*mem\.c:178'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*rs_server\.c:1388'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*osinit\.c:2462'` MUST be 0.
- `dmesg | grep -cE 'NV_ASSERT_FAILED.*gpu_user_shared_data\.c:248'` MUST be 0.

**Macro-application audit (source-side, run in CI):**

- `grep -nE 'NV_ASSERT\b' src/nvidia/.../{kernel_graphics,fecs_event_list,vaspace_api,mem,rs_server,osinit,gpu_user_shared_data}.c` at the 8 line numbers MUST NOT match the bare pattern `NV_OK.*NV_ERR_GPU_IN_FULLCHIP_RESET` without the trailing `NV_ERR_GPU_IS_LOST` clause.
- Any future site introducing a teardown-chain assertion without `NV_ASSERT_OR_GPU_LOST` MUST be flagged by the audit gate.

**Cleanup completion:**

- After surprise-removal A, `nv_pci_remove` completes within bounded time (target: ≤500 ms).
- No host wedge; subsequent `lspci -nn -s <BDF>` succeeds and shows the substrate's "device removed" state cleanly.
- Subsequent `modprobe -r nvidia` followed by `modprobe nvidia` succeeds without operator intervention.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive.** Substrate must transition from "healthy" to "MMIO returns dead-bus sentinel + `pci_dev_is_disconnected` set" atomically. This is the same primitive that F03/F04 exercise; F15 layers a per-site assertion-absence check on top of the existing surprise-removal A run.
- **Source-audit gate (CI).** A `git grep` regex run as part of the patch-set CI catches new bare-`NV_ASSERT` introductions at the 8 known sites or any site matching the teardown-chain reachability pattern. This is a substrate-independent assertion run on the source tree, not the runtime.

## Source citations

- `cascade-scope-audit.md` §"Per-site classification table" (sites 1, 2, 3, 6, 7, 8, 11, 13) and §"Funnel reachability audit (Option 1 design completeness)"
- `cascade-class-design-v4.md` §"v4 architecture" (G1–G10 entry-point guards) and §"REDUNDANCY ELIMINATION"
- `C5-crash-safety.md` §"v3 amendment lineage" and §"Two new requirement classes triangulated to v3"
- `nv-gpu-lost.h:74` (macro definition); `rs_client.c:855` and `rs_server.c:272` (existing call sites)
- MISSION-1 E07 Run 2 (2026-05-26) — field forensic record of the cascade in production
