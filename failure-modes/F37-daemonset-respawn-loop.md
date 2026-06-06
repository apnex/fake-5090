# F37 — DaemonSet respawn loop — Device Plugin Pod crash-restart-loops on dead GPU

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — k8s/cluster-layer concern; documented as adjacent failure mode that the injector's telemetry surface enables observing and as a workload-stability sibling to F36)* |
| **Confidence** | hypothesis |
| **Predecessor evidence** | `consumer-holders-and-teardown-future-work.md` (project-internal); NVIDIA device-plugin source code references to init-time GPU enumeration; community reports of device-plugin DaemonSet crash-loops on cards in error states |
| **fake-5090 mechanism** | **Cluster-layer only — same telemetry-surface dependency as F36.** Substrate provides lost-GPU state; F37 is the case where the Device Plugin Pod itself enters a crash-restart loop because its init-time GPU enumeration trips on the dead card |

## Symptom (driver view)

The NVIDIA Device Plugin runs as a DaemonSet — one Pod per node, restarted by kubelet if it crashes. At Pod startup, the plugin enumerates GPUs (via NVML or direct ioctl, depending on version) to register them with kubelet's device-plugin gRPC interface.

If a GPU is in a lost / persistent-killed state at Pod-start time, the enumeration may:

1. **Crash** if the plugin treats GPU-init failure as fatal (older plugin versions did this; newer ones are more tolerant).
2. **Hang** if the plugin's enumeration path waits on a wedged GPU (e.g. calls NVML's `nvmlDeviceGetCount()` which may itself wait on the dead device).
3. **Loop** if the plugin retries enumeration on partial failure (some versions retry indefinitely, treating "no GPUs found" as transient).

Kubelet's response to a crashed/hung/looping device-plugin Pod is restart per the Pod's restart policy (default `Always`). The restart re-runs the enumeration, hits the same failure, restarts again — DaemonSet respawn loop.

The visible cluster-layer symptoms:

- `kubectl get pods -n kube-system | grep nvidia-device-plugin` shows the Pod in `CrashLoopBackOff` or `Running` with frequent restarts visible in `kubectl describe`.
- Node's `nvidia.com/gpu` capacity is 0 (no GPUs advertised because the plugin keeps dying).
- Workloads requesting GPUs cannot be scheduled to the node.
- Other healthy GPUs on the same node (if any — multi-GPU systems) are also unavailable because the plugin can't register them either (the enumeration trip on the dead one kills the whole Pod).

This is structurally distinct from F36 (consumer Pods holding a dead GPU — F36 is workload-side; F37 is plugin-side). F36 and F37 commonly occur together in real failures: a GPU goes lost → existing workload Pod (F36) gets stuck → device-plugin Pod restarts and crashes on the dead GPU (F37) → all GPUs on the node go unavailable.

## Trigger sequence

**Cluster-layer only.** Reproducing F37 requires:

1. A real k8s cluster with NVIDIA Device Plugin DaemonSet deployed.
2. A node with at least one GPU (the substrate-presented one or a real card).
3. The GPU in lost / persistent-killed state at the time the Device Plugin Pod is restarted (either initial deployment OR a Pod restart for any reason).

The substrate provides the lost-state via surprise-removal A; the cluster-layer behaviour requires the cluster.

1. Cluster running with healthy GPU; Device Plugin Pod running; GPU advertised as schedulable.
2. Substrate fires surprise-removal A. Cascade fires; A3 marks persistent-killed; A4's telemetry surface reflects the state.
3. Trigger a Device Plugin Pod restart (kubelet restart, Pod delete, node reboot — any event that causes the Pod to re-init).
4. Device Plugin Pod restarts; enumeration encounters the persistent-killed GPU.
5. Plugin behaviour depends on version:
   - Older plugin: crashes on enumeration failure → respawn loop.
   - Newer plugin: enumerates surviving GPUs only → registers them; the dead GPU is silently dropped from the advertised resources.

