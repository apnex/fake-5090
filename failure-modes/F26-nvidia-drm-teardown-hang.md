# F26 — `nvidia_drm` teardown hang — DRM-subsystem deregistration blocks on dead GPU

| Field | Value |
|---|---|
| **Sources** | A4 (close-path-telemetry — partial coverage); C5 (sink-state primitive must propagate to DRM path) |
| **Confidence** | hypothesis (design-time hardening identified during audit; no field incident attributed) |
| **Predecessor evidence** | `cascade-scope-audit.md` §"DRM subsystem teardown path" — `nvidia-drm` is a separate kmod whose teardown enters the resserv layer through paths the v3 audit verified are funnel-1-reachable, but the DRM-side hooks themselves are NOT inside the C5 v3 audit scope |
| **fake-5090 mechanism** | Surprise-removal A primitive + DRM-subsystem activation (substrate must be bound by `nvidia-drm` as well as `nvidia`; test exercises `drm_dev_unregister` path during cascade) |

## Symptom (driver view)

`nvidia-drm.ko` is a separate kernel module that registers a DRM device on top of `nvidia.ko`'s per-GPU `OBJGPU`. Under normal teardown, `nv_pci_remove` triggers `nvidia-drm`'s `pci_remove` callback (via the kernel's PCI driver framework), which calls `drm_dev_unregister` → DRM-subsystem deregistration → final `drm_dev_put`.

During the deregistration path, several DRM-subsystem helpers may issue MMIO reads or RPCs to the underlying device — most notably during framebuffer-unmap, mode-set-teardown, and per-CRTC cleanup. If the device is already known-lost (cascade in progress), these helpers should short-circuit via the sink primitive's funnels; if they don't (because the DRM-side code is in `nvidia-drm.ko` rather than `nvidia.ko` and may not consult the sink primitive directly), the helpers wait for the device's MMIO completions and block for the per-operation timeout (typically 1-5 seconds per operation, several operations per teardown).

The cascade-scope-audit confirmed that the resserv-layer paths the DRM-side helpers eventually call (kernel_graphics teardown, vaspace_api teardown, mem.c teardown) ARE in the v3 audit and ARE funnel-protected. But the DRM-side wrapper functions — the ones in `nvidia-drm.ko` itself, before they reach resserv — were not audited in v3. If a DRM-side wrapper does its own MMIO read or its own bounded wait before delegating to resserv, that wait is unprotected.

The class of bug is the same as F16 (RPC funnel gap) but on the DRM path: a wrapper that doesn't consult sink before doing its own bounded operation. The audit gap exists because the DRM-side code is in a separate module that the cascade-scope-audit's `git grep` did not traverse.

## Trigger sequence

1. Daemon brings device to a healthy bound state. Both `nvidia` and `nvidia-drm` are bound; the device has an active DRM device node (`/dev/dri/cardN`). Optionally: a display is connected and a mode-set is active, OR a userspace DRM client (compositor, X server) has open file descriptors on the DRM device.
2. Test fires surprise-removal A.
3. Driver-side cascade detector fires; sink-state set; nvidia.ko-side teardown begins.
4. Kernel's PCI framework invokes `nvidia-drm`'s `pci_remove` callback.
5. `nvidia-drm` enters `drm_dev_unregister`; DRM-subsystem teardown begins.
6. DRM-side helpers issue framebuffer-unmap / mode-set-teardown / CRTC-cleanup operations. If any of these wrappers do their own MMIO read or bounded wait BEFORE delegating to the funnel-protected resserv layer, those waits block for the per-operation timeout.
7. Cumulative delay across DRM-teardown operations: 5-30 seconds per cascade event. The host-visible symptom is "DRM device removal hangs after the cable is pulled, even though `nvidia.ko` reports the GPU is gone within 500 ms."

## Expected post-patch behaviour

A4's close-path-telemetry already covers part of this surface (the `/dev/nvidia0` close-path; see F12). F26 is the DRM-side equivalent: any wrapper in `nvidia-drm.ko` that issues its own MMIO read or its own bounded wait before delegating to a funnel-protected path MUST be fronted by an `osIsGpuBusDead`-equivalent check.

The implementation requires:
1. **A `nvidia-drm`-visible sink-state probe.** `osIsGpuBusDead` lives in `nvidia.ko`; `nvidia-drm` needs a symbol to call. Either export `osIsGpuBusDead` cross-module, or add a thin wrapper `nv_drm_is_gpu_bus_dead(pNvDev)` that consults the underlying `nvidia.ko` `OBJGPU`'s sink state.
2. **Audit of `nvidia-drm.ko` source** to enumerate the wrapper sites — similar to the C5 v3 audit but scoped to the DRM module's source tree (which is in `kernel-open/nvidia-drm/` rather than `src/nvidia/`).
3. **Per-site `osIsGpuBusDead` gate** at each enumerated wrapper, with short-circuit returning the equivalent of `NV_ERR_GPU_IS_LOST` cast to DRM's error-code domain (e.g. `-ENODEV` or `-EIO`).

The audit is the heavy lift. The per-site change is one-line each. The cross-module symbol export is a 3-line build-system change.

## Assertion shape

**DRM-teardown timing (after surprise-removal A with DRM active):**

- Total wall-clock from surprise-removal A to `nvidia-drm` `pci_remove` completion MUST be ≤ 1 s.
- Pre-patch reference (regression-witness): without the F26 fix, the same scenario with DRM active and a mode-set in progress takes ≥ 10 s.

**Per-wrapper short-circuit verification:**

- The DRM-side audit (a new artefact, output of the `nvidia-drm.ko` source review) enumerates sites N₁, N₂, ..., Nₘ. For each site, a `dmesg` canonical-log assertion verifies the gate fires:
  - `dmesg | grep -cE 'nv_drm.*<site_name>.*GPU bus dead'` MUST be ≥ 0 (sites may not fire if the cascade path didn't reach them) and the cumulative count across all sites MUST be ≤ M (one fire per site at most, per cascade event).

**DRM-client error propagation:**

- A userspace DRM client (test harness opens `/dev/dri/cardN` and issues an ioctl during the cascade window) MUST receive an immediate error response (e.g. `-ENODEV`) rather than blocking.

**Cross-module sink-state visibility (smoke test):**

- `cat /sys/module/nvidia/parameters/<sink-state-probe>` and `cat /sys/module/nvidia_drm/parameters/<dr-side-sink-probe>` (if exposed by debug interfaces) MUST report sink-state set within ≤ 500 ms of surprise-removal A firing.

**Negative — F26 fix MUST NOT break normal DRM teardown:**

- A clean `modprobe -r nvidia-drm` invocation (no surprise removal) MUST complete normally — sink-state is not set, the `osIsGpuBusDead`-equivalent check returns false, every DRM-side wrapper proceeds normally.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25.
- **DRM-subsystem activation in test setup.** The substrate doesn't need to model DRM internals; the host-side `nvidia-drm.ko` does its work against the substrate's existing BAR0/BAR1 register-and-memory surface. The test harness ensures `nvidia-drm` is bound (verifiable via `lsmod | grep nvidia_drm`), optionally opens `/dev/dri/cardN` to simulate a userspace client, and fires surprise-removal A while a DRM operation is in flight.
- **Userspace DRM client (optional but recommended).** A minimal test harness binary that opens `/dev/dri/cardN`, issues `drmIoctl(DRM_IOCTL_GET_CAP, ...)` in a loop, and reports the wall-clock latency of each call. During the cascade window, latency MUST stay bounded (≤ 100 ms per call); without F26's fix, calls block for the per-operation timeout.

## Source citations

- `cascade-scope-audit.md` §"Audit scope boundary — what was and wasn't traversed" §"`nvidia-drm.ko` is out of scope for v3"
- `cascade-class-design-v4.md` §"Open questions / risks" §"DRM-subsystem teardown path"
- A4-close-path-telemetry intent doc — partial coverage of `/dev/nvidia0` close-path; F26 is the DRM-side equivalent
- F12 entry (`/dev/nvidia0` close-path wedge — the sibling failure on the `nvidia.ko` side)
- F16 entry (the pattern-class precedent: wrapper that doesn't consult sink before bounded operation)
