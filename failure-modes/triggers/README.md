# Trigger Matrices — `fake-5090` substrate configurations per failure mode

**Date:** 2026-05-27
**Source:** `failure-modes/F<NN>-<slug>.md` entries (one per F-mode).
**Purpose:** Machine-consumable test configurations the `fake-5090` harness loads to drive the substrate into each failure mode and assert the post-patch contract.

One YAML file per F-entry: `triggers/F<NN>.yaml`. The harness discovers them by filename pattern, validates each against the schema below, and routes the substrate-configurable subset into substrate fixtures while parking the rest (real-hw / cluster / doc-only) for operator workflows.

## Schema

```yaml
failure_mode: F<NN>                   # canonical ID matching the .md entry
slug: <kebab-case>                    # matches the .md filename's slug
reproducibility:                      # enum, determines harness routing
  substrate-v1        | # Phase 1 or 2 substrate; runs in CI
  substrate-phase3    | # multi-device or hot-plug-framework; Phase 3 gate
  real-hw-only        | # routes to apnex rig, not CI
  cluster-only        | # routes to k8s integration harness
  telemetry-only      | # asserts surface presence, not failure stimulus
  doc-only              # asserts doc structure, no runtime
substrate_phase: 1 | 2 | 3 | n/a      # numeric phase capability required
preconditions:
  topology:     <single-device | multi-device | tb-attached | pcie-discrete | ...>
  driver_state: <healthy-bound | cold-boot | pre-probe | n/a>
  workload:     <idle | cuda-active | uvm-active | drm-active | nvml-poll | ...>
config:                               # substrate knobs (BAR state, AER mask, ...)
  <key>: <value>
trigger:
  primitive: <surprise-removal-A | aer-fatal-inject | gsp-firmware-wedge | ...>
  args:
    <key>: <value>
assertions:
  driver_observable:                  # dmesg patterns, sysfs paths, uevents
    - <string>
  substrate_observable:               # substrate-side counters, register state
    - <string>
  bounded_timing:                     # explicit ms ceilings
    <metric>_max_ms: <int>
out_of_scope_notes: |                 # only present for non-substrate-v1 classes
  <free text explaining what the substrate CANNOT drive>
source_f_entry: failure-modes/F<NN>-<slug>.md
```

## Quoting convention

Lines containing `:` outside structural use (e.g. dmesg patterns like `NVRM: Xid (PCI:<BDF>): 79`) MUST be single-quoted to prevent YAML parsers (or LSP linters) from misreading them as nested mappings. The convention applies to all `*_observable` list items.

## Substrate primitives — vocabulary

The `trigger.primitive` field references one of the substrate's exposed test primitives. The current set:

| Primitive | Phase | Description |
|---|---|---|
| `config-space-transient-blip` | 1 | Config-space returns sentinel for a bounded window, then recovers |
| `surprise-removal-A` | 2 | Silent dead-bus: MMIO+config return 0xFFFFFFFF, no `.remove()` uevent |
| `surprise-removal-B-or-C` | 2 | Cold-cold-boot scenarios (WPR2 stuck pre-probe) |
| `surprise-removal-D` | 2 | μs-race window between MMIO and config failure (marker propagation test) |
| `aer-fatal-inject` | 2 | Single AER uncorrectable-fatal injection |
| `aer-fatal-sustained` | 2 | AER refires on every recovery attempt (storm test) |
| `aer-internal-error-inject` | 2 | AER Internal-Error class specifically (F8 unmask verification) |
| `bar-mmio-state-transition` | 2 | Transition substrate from healthy MMIO to Mode-B wedge mid-workload |
| `silent-wedge-without-signal` | 2 | DMA wedges; no AER, no Xid (F22 surface) |
| `gsp-firmware-wedge` | 2 | GSP accepts RPC; never responds (F20) |
| `topology-enumeration` | 2 | Driver probe walks the topology to assert classification |
| `topology-surprise-cold-boot-dead` | 2 | Chip presents dead state at cold-boot (F23) |
| `dma-boundary-crossing-descriptor` | 2 | DMA descriptor straddles BAR1/BAR3 boundary (F32) |
| `hot-plug-event-burst` | 3 | Substrate emits N plug/unplug events with configurable cadence (F34) |
| `surprise-removal-A-on-gpu-a-only` | 3 | Multi-device topology; surprise-remove one only (F19, F37) |

Phase 3 primitives require multi-device topology and/or substrate participation in the host's hot-plug enumeration framework.

## Index

### Substrate-reproducible (CI-runnable, Phase ≤ 2)

