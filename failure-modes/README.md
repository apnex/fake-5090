# Failure-Mode Index — nvidia-driver-injector patch set

**Date:** 2026-05-27
**Patch set:** `nvidia-driver-injector` C1–C5, E1, A1–A5
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
| Total failure modes | 14 |
| Reproducible in `fake-5090` (v1 substrate sufficient) | 11 |
| Real-hardware-only (route to `scripts/aer-harness`) | 1 (F14) |
| Not-reproducible-by-design (telemetry-presence asserted instead) | 2 (F11, F13) |
| Confidence: field-bug | 10 |
| Confidence: hypothesis | 3 |
| Confidence: doc-only (misattributed but documented) | 1 |

## Index

| ID | Failure (driver perspective) | Source patches | Confidence | Predecessor evidence | fake-5090 mechanism |
|----|------------------------------|----------------|------------|----------------------|---------------------|
| [F1](F01-single-read-commit.md) | Single-read commit to permanent GPU-lost (issue #979 / Mode B silent freeze) | C3, C5 | field-bug | aorus-5090 PM #2 | Config-space transient blip injection |
| [F2](F02-no-error-handlers.md) | `pci_error_handlers` absent → `pcie_do_recovery` aborts | C4, A3 | hypothesis | aorus-5090 Lever M-base patch 0007 | AER fatal injection + dispatch trace |
| [F3](F03-cleanup-asserts-panic.md) | Cleanup-path asserts panic on lost GPU (crash dump / RPC / resserv) | C5 | field-bug | aorus-5090 Lever J-2 0002–0004, N 0006, O 0008 | Surprise-removal A (DeviceLost/0xFFFFFFFF) |
| [F4](F04-dead-bus-mmio-sentinel.md) | Dead-bus MMIO readers loop returning 0xFFFFFFFF | C5, A2 | field-bug | aorus-5090 Lever I patch 0001 | BAR-MMIO state machine + surprise-removal A |
| [F5](F05-modeb-dma-silent-freeze.md) | Mode B DMA-path silent freeze not reachable by passive polling | A2 | hypothesis | aorus-5090 Lever Q 0010–0015 | BAR-MMIO state machine (watchdog-detected transition) |
| [F6](F06-wpr2-stuck-cold-cold-boot.md) | WPR2 stuck after `rm_init_adapter` failure (cold-cold-boot wedge) | A3 | field-bug | aorus-5090 PM #1; Lever M-recover 0024–0028 | BAR-MMIO state (WPR2 stuck-up) + surprise-removal B/C |
| [F7](F07-tb4-misclassified-internal.md) | TB4/USB4 GPU misclassified as internal | E1 | field-bug | aorus-5090 iommu-gsp-lockdown-analysis | Topology modelling (`pci_is_thunderbolt_attached`, untrusted) |
| [F8](F08-aer-internal-mask-silent-drop.md) | AER Internal-Error mask demotes Internal errors to silent-drop | C2 | hypothesis | aorus-5090 G3-H UncMaskClear | Config-space (writable UncMask register) |
| [F9](F09-gpulost-marker-propagation-gap.md) | `osHandleGpuLost` marker-propagation gap (RM set, Linux not) | C5, A2 | field-bug | MISSION-1 E07 Run 2 (2026-05-26) | Surprise-removal D (μs A→B race) |
| [F10](F10-recovery-storm.md) | Recovery storm — 21 attempts in 9 min, no cap/rate-limit/persistent kill | A3 | field-bug | aorus-5090 PM #8; patches 0024+0026+0027+0028 (H1/H2/H3/H4) | AER sustained fault injection (exercise H1 cap + H3 persistence) |
| [F11](F11-observer-induced-failure.md) | Observability tool triggers the failure it monitors (nvidia-smi 17 s cycle) | A2, A4 | field-bug | aorus-5090 PM #11 | *(not reproducible — assert sysfs surface exists)* |
| [F12](F12-dev-nvidia0-close-wedge.md) | `/dev/nvidia0` close-path wedge (Problem 2) | A4 | field-bug | aorus-5090 PM #3; patch 0029 | BAR-MMIO state (WPR2 captured on LAST-CLOSE) |
| [F13](F13-uvm-close-misattributed.md) | UVM close-path wedge (Problem 4 — misattributed; benign) | A4 | doc-only | aorus-5090 PM #4 (admitted misattribution) | *(not reproducible — assert UVM marker emission)* |
| [F14](F14-iommu-gen3-gen4-gsp-lockdown.md) | IOMMU + Gen3↔Gen4 retraining → GSP_LOCKDOWN cascade | *(no injector patch; cmdline+Lever T)* | field-bug | aorus-5090 PM #6, PM #7, PM #9; Lever T cmdline; bridge LnkCtl2 bit 5 | *(real-HW only — Barlow Ridge bridge)* |

## File schema

Every `F<NN>-<slug>.md` has:

1. **Header** — name, sources, confidence, predecessor.
2. **Symptom (driver view)** — what the driver sees, concretely.
3. **Trigger sequence** — numbered actions the test (or daemon) performs.
4. **Expected post-patch behaviour** — what the injector patch makes the driver do instead.
5. **Assertion shape** — exact `dmesg` patterns, `/sys/...` paths, uevent strings, or sentinel reads a test author can check.
6. **fake-5090 mechanism mapping** — which Phase-1 / Phase-2 capability is exercised; "Real-HW only" if not.
7. **Source citations** — intent-doc §-numbers + predecessor cross-refs.

## Patches → failures map (reverse index)

| Patch | F-entries it defends |
|---|---|
| C2 aer-internal-unmask | F8 |
| C3 gpu-lost-retry | F1 |
| C4 err-handlers-scaffold | F2 |
| C5 crash-safety | F1, F3, F4, F9 |
| E1 egpu-detection | F7 |
| A1 pcie-primitives | *(substrate — F2, F4, F6, F12 cite indirectly)* |
| A2 bus-loss-watchdog | F4, F5, F9, F11 |
| A3 recovery | F2, F6, F10 |
| A4 close-path-telemetry | F11, F12, F13 |
| A5 version-and-toggles | *(substrate — observability surface for A2/A3/A4)* |

## Conventions used in per-entry files

- BDF placeholders rendered as `<BDF>` (e.g. `0000:08:00.0` on the real eGPU).
- All telemetry strings are quoted verbatim from intent docs so test authors can `grep` for them without paraphrase drift.
- "*(field-bug)*" tag means an actual observed incident; "*(hypothesis)*" means design-time hardening with no observed incident yet; "*(doc-only)*" means documented in the intent docs as benign-but-instrumented.
