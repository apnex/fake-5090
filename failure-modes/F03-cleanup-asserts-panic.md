# F3 — Cleanup-path asserts panic on lost GPU (crash dump / RPC / resserv)

| Field | Value |
|---|---|
| **Sources** | C5-crash-safety |
| **Confidence** | field-bug |
| **Predecessor evidence** | aorus-5090-egpu Lever J-2 patches 0002–0004 (engine-dump short-circuit), Lever N patch 0006 (GSP RPC short-circuit), Lever O patch 0008 (resserv asserts) |
| **fake-5090 mechanism** | Surprise-removal A (DeviceLost / 0xFFFFFFFF on MMIO + config) |

## Symptom (driver view)

Once the GPU is marked lost, the driver still runs cleanup paths (crash-dump collection, engine iteration, GSP RPC frees, resserv resource teardown). Many of these contain `NV_ASSERT` calls that panic-style fail when an underlying RM call returns `NV_ERR_GPU_IS_LOST`. On unpatched code, a single dead-bus event during cleanup turns into a kernel-side `BUG()` / kernel panic — or, milder but still bad, an infinite engine-iteration loop reading `0xFFFFFFFF` and re-asserting every step.

Three concrete unpatched failure paths:

1. `rcdbAddRmGpuDump` iterates every engine even after the first one returns lost.
2. `nvDumpAllEngines` continues calling dump callbacks against the dead bus.
3. `rpcRmApiFree_GSP` and `_issueRpcAndWait` hang waiting for GSP responses that will never arrive.

## Trigger sequence

1. Daemon brings device to healthy bound state; driver allocates resources (open a client, create an engine context, allocate an RPC channel).
2. Daemon transitions device to surprise-removal A state: all MMIO and config reads return `0xFFFFFFFF`; no kernel-side `.remove()` notification fires (this is the *silent* signature — distinct from B/C/D).
3. Test exercises a cleanup path that crosses the now-dead bus:
   - **3a.** Trigger a crash-dump collection (`echo c > /proc/sysrq-trigger` is too broad; use the driver's own crash-dump entry point if exposed, or trigger via `force_trigger` on the recovery sysfs combined with a fault).
   - **3b.** Issue an `nvidia-smi` engine-listing call while the device is dead.
   - **3c.** Close the file descriptor of a GSP-using client allocation.

## Expected post-patch behaviour

C5 introduces the `NV_ASSERT_OR_GPU_LOST` family. Each touched assertion site converts a panic-style assert into:

- a soft `return NV_ERR_GPU_IS_LOST` (or equivalent), AND
- a one-shot log latch (log-once per site to prevent log storm).

Behaviour after the patch:

- `rcdbAddRmGpuDump` returns immediately when the first engine reports lost; does NOT iterate the rest.
- `nvDumpAllEngines` breaks out of its loop on the first `NV_ERR_GPU_IS_LOST` return.
- GSP RPC paths short-circuit instead of waiting for responses; resource cleanup *completes* instead of hanging.
- Each converted site emits exactly one log line per binding-lifetime (the latch).

## Assertion shape

**Crash-safety check (no panic):**

- After trigger 3a/3b/3c, `dmesg` MUST NOT contain `BUG:`, `kernel panic`, or `general protection fault` lines attributable to the test window.
- `uptime` of the kernel does not reset (no panic-reboot).

**Short-circuit observability (per-site latches):**

- `dmesg | grep -cE 'tb_egpu .*GPU is lost.*site=rcdbAddRmGpuDump'` returns exactly 1 (not 0, not 2+).
- `dmesg | grep -cE 'tb_egpu .*GPU is lost.*site=nvDumpAllEngines'` returns exactly 1.
- `dmesg | grep -cE 'tb_egpu .*GPU is lost.*site=rpcRmApiFree_GSP'` returns exactly 1.
- (Exact format strings live in C5 intent doc's logging table — check that doc when implementing tests.)

**Cleanup-completion check:**

- File-descriptor close returns within bounded time (e.g. `timeout 5 ./close-fd-test` exits 0, not 124).
- After cleanup, `lsof | grep nvidia` shows the closed FDs have actually released — they are not stuck in some "closing" state.

**Loop-bounding check:**

- During trigger 3b, count of MMIO reads served by the daemon must NOT grow unboundedly. A reasonable cap: ≤ (engine_count) + small constant. Without the patch, the daemon will see thousands of reads before the kernel gives up.

## fake-5090 mechanism mapping

- **Phase 1 + Surprise-removal A primitive.** The substrate needs:
  - A `device.die()` daemon command that transitions all MMIO/config reads to `0xFFFFFFFF` *without* firing a vfio-user `device_remove` event to the kernel (silent death).
  - MMIO read counter telemetry on the daemon side so the test can assert the loop-bounding property.
- No AER, no topology, no bridge reset needed for F3.

## Source citations

- `C5-crash-safety.md` §"Crash-dump short-circuit" (intent doc lines ~140–200), §"Engine dump loop" (~210–260), §"GSP RPC short-circuit" (~270–340), §"NV_ASSERT_OR_GPU_LOST macro family" (~400–460), §"Log-once latch per site" (~470–510).
- aorus-5090-egpu predecessor patches 0002, 0003, 0004 (Lever J-2 — engine-dump short-circuits), 0006 (Lever N — GSP RPC), 0008 (Lever O — resserv asserts). C5 supersedes these by introducing the macro family that obsoletes per-site one-off patches.
- aorus-5090-egpu `docs/failure-modes-index.md` — the row for "panic during cleanup of lost GPU" links to the diagnostic archive that originally surfaced this failure.
