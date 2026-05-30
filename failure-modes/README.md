# Failure-Mode Index — nvidia-driver-injector patch set

**Date:** 2026-05-27
**Patch set:** `nvidia-driver-injector` C1–C5, E1, A1–A8 *(A6/A7/A8 = the F40b bounded-wait + observability family, added after the 2026-05-27 index date)*
**Source of truth:** `docs/patch-intents/*.md` in the injector repo (GIVEN/WHEN/THEN scenarios are authoritative; raw patches are sanity-check material only).
**Purpose:** Test specification for `fake-5090`. Each F-entry describes a failure mode the injector patches defend against, expressed from the *driver's* perspective, with the concrete `fake-5090` mechanism that reproduces it and the concrete assertion shape a test author can implement.

## Scope rules

- One file per failure mode under `failure-modes/F<NN>-<slug>.md`.
- Excluded from F-rows by design:
  - **A1** (pcie-primitives) — pure substrate; helpers are cited as *Supporting patches* in other rows.
  - **A5** (version-and-toggles) — sysfs toggles only, no failure mode of its own.
  - **C1** (kbuild-version-mk) — build infra.

## Summary

| Metric | Count |
|---|---|
| Total failure modes | 42 |
| Reproducible in `fake-5090` (v1 substrate sufficient) | 27 |
| Reproducible in `fake-5090` (v1+ substrate with chip-state + close-path extensions) | 1 (F40 + F41 as a pair; v1 base + chip-ReBAR-CTRL substrate state + close-path-step substrate state) |
| Reproducible with Phase 3 multi-device substrate | 2 (F19, F37) |
| Real-hardware-only (route to `scripts/aer-harness` or apnex rig) | 5 (F14, F21, F29, F30, F31, F35) |
| Cluster-layer-only (route to real k8s cluster) | 2 (F36, F37) |
| Not-reproducible-by-design (telemetry-presence asserted instead) | 2 (F11, F13) |
| Documentation-class entries (no runtime assertion) | 1 (F38) |
| Confidence: field-bug | 21 |
| Confidence: hypothesis | 12 |
| Confidence: community-evidence | 8 |
| Confidence: doc-only (misattributed but documented) | 1 |

## Index

### Layer 1 — Driver-internal cascade (RM core)