| ID | Primitive | Predecessor Phase | Notes |
|---|---|---|---|
| [F01](F01.yaml) | config-space-transient-blip | 1 | C3 retry verification |
| [F02](F02.yaml) | aer-fatal-inject | 2 | C4 err-handlers dispatch |
| [F03](F03.yaml) | surprise-removal-A | 2 | Cleanup-asserts panic prevention |
| [F04](F04.yaml) | surprise-removal-A | 2 | Steady-state MMIO sentinel + A2 watchdog |
| [F05](F05.yaml) | bar-mmio-state-transition | 2 | Mode-B silent freeze |
| [F06](F06.yaml) | surprise-removal-B-or-C | 2 | WPR2 stuck cold-cold-boot |
| [F07](F07.yaml) | topology-enumeration | 2 | TB4 classification (E1) |
| [F08](F08.yaml) | aer-internal-error-inject | 2 | C2 unmask verification |
| [F09](F09.yaml) | surprise-removal-D | 2 | Marker propagation race |
| [F10](F10.yaml) | aer-fatal-sustained | 2 | A3 H1/H2/H3/H4 storm gating |
| [F12](F12.yaml) | last-close-with-wpr2-captured | 2 | /dev/nvidia0 close-path |
| [F15](F15.yaml) | surprise-removal-A | 2 | NV_ASSERT_OR_GPU_LOST 8-site coverage |
| [F16](F16.yaml) | surprise-removal-A | 2 | Large-RPC bypass funnel |
| [F17](F17.yaml) | surprise-removal-A | 2 | kfsp arithmetic-invariant funnel |
| [F18](F18.yaml) | surprise-removal-A | 2 | osinit `is_external_gpu` polarity |
| [F20](F20.yaml) | gsp-firmware-wedge | 2 | GSP RPC timeout / Xid 119 |
| [F22](F22.yaml) | silent-wedge-without-signal | 2 | No-Xid no-AER hang |
| [F23](F23.yaml) | topology-surprise-cold-boot-dead | 2 | Instant Xid 79 + persistent-kill |
| [F24](F24.yaml) | surprise-removal-A | 2 | Sanity-check noise budget |
| [F25](F25.yaml) | surprise-removal-A | 2 | Dump-RPC self-DoS funnel |
| [F26](F26.yaml) | surprise-removal-A | 2 | nvidia_drm teardown short-circuit |
| [F27](F27.yaml) | surprise-removal-A | 2 | Reset-state R3 (persistent-dead) |
| [F28](F28.yaml) | surprise-removal-A | 2 | Client-count WARN cascade |
| [F32](F32.yaml) | dma-boundary-crossing-descriptor | 2 | BAR1/BAR3 boundary DMA |
| [F33](F33.yaml) | surprise-removal-A | 2 | TB submodes (signal-loss / cc-pin) |
| [F39](F39.yaml) | surprise-removal-A | 2 | UVM Xid 154 latency |

### Phase 3 (multi-device or hot-plug-framework gate)

| ID | Primitive | Notes |
|---|---|---|
| [F19](F19.yaml) | surprise-removal-A-on-gpu-a-only | Cross-GPU isolation; gates Follow-up F1 |
| [F34](F34.yaml) | hot-plug-event-burst | Substrate emits plug-events through host hot-plug fwk |
| [F37](F37.yaml) | surprise-removal-A-on-gpu-a-only | Multi-GPU plugin-style enumeration |

### Telemetry-presence only (substrate asserts surface, not stimulus)

| ID | Notes |
|---|---|
| [F11](F11.yaml) | Observer-induced failure; A4 surface presence + non-stimulating reads |
| [F13](F13.yaml) | UVM close-path marker emission (benign instrumentation) |

### Real-hardware only (routes to apnex rig)

| ID | Required hardware | Mitigation |
|---|---|---|
| [F14](F14.yaml) | Barlow Ridge + 5090 | iommu=off + bridge-link-cap service |
| [F21](F21.yaml) | Real 5090 + sustained inference | A3-extension future work |
| [F29](F29.yaml) | Real TB host + 5090 | `pci=realloc=on` |
| [F30](F30.yaml) | Under-sized bridge windows + 5090 | `pci=hpmemsize=64G` |
| [F31](F31.yaml) | Tight UEFI resource map + 5090 | UEFI Above-4G / Resizable-BAR |
| [F35](F35.yaml) | boltd + TB-attached 5090 | `boltctl enroll --policy=auto` |

### Cluster only (routes to k8s integration harness)

| ID | Substrate contribution | Cluster contribution |
|---|---|---|
| [F36](F36.yaml) | A4 surface presence + stability assertion | Pod eviction, drain success |
| [F37](F37.yaml) | Fast-fail driver-query + isolation | Plugin Pod CrashLoopBackOff prevention |

(F37 appears in both Phase 3 and Cluster lists — its substrate-side contract is Phase 3, its full cluster contract requires real k8s.)

### Documentation only

| ID | Asserts |
|---|---|
| [F38](F38.yaml) | A3 intent doc + future-work tracker + apnex setup-guide structure |

## Validation

The harness validates each YAML against the schema before loading. To run the validator standalone:

```bash
# (validator script TBD — to live at scripts/validate-trigger-yaml.py)
python scripts/validate-trigger-yaml.py failure-modes/triggers/*.yaml
```

PyYAML safe-load parses all 39 files cleanly today. LSP linters may flag false-positives on lines containing `:` inside double-quoted strings; the single-quote convention above resolves those. The substantive lint signal (yaml_load failures) is clean across the index.

## Cross-references

- Each trigger's `source_f_entry` field links back to the failure-mode definition in `../F<NN>-<slug>.md`.
- The failure-mode definitions' `fake-5090 mechanism mapping` section describes the substrate primitive in prose; this directory crystallises that prose into machine-consumable form.
- The reverse index in `../README.md` maps patches → F-entries; combined with this directory's primitive vocabulary, an operator can trace patch → failure → substrate fixture in two hops.

## Counts

| Category | Count |
|---|---|
| Total trigger files | 39 |
| Substrate-v1 (CI-runnable) | 26 |
| Substrate-Phase-3 (multi-device / hot-plug-fwk) | 3 |
| Telemetry-presence only | 2 |
| Real-hardware only | 6 |
| Cluster only | 2 |
| Documentation only | 1 |

(F37 is counted under both Phase 3 and Cluster; F36 under Cluster; the totals above sum to 40 because of the F37 double-count — there are 39 distinct files.)