The F37-significant case is the older / less-tolerant plugin behaviour — and even newer plugins can hit pathological cases if the enumeration HANGS (rather than failing cleanly).

## Expected post-patch behaviour

**The injector cannot fix the cluster-layer issue.** What it provides:

1. **A4 telemetry surface** (same as F36) so the plugin can consume the "this GPU is persistent-killed" signal and skip it cleanly during enumeration.
2. **Fast-fail driver responses** for queries against dead GPUs — the C5 v4 funnels ensure NVML calls (and lower-level ioctls) return errors quickly rather than hanging. This prevents the plugin's enumeration from hanging on the dead device.
3. **Persistent-kill marker** (A3 + A4) — the plugin can read the marker and skip the device from enumeration entirely, rather than attempting to query it.

The F37-specific expected behaviour from the injector's side: any plugin-style enumeration loop that calls NVML / nvidia-smi / direct ioctls MUST receive a fast error (≤ 500 ms) on the dead device, never a hang.

## Assertion shape

**Fast-fail driver-query verification (in scope for fake-5090):**

- After substrate fires surprise-removal A and the cascade completes, the test harness simulates a plugin-style enumeration loop:
  - Call `nvmlDeviceGetCount()` — MUST return a count that excludes the dead device, within ≤ 500 ms.
  - Call `nvmlDeviceGetHandleByIndex(N)` for the surviving devices — MUST succeed for each, within ≤ 100 ms each.
  - Call `nvmlDeviceGetHandleByIndex(N_DEAD)` for the dead device — MUST fail fast (NVML error), within ≤ 100 ms.
  - Call `nvmlDeviceGetUUID(handle_DEAD)` (if the plugin still has a stale handle) — MUST fail fast, within ≤ 100 ms.
- `nvidia-smi -L` MUST complete within ≤ 500 ms.

**Multi-GPU isolation (cross-references F19):**

- Two-device substrate: GPU-A dead, GPU-B healthy.
- Plugin-style enumeration MUST register GPU-B successfully.
- The presence of GPU-A in dead state MUST NOT cause GPU-B to fail to register.
- (Pre-F1: F19's contamination may break this; post-F1: clean.)

**Cluster-layer behaviour assertions (real-cluster-only):**

- Run on a real k8s cluster with NVIDIA Device Plugin deployed.
- Fire surprise-removal A; restart the Device Plugin Pod.
- Plugin SHOULD register surviving GPUs within ≤ 60 s.
- Pod SHOULD NOT enter CrashLoopBackOff.
- Node's `nvidia.com/gpu` capacity SHOULD reflect the surviving GPU count.

**Negative — telemetry consumption is plugin responsibility:**

- The substrate-side fast-fail behaviour is necessary but not sufficient for F37 mitigation. A plugin that doesn't read the persistent-kill marker may still attempt to query the dead device via stale handles or via PCI-rescan; the plugin-layer responsibility is documented but not asserted by fake-5090.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25-F28/F32-F34.
- **NVML / driver-query fast-fail observability.** The substrate must support the cascade-class teardown completing such that subsequent driver queries (via NVML or direct ioctl) fail fast. This is already the C5 v4 contract; F37's assertion shape exercises the contract from a plugin-simulating angle.
- **Multi-device topology (cross-reference F19).** F37's multi-GPU isolation assertion requires two-device substrate setup.
- **Cluster-layer testing (out of scope).** Same as F36 — the actual k8s behaviour requires a cluster.

## Source citations

- NVIDIA k8s device-plugin source — enumeration paths (`pkg/nvml/nvml.go` and equivalents)
- `consumer-holders-and-teardown-future-work.md` (project-internal)
- Community bug reports of device-plugin CrashLoopBackOff on dead GPUs
- C5 v4 design — fast-fail contract for queries against sink-state-set devices
- A3 / A4 intent docs — persistent-kill marker + telemetry surface
- F36 entry (consumer-side cluster failure — workload Pod analog)
- F19 entry (cross-GPU contamination — directly relevant to multi-GPU plugin isolation)