| ID | Failure (driver perspective) | Source patches | Confidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|---------------------|
| [F1](F01-single-read-commit.md) | Single-read commit to permanent GPU-lost (issue #979 / Mode B silent freeze) | C3, C5 | field-bug | Config-space transient blip injection |
| [F2](F02-no-error-handlers.md) | `pci_error_handlers` absent → `pcie_do_recovery` aborts | C4, A3 | hypothesis | AER fatal injection + dispatch trace |
| [F3](F03-cleanup-asserts-panic.md) | Cleanup-path asserts panic on lost GPU (crash dump / RPC / resserv) | C5 | field-bug | Surprise-removal A (DeviceLost/0xFFFFFFFF) |
| [F4](F04-dead-bus-mmio-sentinel.md) | Dead-bus MMIO readers loop returning 0xFFFFFFFF | C5, A2 | field-bug | BAR-MMIO state machine + surprise-removal A |
| [F8](F08-aer-internal-mask-silent-drop.md) | AER Internal-Error mask demotes Internal errors to silent-drop | C2 | hypothesis | Config-space (writable UncMask register) |
| [F9](F09-gpulost-marker-propagation-gap.md) | `osHandleGpuLost` marker-propagation gap (RM set, Linux not) | C5, A2 | field-bug | Surprise-removal D (μs A→B race) |
| [F15](F15-assert-or-gpu-lost-under-applied.md) | `NV_ASSERT_OR_GPU_LOST` macro defined but under-applied at 8 teardown sites | C5 (v3+v4) | field-bug | Surprise-removal A + source-audit gate |
| [F16](F16-issue-rpc-and-wait-large-bypass.md) | `_issueRpcAndWaitLarge` bypass — multi-chunk Control RPCs skip the GPU_IS_LOST funnel | C5 (v4) | field-bug | Surprise-removal A + synthetic large RPC |
| [F17](F17-kfsp-arithmetic-invariant-dead-bus.md) | kfsp arithmetic invariant on dead-bus reads — funnel sentinel cannot satisfy | C5 (v4 G9) | field-bug | Surprise-removal A + FSP EMEM register backend |
| [F18](F18-osinit-rmshutdown-isexternal-gotcha.md) | `osinit.c:2462` `RmShutdownAdapter` assert + `is_external_gpu \|\|` audit gotcha | C5 (v4) | field-bug | Surprise-removal A + `is_external_gpu` polarity |
| [F24](F24-sanity-check-noise-budget.md) | Sanity-check noise — diagnostic `NV_PRINTF` lines reachable on lost-GPU paths | C5 (v3 sanity-check class) | hypothesis | Surprise-removal A + dmesg-noise budget assertion |
| [F25](F25-dump-rpc-self-dos.md) | Crash-dump RPC self-DoS — `DUMP_PROTOBUF_COMPONENT` fn 78 blocks 5 s | C5 (v4) | field-bug | Surprise-removal A + crash-dump RPC scenario |
| [F28](F28-client-count-warn-cascade.md) | Client-count `WARN_ON` cascade — kernel-side close-path WARN floods on lost GPU | C5 (v4) | hypothesis | Surprise-removal A + active-client FD scenario |
| [F40](F40-rmshutdownadapter-incomplete-init-wedge.md) | `RmShutdownAdapter` destructive-teardown wedge on chip-responsive but incompletely-initialized state | **A6** (open-arm bounded-wait), **A7** (shutdown-arm bounded-wait) — the in-driver F40b fix; C5 (sink); A4 (telemetry that surfaced it) | field-bug (n=2 confirmed; wedge fires AFTER `nvidia_close_callback` returns — printk goes silent ~immediately; user-visible symptom delayed ~5 min) | Chip-state substrate (`userspace-recovered`) + step-level wedge-injection in destructive teardown |
| [F42](F42-leaked-bounded-wait-worker-uaf.md) | Leaked F40b bounded-wait worker executes in freed `nvidia.ko` `.text`/`nvl` when `rmmod`/`unbind` races it → use-after-free | A7 (`flush_work` guard, SH-3); **A6 = gap (no guard)** | hypothesis (source-audit certain; not crash-observed) | Host-kernel lifecycle race: chip-hang substrate + teardown-during-hang + KASAN build |

### Layer 2 — Driver-internal cascade (cross-module)

| ID | Failure (driver perspective) | Source patches | Confidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|---------------------|
| [F19](F19-nvgpuops-reportfatal-system-global.md) | `nvGpuOpsReportFatalError` system-global escalation — cross-GPU Xid 154 contamination | C5 (Follow-up F1 — out of scope for v4 base) | field-bug | Phase 3 multi-device substrate |
| [F26](F26-nvidia-drm-teardown-hang.md) | `nvidia_drm` teardown hang — DRM-subsystem deregistration blocks on dead GPU | A4 (partial); C5 cross-module | hypothesis | Surprise-removal A + DRM-active scenario |
| [F39](F39-xid154-compute-disconnect-race.md) | Xid 154 compute-disconnect race — multi-second window between Xid 79 and Xid 154 | C5 (UVM-side extension) | field-bug | Surprise-removal A + UVM-channel scenario |

### Layer 3 — Firmware / detector-class

| ID | Failure (driver perspective) | Source patches | Confidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|---------------------|
| [F5](F05-modeb-dma-silent-freeze.md) | Mode B DMA-path silent freeze not reachable by passive polling | A2 | hypothesis | BAR-MMIO state machine (watchdog-detected transition) |
| [F20](F20-gsp-heartbeat-rpc-timeout.md) | GSP heartbeat / RPC timeout (Xid 119) — firmware wedged, not bus-dead | C5 (v4 detector class) | field-bug | Wedged-firmware substrate state |
| [F21](F21-gsp-firmware-internal-halt-xid8.md) | GSP firmware-internal halt under sustained load (Xid 8) — in-firmware fault | *(no injector patch)* | community-evidence | **Real-HW only** (firmware-internal scheduler hang) |
| [F22](F22-silent-hard-hang-no-signal.md) | Silent hard hang under sustained inference — no Xid, no AER, no dmesg signal | A2 | community-evidence | Wedge-without-signal substrate state |

### Layer 4 — Recovery / persistence

| ID | Failure (driver perspective) | Source patches | Confidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|---------------------|
| [F6](F06-wpr2-stuck-cold-cold-boot.md) | WPR2 stuck after `rm_init_adapter` failure (cold-cold-boot wedge) | A3 | field-bug | BAR-MMIO state (WPR2 stuck-up) + surprise-removal B/C |
| [F10](F10-recovery-storm.md) | Recovery storm — 21 attempts in 9 min, no cap/rate-limit/persistent kill | A3 | field-bug | AER sustained fault injection (exercise H1+H3 cap) |
| [F23](F23-instant-xid79-cold-boot.md) | Instant Xid 79 on cold-boot — eGPU never reaches usable state | A3 (bounded retry + persistent-kill); C5 (sink) | community-evidence | Topology surprise primitive |
| [F27](F27-reset-required-loop.md) | `Reset Required` loop — `pci_reset_function` succeeds, GPU re-enters lost state | A3 (H1 cap + H3 persistent-kill) | field-bug | Reset-behaviour configuration (R0/R1/R2/R3 states) |
| [F38](F38-recovery-primitive-gap.md) | Recovery-primitive gap — `pci_reset_function` is not always the right primitive | A3 (bounded scope) | hypothesis | **Documentation-class** — no runtime assertion |

### Layer 5 — Bus / transport / hot-plug

| ID | Failure (driver perspective) | Source patches | Confidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|---------------------|
| [F7](F07-tb4-misclassified-internal.md) | TB4/USB4 GPU misclassified as internal | E1 | field-bug | Topology modelling (`pci_is_thunderbolt_attached`, untrusted) |
| [F14](F14-iommu-gen3-gen4-gsp-lockdown.md) | IOMMU + Gen3↔Gen4 retraining → GSP_LOCKDOWN cascade | *(cmdline + Lever T)* | field-bug | **Real-HW only** (Barlow Ridge bridge) |
| [F29](F29-allocator-retry-asymmetry.md) | PCIe allocator retry asymmetry — kernel gives up faster on hot-plug than cold-boot | *(no injector patch — `pci=realloc=on`)* | community-evidence | **Kernel-class only** |
| [F30](F30-bridge-window-stickiness.md) | Bridge-window stickiness — TB downstream-port windows under-sized for hot-plug | *(no injector patch — `pci=hpmemsize=`)* | community-evidence | **Bridge-class only** |
| [F31](F31-boot-time-bar-failure.md) | Boot-time BAR allocation failure — cold-boot resource pressure | *(no injector patch — UEFI settings)* | hypothesis | **Firmware/kernel-class only** |
| [F41](F41-chip-rebar-control-reset-on-tb-hotadd.md) | Chip ReBAR Control register reset on TB hot-add — H1 broken-BAR1 root cause | *(no injector patch — kernel TB hot-add E27 candidate; `tools/fix-bar1.sh` userspace recovery)* | field-bug | Chip-side ReBAR CTRL state + TB tunnel teardown event |
| [F32](F32-bar1-bar3-boundary-dma.md) | BAR1/BAR3 boundary DMA failure — DMA target straddles the device-side BAR boundary | A2 (forward-progress catch); A1 (substrate) | hypothesis | Substrate-side BAR-boundary state |
| [F33](F33-tb-signal-loss-vs-cc-pin.md) | TB signal-loss vs CC-pin disconnect — link-down has two physical causes | A2 (detection); E1 (transport tag); A4 (telemetry) | field-bug | Transport-classification sub-modes |
| [F34](F34-spurious-plug-event-burst.md) | Spurious plug-event burst — TB controller emits rapid plug/unplug from cable noise | A3 (H1 cap); A4 (burst-detection surface) | community-evidence | Hot-plug-event burst primitive |
| [F35](F35-bolt-authorization-gap.md) | `bolt` authorization gap — TB device fails to auto-authorize, GPU never enumerates | *(no injector patch — `boltctl`)* | community-evidence | **User-space-class only** |

### Layer 6 — Userland / observability / cluster

| ID | Failure (driver perspective) | Source patches | Confidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|---------------------|
| [F11](F11-observer-induced-failure.md) | Observability tool triggers the failure it monitors (nvidia-smi 17 s cycle) | A2, A4 | field-bug | *(not reproducible — assert sysfs surface exists)* |
| [F12](F12-dev-nvidia0-close-wedge.md) | `/dev/nvidia0` close-path wedge (Problem 2) | A4 | field-bug | BAR-MMIO state (WPR2 captured on LAST-CLOSE) |
| [F13](F13-uvm-close-misattributed.md) | UVM close-path wedge (Problem 4 — misattributed; benign) | A4 | doc-only | *(not reproducible — assert UVM marker emission)* |
| [F36](F36-cluster-consumer-holders.md) | Cluster-side consumer holders — Pod referencing dead GPU prevents node drain | *(no injector patch — telemetry-surface dependency)* | hypothesis | **Cluster-layer only** + A4 surface |
| [F37](F37-daemonset-respawn-loop.md) | DaemonSet respawn loop — Device Plugin Pod crash-restart-loops on dead GPU | *(no injector patch — fast-fail driver-query contract)* | hypothesis | **Cluster-layer only** + Phase 3 multi-device |

## File schema

Every `F<NN>-<slug>.md` has:

1. **Header** — name, sources, confidence, predecessor.
2. **Symptom (driver view)** — what the driver sees, concretely.
3. **Trigger sequence** — numbered actions the test (or daemon) performs.
4. **Expected post-patch behaviour** — what the injector patch makes the driver do instead.
5. **Assertion shape** — exact `dmesg` patterns, `/sys/...` paths, uevent strings, or sentinel reads a test author can check.
6. **fake-5090 mechanism mapping** — which Phase-1 / Phase-2 / Phase-3 capability is exercised; "Real-HW only" / "Kernel-class only" / "Cluster-layer only" / "Documentation-class" if not.
7. **Source citations** — intent-doc §-numbers + predecessor cross-refs + community-issue references.

## Patches → failures map (reverse index)

| Patch | F-entries it defends |
|---|---|
| C2 aer-internal-unmask | F8 |
| C3 gpu-lost-retry | F1 |
| C4 err-handlers-scaffold | F2 |
| C5 crash-safety (v3+v4) | F1, F3, F4, F9, F15, F16, F17, F18, F19 (Follow-up F1), F20, F24, F25, F26 (partial), F28, F39 (UVM extension), F40 (out-of-scope coverage class — documented boundary) |
| E1 egpu-detection | F7, F33 (transport tag) |
| A1 pcie-primitives | *(substrate — F2, F4, F6, F12, F32 cite indirectly)* |
| A2 bus-loss-watchdog | F4, F5, F9, F11, F22, F32, F33 |
| A3 recovery | F2, F6, F10, F23, F27, F34, F38 |
| A4 close-path-telemetry | F11, F12, F13, F26 (partial), F33 (disconnect-reason), F34 (burst-detection), F36 (cluster surface), F37 (cluster surface), F40 (close-path bounding telemetry that surfaced the wedge) |
| A5 version-and-toggles | *(substrate — observability surface for A2/A3/A4)* |
| A6 f40b-bounded-wait-open | F40 **open arm** (`RmInitAdapter` GSP-lockdown wedge — confirmed mechanism: `kgspBootstrap_GH100 → gpuTimeoutCondWait(_kgspLockdownReleasedOrFmcError)`, the GSP never releases lockdown on a userspace-recovered chip). Bounds the wait → deterministic `-EIO`. **Coverage boundary (patch property, NOT a failure mode):** does NOT cover the *first* open of a bind — its gate `nv->is_external_gpu` is set lazily inside that open's `RmInitAdapter` (`osinit.c:1301`); a re-probe/rebind onto a bad chip makes the wedge the unguarded first open. **Also relates to F42:** the leaked open-path worker has no `flush_work` guard → UAF if rebind/rmmod races it; the A9 first-open fix broadens that window. |
| A7 f40b-bounded-wait-shutdown | F40 **shutdown arm** (`rm_shutdown_adapter` ~600 ms GSP-unload RPC — not a hang). Same `is_external_gpu` first-open coverage boundary as A6. **Defends [F42]** — the SH-3 `flush_work` guard joins the leaked shutdown worker before teardown, preventing the leaked-worker UAF (A6 has no such guard). |
| A8 f40b-sysfs-observability | F40 (observability surface, not a defense): `tb_egpu_state`, `tb_egpu_is_external`, `tb_egpu_f40b_fires`, recovery counters. `tb_egpu_is_external` makes the A6/A7 first-open coverage boundary observable. |

## Patches → out-of-scope cases (operator/cluster/kernel paths)

| Failure mode | Required mitigation pathway |
|---|---|
| F14 | Kernel cmdline `iommu=off` + `aorus-egpu-bridge-link-cap` service (LnkCtl2 bit 5 cap) |
| F21 | A3-extension (future work) or real-hardware soak testing; no current injector lever |
| F29 | Kernel cmdline `pci=realloc=on` |
| F30 | Kernel cmdline `pci=hpmemsize=64G` or per-board `setpci`-driven bridge-window pre-sizing |
| F31 | UEFI settings (Above 4G Decoding, Resizable BAR Support) or kernel cmdline `pci=use_crs` / `pci=nocrs` |
| F35 | `boltctl enroll --policy=auto <UUID>` or `boltd` policy configuration |
| F36 | NVIDIA Device Plugin cluster-aware enhancement (consume A4 telemetry surface) |
| F37 | NVIDIA Device Plugin fast-fail enhancement (consume A4 persistent-kill marker) |
| F38 | Operator-driven `setpci` SBR-toggle for bus-level reset (per-board recipe) |
| F40 | Persistence-mode engagement immediately after `modprobe nvidia` (verified n=2 2026-05-28); injector container entrypoint + `tools/fix-bar1.sh --bind` already do this. **Driver-side fix landed (A6/A7 = F40b):** bounded-wait wrappers on both arms → deterministic `-EIO` instead of wedge (open-arm mechanism confirmed = GSP lockdown-release wait). **Remaining coverage boundary:** A6/A7 gate on the lazily-set `is_external_gpu`, so they do NOT guard the *first* open of a bind — a re-probe/rebind onto a bad chip still wedges; fix candidate = classify at probe time. Persistence-mode engagement stays the operational mitigation (keeps `usage_count>0` so LAST-CLOSE never enters the wedge-prone re-init). |
| F41 | Userspace recovery via `tools/fix-bar1.sh` (write chip ReBAR Control to max + pciehp slot cycle; verified n=2 2026-05-28) until kernel E27 patch lands on TB hot-add code path. Operator alternative: avoid TB tunnel teardown events (no cable yank, no chassis power-cycle, no boltctl deauth) and reboot-recovery if one occurs. |

## Conventions used in per-entry files

- BDF placeholders rendered as `<BDF>` (e.g. `0000:08:00.0` on the real eGPU).
- All telemetry strings are quoted verbatim from intent docs so test authors can `grep` for them without paraphrase drift.
- "*(field-bug)*" tag means an actual observed incident.
- "*(hypothesis)*" means design-time hardening with no observed incident yet.
- "*(community-evidence)*" means observed and reported in NVIDIA's public issue tracker but not (yet) reproduced on the apnex rig.
- "*(doc-only)*" means documented in the intent docs as benign-but-instrumented.

## Confidence-tier rationale

The 4-tier classification distinguishes:

- **field-bug** — direct observation in our test rig / production. Highest weight in patch-defence ranking.
- **hypothesis** — design-time hardening based on source-audit reasoning; the failure mode is structurally reachable but not observed. Medium weight; patch defends pre-emptively.
- **community-evidence** — tracker-level evidence (NVIDIA open-gpu-kernel-modules issues #461, #888, #900, #916, #979, #1045, #1111, #1134, #1151, #1159) reproducing or describing the failure on hardware/configurations we don't currently test. Cited in upstream-PR rationale and for triage cross-reference; not first-party reproducible.
- **doc-only** — documented in patch-intent corpus as benign-but-instrumented (e.g. F13's UVM close-path was misattributed; the documentation is the artefact of value).
