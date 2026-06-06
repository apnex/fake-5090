# F19 — `nvGpuOpsReportFatalError` system-global escalation — cross-GPU contamination

| Field | Value |
|---|---|
| **Sources** | Follow-up F1 (out of scope for C5 v4 base; documented as known gap) |
| **Confidence** | field-bug |
| **Predecessor evidence** | NVIDIA open-gpu-kernel-modules issue #979 — jciolek report (external 5090 surprise-removal triggered Xid 154 on internal RTX 3060); `cascade-class-design-v4.md` §"Follow-up F1" |
| **fake-5090 mechanism** | Multi-device topology (two substrate instances, one with surprise-removal A primitive fired) + UVM-shim path observation |

## Symptom (driver view)

`nvGpuOpsReportFatalError` (`src/nvidia/src/kernel/rmapi/nv_gpu_ops.c:11678`) is a UVM-side escalation entry point whose signature takes no GPU identity:

```c
void nvGpuOpsReportFatalError(NV_STATUS error)
{
    OBJSYS *pSys = SYS_GET_INSTANCE();
    NV_PRINTF(LEVEL_ERROR,
              "uvm encountered global fatal error 0x%x, ...\n", error);
    sysSetRecoveryRebootRequired(pSys, NV_TRUE);  /* system-wide flag */
}
```

The function unconditionally calls `sysSetRecoveryRebootRequired(pSys, NV_TRUE)` — a system-global "Node Reboot Required" flag. Any UVM fatal error from any GPU triggers Xid 154 on every GPU enumerated in `gpumgr`, regardless of which GPU originated the error.

The #979 jciolek case is the canonical instance: external 5090 surprise-removal → UVM observes the dead device → `nvGpuOpsReportFatalError` fires → `sysSetRecoveryRebootRequired` sets the global flag → internal RTX 3060 (which never lost its bus, never had any driver-internal failure) ALSO shows Xid 154 / Node Reboot Required and is now in a "must reboot host to recover" state per the userspace ABI.

The per-GPU sink-state primitive `cleanupGpuLostStateAtomic(pGpu, ...)` introduced by the C5 v4 architecture CANNOT prevent this escalation. The sink-state machine is per-`OBJGPU`; `nvGpuOpsReportFatalError` does not consult any `OBJGPU` — it takes a bare `NV_STATUS`, queries `OBJSYS`, sets a system-global flag. The escalation path is structurally outside the sink primitive's reach.

This is why the C5 v4 design tracks it as **Follow-up F1** — a separate patch that extends the signature to `nvGpuOpsReportFatalError(OBJGPU *pGpu, NV_STATUS error)` (or `NvU32 gpuId`), updates callers (`rm-gpu-ops.c:918` + the UVM ABI shim), and gates `sysSetRecoveryRebootRequired` on whether a non-lost GPU remains in the system.

## Trigger sequence

1. Substrate is configured with TWO PCI devices: GPU-A ("external", subject to surprise removal) and GPU-B ("internal", remains healthy throughout).
2. Daemon brings both devices to healthy bound state. Both visible via `nvidia-smi -L`; both have UVM channels active (some active CUDA context on each is helpful to ensure UVM is actively engaged with both).
3. Test fires surprise-removal A on GPU-A only.
4. Driver detects bus loss on GPU-A; per-GPU sink-state set for GPU-A (via the C5 v4 primitive).
5. UVM observes the dead GPU-A. UVM's internal fatal-error path eventually calls `nvGpuOpsReportFatalError(NV_ERR_*)`.
6. `nvGpuOpsReportFatalError` calls `sysSetRecoveryRebootRequired(pSys, NV_TRUE)`. The global flag is set.
7. Subsequent queries against GPU-B observe Xid 154 / Node Reboot Required despite GPU-B never having experienced any failure.

## Expected post-patch behaviour (after F1 follow-up lands)

The signature change makes the function `nvGpuOpsReportFatalError(OBJGPU *pGpu, NV_STATUS error)`. Implementation gates `sysSetRecoveryRebootRequired` on a `gpumgr`-enumeration check: only set the global flag if zero non-lost GPUs remain. With F1 applied:

1. After surprise-removal A on GPU-A, GPU-A's per-GPU state is "lost" (sink set).
2. GPU-B remains healthy; `gpumgr` enumeration returns "1 healthy GPU remains."
3. `nvGpuOpsReportFatalError(pGpuA, error)` runs; the gating check sees a healthy GPU remains; the global flag is NOT set.
4. GPU-B's userspace state remains "healthy"; no Xid 154 on GPU-B.

Until F1 lands, F19 is a known, documented gap. The fake-5090 test for F19 in v1 of the failure index measures the **current** behaviour (cross-contamination present) and serves as the regression-witness for when F1 eventually lands.

## Assertion shape

**Pre-F1 baseline (current behaviour — documents the gap):**

- Two-device substrate, surprise-removal A on GPU-A.
- `dmesg | grep -cE 'NVRM: Xid.*154.*GPU-A_BDF'` SHOULD be ≥ 1.
- `dmesg | grep -cE 'NVRM: Xid.*154.*GPU-B_BDF'` SHOULD be ≥ 1 (this is the cross-contamination — currently expected; F1 fixes it).
- `nvidia-smi -i <GPU-B_index>` SHOULD report a degraded state attributable to the global flag.

**Post-F1 behaviour (when the follow-up patch lands):**

- Same trigger; `dmesg | grep -cE 'NVRM: Xid.*154.*GPU-B_BDF'` MUST be 0.
- `nvidia-smi -i <GPU-B_index>` MUST return healthy.
- `nvidia-smi -i <GPU-A_index>` MUST report the lost-GPU state per F18/F15 expectations.

**Signature-change audit (source-side, CI — triggered when F1 lands):**

- `grep -nE 'nvGpuOpsReportFatalError\s*\(' src/nvidia/ kernel-open/` — every match MUST take a `OBJGPU *` or `NvU32 gpuId` as first parameter.
- `grep -nE 'sysSetRecoveryRebootRequired' src/nvidia/` — every call MUST be gated by a `gpumgr` enumeration check (no bare unconditional calls).

## fake-5090 mechanism mapping

- **Phase 3 multi-device substrate.** The test harness instantiates two substrate processes (or one substrate exposing two vfio-user endpoints), each presenting a distinct PCI device to the host. Both devices are brought up healthy; the surprise-removal A primitive is fired against ONE only.
- **UVM-path observation.** The test harness needs to confirm UVM actually called `nvGpuOpsReportFatalError`. The simplest observation is checking `dmesg` for the canonical `"uvm encountered global fatal error"` log line and the subsequent `sysSetRecoveryRebootRequired` evidence (Xid 154 on the non-failed device).
- **Polarity matters.** The test MUST verify GPU-B (the non-failed device) shows the cross-contamination symptom — not just that GPU-A failed. A single-device run cannot distinguish F19 from F03/F04/F15.

## Source citations

- `cascade-class-design-v4.md` §"Why per-GPU not global — and what it does NOT fix" and §"Follow-up F1: `nvGpuOpsReportFatalError` signature change (out of scope for v4 base)"
- NVIDIA open-gpu-kernel-modules issue #979 — jciolek report (external 5090 surprise-removal → Xid 154 on internal 3060)
- `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c:11678` (function definition)
- `rm-gpu-ops.c:918` (current caller, no `pGpu`); UVM ABI shim (cross-component coordination point)
- Memory: `project_issue_979_upstream_state_2026_05_22` — upstream filing context
