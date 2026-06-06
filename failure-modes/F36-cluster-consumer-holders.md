# F36 — Cluster-side consumer holders — Pod referencing dead GPU prevents node drain

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — k8s/cluster-layer concern; documented as adjacent failure mode that the injector's telemetry surface enables observing)* |
| **Confidence** | hypothesis (design-time analysis surfaced during cluster-integration planning; documented in `consumer-holders-and-teardown-future-work.md`) |
| **Predecessor evidence** | `consumer-holders-and-teardown-future-work.md` (project-internal architecture doc); kubernetes device-plugin lifecycle documentation |
| **fake-5090 mechanism** | **Cluster-layer only — telemetry-surface dependency.** Substrate provides the GPU's lost-state observability via A4's sysfs surface; the cluster-layer logic (kubelet device-plugin re-registration, NVIDIA Device Plugin Pod liveness probes) consumes the surface; F36 is the case where the cluster layer doesn't consume it correctly |

## Symptom (driver view)

When a GPU enters lost-state (cascade fires, A3 marks persistent-killed), the kernel-side device is in a known-dead state. From the driver's view, the cascade-class teardown has completed cleanly; A4's telemetry exposes "this GPU is persistent-killed."

From the **cluster's** view, the situation is more complex:

1. **NVIDIA Device Plugin Pod** (the kubelet device-plugin DaemonSet) registers GPUs as schedulable resources with kubelet. If the Pod is not aware that the GPU has gone persistent-killed, kubelet continues to advertise it as available. New workloads may get scheduled onto the dead GPU and fail to start.
2. **Workload Pods currently holding the dead GPU** (via `nvidia.com/gpu` resource request → kubelet → device-plugin) are not automatically evicted. They remain `Running` from kubelet's view; their containers may be stuck in CUDA-init failure loops or hung waiting for the dead device.
3. **Node drain** (`kubectl drain <node>`) waits for Pods to gracefully terminate. Pods holding the dead GPU may not respond to termination (the container is wedged on the dead device, not the workload itself); drain hangs.
4. **Workload re-scheduling** to other nodes / other GPUs is blocked on the consumer Pod actually terminating — which requires either operator intervention (`--force` drain), the Pod's grace-period expiring (typically 30s, but can be longer), or the consumer process inside the Pod actually exiting (which won't happen if it's wedged on the dead GPU).

