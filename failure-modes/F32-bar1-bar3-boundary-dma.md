# F32 — BAR1/BAR3 boundary DMA failure — DMA target straddles the device-side BAR boundary

| Field | Value |
|---|---|
| **Sources** | A2 (forward-progress watchdog detects the resulting hang) — partial coverage; A1 (pcie-primitives) provides the substrate-side surface |
| **Confidence** | hypothesis |
| **Predecessor evidence** | Aorus 5090 BAR-layout reverse-engineering notes; cascade-scope-audit §"BAR1/BAR3 boundary considerations" |
| **fake-5090 mechanism** | Substrate-side BAR-boundary state: substrate exposes BAR1 (32 GB) and BAR3 (32 MB) at adjacent or specified addresses; DMA-target injection at the boundary produces device-side completion failures observable as forward-progress stall |

## Symptom (driver view)

The 5090 (and similar Blackwell/Ada parts) exposes:
- BAR0 (256 MB): MMIO register space.
- BAR1 (32 GB): system-memory aperture / VRAM aperture.
- BAR3 (32 MB): small secondary aperture (typically GSP-internal or specialised use).

When the kernel-side PCIe allocator assigns these BARs, BAR1 and BAR3 may end up adjacent in physical address space (depending on bridge-window layout). The device-side DMA engine has internal logic that decodes incoming DMA addresses to determine which BAR's aperture they target; if a DMA descriptor specifies a target address that crosses the BAR1/BAR3 boundary (either due to a multi-page DMA crossing the boundary, or a software bug that points a descriptor at an invalid address near the boundary), the device-side decoder produces an internal error that is NOT signalled as a host-visible AER fault — it is logged in device-internal counters and the DMA silently fails or returns garbage.

The driver sees no AER signal. The driver sees no Xid (the DMA engine's internal error counter is not wired to a host-side error path for this class). The driver sees the application-visible symptom: a memory transfer that doesn't complete, a kernel that runs but produces wrong output, or a compute task that hangs waiting for a write that never arrives.

This is structurally similar to F22 (silent hard hang) in that it presents as "device is alive but workload hangs"; the difference is the mechanism — F32 is a device-side decoder bug triggered by a specific DMA address pattern; F22 is the broader silent-wedge class with multiple possible causes.

The A2 forward-progress watchdog catches F32 by the same mechanism it catches F22: the completion counter stops advancing; after the configured threshold, the watchdog declares the GPU lost and the cascade-class teardown runs. F32's distinct identity is in the test methodology — a specific DMA-pattern injection scenario — and the device-side counter that the substrate must expose to make the failure verifiable.

## Trigger sequence

1. Daemon brings device to a healthy bound state with BAR1 and BAR3 at substrate-configured adjacent addresses (the test harness verifies via `lspci -vvv -s <BDF>` that the kernel did indeed lay them out adjacently — if not, the test is no-op'd with a "BAR layout doesn't trigger F32 in this run" message).
2. Test issues a DMA operation whose target address is at the BAR1/BAR3 boundary. Concrete shapes:
   - A multi-page DMA whose first page is in BAR1's aperture and last page crosses into BAR3's aperture.
   - A single-page DMA targeted exactly at the last page of BAR1 (the descriptor may straddle if the device reads beyond the descriptor's claimed extent due to prefetch).
3. Substrate's DMA engine processes the descriptor. In F32 mode, the substrate detects the boundary-crossing and:
   - Does NOT raise an AER signal.
   - Does NOT update the engine's completion counter.
   - DOES increment a device-internal error counter (substrate-exposed for test observability).
4. Application waits for the DMA completion that never arrives. After A2's watchdog threshold, sink-state is set; cascade runs.

## Expected post-patch behaviour

A2's forward-progress watchdog catches the stall and triggers sink-state via `cleanupGpuLostStateAtomic`. The cascade-class teardown runs cleanly (per F22's assertion shape). The application receives an error on the next driver call rather than hanging indefinitely.

For F32 specifically (additional to F22's general handling): the device-internal error counter, if exposed by the substrate, is read out post-teardown and surfaced in A4's telemetry as "BAR-boundary DMA error count" — operators triaging F32 vs. F22 can read this counter to distinguish causes. The injector itself does NOT fix F32 (the fix is either device-firmware or DMA-allocator-side to avoid the boundary); the injector's value is bounded failure handling.

## Assertion shape

**Forward-progress detection (same as F22):**

- A2's Q-watchdog MUST detect the stall within bounded time (≤ 5 × `NV_TB_EGPU_BUS_LOSS_WATCHDOG_HZ` × 1000 ms = 1250 ms at 4 Hz default).
- `dmesg | grep -cE 'tb_egpu_qwd.*dead bus detected'` (or canonical Q-watchdog log) MUST be ≥ 1.

**Device-internal counter verification (F32-specific):**

- Substrate exposes a "BAR-boundary DMA error count" counter via a designated debug register.
- Test harness reads the counter pre-DMA, post-DMA, and post-cascade.
- Pre-DMA: counter == 0.
- Post-DMA (before A2 fires): counter == 1 (the boundary-crossing DMA registered the error).
- Post-cascade: counter unchanged (no further DMAs were attempted after sink set).

**Distinction from F22 (operator triage):**

- F22 scenario: counter remains 0 throughout (no boundary-crossing involved).
- F32 scenario: counter increments by 1 per boundary-crossing DMA.
- A4's telemetry exposes the counter; operator triage workflow checks the counter to distinguish F22 (generic silent wedge) from F32 (specific boundary-crossing).

**Negative — F32 reproducibility requires the right BAR layout:**

- If `lspci -vvv -s <BDF>` shows BAR1 and BAR3 NON-adjacent, the F32 test is a no-op. Test framework reports "skipped: BAR layout does not satisfy F32 precondition" rather than failing.
- The test framework retries with different `pci=hpmemsize=` configurations to maximise adjacent-layout coverage.

**Bounded teardown:**

- Total wall-clock from DMA submission to host settled in lost-GPU state MUST be ≤ 5 s (Q-watchdog detection + sink propagation + cleanup).

## fake-5090 mechanism mapping

- **Phase 2 substrate-side BAR-boundary state.** The substrate exposes BAR1 (32 GB) and BAR3 (32 MB) per the 5090's actual BAR layout. When configured in "F32 mode," the substrate's DMA-handler logic specifically detects boundary-crossing descriptors and produces the silent-failure behaviour.
- **Forward-progress counter (same as F22).** The substrate exposes a completion-counter that the A2 watchdog polls; in F32 mode, the counter freezes when a boundary-crossing DMA is submitted.
- **Device-internal error counter (F32-specific).** The substrate exposes a separate counter that increments when boundary-crossing DMAs are detected. This counter is the diagnostic differentiator between F22 and F32.
- **BAR-layout introspection.** The test harness queries `lspci` to confirm BAR1/BAR3 adjacency before running the F32 test; if non-adjacent, the test is no-op'd.

## Source citations

- Aorus 5090 BAR-layout reverse-engineering notes (project-internal; predecessor to apnex/fake-5090)
- `cascade-scope-audit.md` §"BAR1/BAR3 boundary considerations" — preliminary classification
- A2 intent doc — Q-watchdog forward-progress mechanism
- A4 intent doc — telemetry-surface for device-internal counters
- F22 entry (silent hard hang — F32's sibling, broader class)
- F05 entry (Mode B DMA-path silent freeze — earliest known forward-progress-class failure)
