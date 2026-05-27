# F1 — Single-read commit to permanent GPU-lost

| Field | Value |
|---|---|
| **Sources** | C3-gpu-lost-retry, C5-crash-safety (sentinel propagation) |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu failure-modes-index #2 (Mode B silent freeze); NVIDIA open-driver issue #979 |
| **fake-5090 mechanism** | Config-space transient blip (Phase-1 substrate) |

## Symptom (driver view)

A single transient PCIe config-space read of `NV_PMC_BOOT_0` returns `0xFFFFFFFF` (or any value not matching the cached chip identifier). The unpatched `osHandleGpuLost` commits the device to permanent GPU-lost state on that one mismatch — subsequent reads succeed, but the driver has already flipped its RM-internal `gpuLost` marker and refuses to use the device for the lifetime of the binding. Manifests in the field as the Mode B "silent freeze" reported in NVIDIA-Linux issue #979.

## Trigger sequence

1. Daemon serves normal config space (matching `PMC_BOOT_0` identifier) until the test arms the fault.
2. Test arms: "on the next read of `PMC_BOOT_0` from any thread tagged `nvidia*`, return one mismatching value, then resume normal."
3. Driver code path under test (probe, runtime-PM resume, or any `osHandleGpuLost` preflight) reads `PMC_BOOT_0`.
4. Daemon returns the one mismatch.
5. Daemon resumes serving the correct identifier for all subsequent reads.
6. (Optional sweep) repeat with the mismatch on read N=1, 2, 4, 8, … to characterise budget.

## Expected post-patch behaviour

C3 introduces a bounded retry budget (`N` reads with `M µs` spacing) on the preflight read. A single transient blip is silently absorbed; the matching identifier on a later attempt closes the episode as "recovered transient." The driver does NOT flip its `gpuLost` marker. A real disconnect (budget exhausted with all reads mismatching) still escalates correctly. C5 belt-and-braces: even if the marker IS flipped downstream, both the RM-internal and Linux `pdev`-side markers are propagated together so no path diverges.

## Assertion shape

**PASS conditions** (after a single-blip trigger):

- `dmesg | grep -E '^NVRM: GPU [0-9a-f:.]+: GPU-lost check: transient PCIe read recovered after [0-9]+ retr(y|ies)$'` — exactly one matching line.
- The driver's binding to the device is still active: `readlink /sys/bus/pci/devices/<BDF>/driver` resolves to `nvidia`.
- Subsequent `cat /sys/bus/pci/devices/<BDF>/tb_egpu_qwd_state` (if A2 is in the build) returns `READY` (not `LOST`).
- No `TB_EGPU_GPU_STATE=PERMANENT_FAIL` uevent fired in the test window.

**FAIL conditions** (unpatched-driver shape, must NOT occur post-patch):

- Any `NVRM: ... GPU is lost` line within the trigger window.
- Driver unbinding itself from the device.

**Boundary case** — budget exhaustion (all `N` reads mismatch): the driver SHOULD escalate to lost-state; this is the genuine-disconnect path and must remain correct. Test this separately by arming "next `N+1` reads all return `0xFFFFFFFF`."

## fake-5090 mechanism mapping

- **Phase 1 — Hello-World PCIe endpoint** is sufficient: config-space serving with per-register, per-read scriptable overrides.
- The blip injection is a vfio-user MMIO/config-space callback that returns the scripted value once, decrements a counter, then falls through to the cached real value.
- No AER injection needed for F1 (this is a *config-space* transient, not a TLP error).

## Source citations

- `C3-gpu-lost-retry.md` §"Requirement: bounded retry budget" and §"Recovered-transient log line" (lines ~80–110, ~166–170 for the exact format string).
- `C5-crash-safety.md` §"Fresh dead-bus read on previously-healthy GPU" — explains why C5's sentinel work alone is insufficient without C3 (a single read still commits without retry).
- aorus-5090-egpu `docs/failure-modes-index.md` row 2 (Mode B silent freeze) for predecessor field evidence.
- Upstream: `NVIDIA-Linux issue #979` ("This doesn't support PEX Reset and Recovery yet.") — the canonical bug-report this F-entry closes.