F36 is the class of failures arising from this gap. The injector cannot fix it — the resolution is at the cluster layer (device-plugin updates, kubelet integration, GPU operator's NodeFeatureDiscovery refinement). The injector's contribution is **exposing the lost-state telemetry** that the cluster layer needs to consume.

This is structurally distinct from F37 (DaemonSet respawn — a related cluster-class failure where the device-plugin Pod itself restarts repeatedly because of the dead GPU). F36 is consumer-side; F37 is plugin-side.

## Trigger sequence

**Cluster-layer only.** Reproducing F36 requires a real k8s cluster with NVIDIA Device Plugin DaemonSet deployed. The substrate-side reproduction is partial — fake-5090 can provide the GPU lost-state telemetry that the cluster layer should consume, but the cluster-layer behaviour requires the cluster itself.

1. K8s cluster running on a node with the GPU in a healthy state. NVIDIA Device Plugin Pod is running and has registered the GPU as a schedulable resource.
2. Workload Pod is scheduled to the node, requests `nvidia.com/gpu: 1`, is allocated the GPU, and is `Running`.
3. Substrate fires surprise-removal A. Driver-side cascade fires; A3 marks persistent-killed; A4's telemetry surface reflects the state.
4. Cluster-layer behaviour observed:
   - Device Plugin Pod: does it re-register the GPU as unavailable? (Currently: NO, unless the Pod has explicit logic to consume A4's surface.)
   - Workload Pod: does it get evicted? (Currently: NO, kubelet doesn't see the GPU as gone.)
   - Node drain: does it succeed? (Currently: NO if the workload is wedged on the dead device.)

## Expected post-patch behaviour

**The injector cannot fix the cluster-layer issue.** What it provides is the telemetry surface that a cluster-aware patch can consume. Specific expected behaviours when the full stack is configured:

1. **A4 surfaces** `disconnect-reason`, `persistent-kill-state`, and `last-state-change-timestamp` per device via stable sysfs paths.
2. **NVIDIA Device Plugin Pod** (or a custom watchdog DaemonSet) consumes the A4 surface, detects `persistent-kill-state == true`, and:
   - Re-registers the GPU with kubelet as unavailable (kubelet updates Node resource).
   - Optionally: signals the consumer Pod (via the device-plugin's PreStopHook or via a sidecar) to terminate.
3. **Workload Pod's failure-handling** (workload-author responsibility) — CUDA init failure or runtime CUDA error should propagate to Pod restart-policy; the Pod should fail-fast on the dead device.

F36's specific contribution to the injector's failure index is: documenting that A4's telemetry surface MUST be designed for cluster-layer consumption (stable paths, parseable values, no transient-only states).

## Assertion shape

**A4 telemetry-surface stability assertions (in scope for fake-5090):**

- After substrate fires surprise-removal A and the cascade completes, the following sysfs paths MUST exist with stable, parseable values:
  - `/sys/bus/pci/devices/<BDF>/<A4-persistent-kill-path>` — boolean true/false.
  - `/sys/bus/pci/devices/<BDF>/<A4-last-state-change-timestamp-path>` — Unix timestamp.
  - `/sys/bus/pci/devices/<BDF>/<A4-disconnect-reason-path>` — enum (signal_loss / cc_pin / cold_boot / etc.).
- The paths MUST be readable by a non-root user (cluster pods typically run as non-root).
- The values MUST persist across the time-window relevant to cluster reconciliation (typically: at least 60 s; the values represent the LAST observed state, not transient).

**Cluster-layer behaviour assertions (real-cluster-only):**

- Run on a real k8s cluster with NVIDIA Device Plugin deployed and a custom watchdog DaemonSet that consumes A4's surface.
- Fire surprise-removal A (via substrate OR via real-hardware cable yank).
- Device Plugin SHOULD re-register GPU as unavailable within ≤ 30 s.
- Workload Pod SHOULD enter a failure state (CrashLoopBackOff or Pending re-schedule) within ≤ 60 s.
- `kubectl drain <node> --grace-period=60` SHOULD succeed within bounded time (≤ 120 s).

**Negative — F36 fix requires both layers:**

- Without the cluster-layer watchdog consuming A4's surface, the cluster-layer assertions WILL fail. This is expected — F36 explicitly documents that the injector provides half the solution (telemetry surface); cluster-layer code provides the other half.

## fake-5090 mechanism mapping

- **Phase 2 surprise-removal A primitive** — same as F03/F04/F15-F18/F25-F28/F32-F34.
- **A4 telemetry-surface dependency.** F36's value depends entirely on A4 exposing stable cluster-consumable sysfs paths. The substrate's role is providing the lost-state condition; A4's role is exposing it; the cluster watchdog's role is consuming it.
- **Cluster-layer testing (out of scope for substrate).** The full F36 scenario requires a real k8s cluster. The substrate-side test asserts only the telemetry-surface contract; cluster-side behaviour is asserted in a separate cluster-integration test suite (not in fake-5090 v1).

## Source citations

- `consumer-holders-and-teardown-future-work.md` (project-internal architecture doc)
- Kubernetes documentation: device-plugin lifecycle, kubelet resource advertising
- NVIDIA k8s device-plugin documentation
- A4-close-path-telemetry intent doc — telemetry-surface design (must support cluster consumption)
- F37 entry (DaemonSet respawn — sibling cluster-class failure on the plugin side)
- F23/F27 entries (the A3 persistent-kill states that F36 surfaces to cluster consumers)
