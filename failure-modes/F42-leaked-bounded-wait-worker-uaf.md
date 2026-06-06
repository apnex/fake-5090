# F42 — leaked bounded-wait worker UAF (worker outlives module/device teardown)

**Sources (defense):** A7 (f40b-bounded-wait-shutdown — the SH-3 `flush_work` guard). **Coverage gap:** A6 (open path — no guard; see reverse-map).
**Confidence:** hypothesis (no crash observed), but **source-CONFIRMED** on both paths by two independent audits — SH-3 4-agent gate (shutdown path) + the R0 correctness audit 2026-05-31 (open path): the A6 open worker writes `nvlfp`/`sp` after `nvidia_open`'s `failed:` path frees them (nv.c), a UAF reachable with **zero operator action**, *widened* by A9 (every first-open now dispatches a leakable worker). **FIXED on the open path by R0** (`flush_work` leak→join, apnex.25 — mirrors A7's SH-3 guard; A6 was the documented gap, now closed). The 2026-05-29 20:52 wedge was *not* this (A7 not in that build).
**Predecessor / related:** [F40](F40-reinit-gsp-lockdown-wedge.md) (the open-arm wedge **A6's** worker bounds) + [F43](F43-gsp-unload-rpc-latency-vs-naive-budget.md) (the shutdown-path RPC latency **A7's** worker bounds) — F42 is a lifecycle bug *in that bounding mechanism*, reachable on either path; [F03](F03-cleanup-asserts-panic.md) / [F26](F26-nvidia-drm-teardown-hang.md) (teardown-path memory-safety family).

## Symptom (driver view)

The F40b bounded-wait wrappers (A6 open-path, A7 shutdown-path) run the chip-touching RM call on a **worker queued to the shared `system_long_wq`**, and the syscall/teardown thread returns on timeout **while the worker is still running** ("leaked", by design — refcount-2 "last-one-frees" protocol). The leaked worker holds **no reference on `nvidia.ko`**. If module-unload or device-remove runs before the worker finishes, the worker's own code (`nvidia.ko` `.text`) and the `nvl`/`nv` state it dereferences are **freed underneath it → use-after-free** in a kernel workqueue context.

It is a **double-UAF**:
1. leaked worker on a *shared* workqueue with no module ref — `try_module_get` is ineffective because the worker is queued *after* `delete_module`'s refcount gate;
2. `nv_pci_remove_helper` frees `.text`/`nvl` while the worker still executes.

## Trigger sequence

1. An F40b wrapper fires (open *or* shutdown timeout) → schedules the chip-touching RM call onto `system_long_wq` (refcount-2 heap work struct) + `wait_for_completion_timeout`.
2. The timeout elapses → the wrapper returns (`-EIO` for open / proceeds for shutdown) **while the worker is still inside the chip-touching RM call** (GSP busy-poll / non-completing MMIO).
3. `rmmod nvidia` (`free_module`) **or** `unbind` (`nv_pci_remove_helper`) runs before the worker completes.
4. The worker dereferences freed `nvidia.ko` `.text` / `nvl` → **UAF** (kernel `.text`/heap freed beneath a running worker → oops / KASAN report / silent corruption).

(Concrete substrate: the F40 `userspace-recovered` bad chip makes the open-arm worker busy-poll for a long/unbounded time, widening step-2's window; a re-probe path — manual rebind, A3 `slot_reset`, hot-plug — then races `unbind`/`rmmod`.)

## Expected post-patch behaviour

**A7 (shutdown path):** the SH-3 `flush_work(&w->work)` guard in the timeout branch **joins the worker before the teardown path proceeds** → the worker is finished before `.text`/`nvl` are freed → no UAF. (Safe because the shutdown worker provably returns in ~600 ms.)

**A6 (open path) — coverage gap, intentional:** A6 has **no** flush guard; it leaves the open-path worker leaked, because the open worker's exit is **unverified** (it may busy-poll for the RM-internal multi-second timeout, or longer), so a naive `flush_work` could itself hang the teardown. The correct A6 fix is to make the worker **provably self-terminating** (consult the C5 sink / force the MMIO to fail) so it can be safely joined — the A6 "leak→join lifecycle" hardening (queued, v5 strategic review). The A9 probe-time-classification fix *broadens* this gap by arming A6 on re-probe first-opens.

## Assertion shape

- **Bug present:** an `rmmod`/`unbind` during an in-flight F40b worker → KASAN use-after-free (or oops) in `system_long_wq` worker context; dmesg shows `tb_egpu [F40b]: ... scheduled to bounded worker` **without** a matching `completed/timed out` line *and* the worker still running at teardown.
- **Fix verified (A7):** the teardown blocks until the worker returns (the flush guard joins it); no UAF; `nvidia.ko` not freed until the worker exits. SH-3 validated this with a forced 100 ms close-path timeout (worker joined, no zombie).
- **Observability:** `tb_egpu_f40b_fires` increments per fire (a fire ⇒ a leaked worker existed); `tb_egpu_is_external` (A8 v2.2) reveals whether A6/A7 were even armed.

## fake-5090 mechanism mapping

A **host-kernel lifecycle race** (workqueue worker vs `free_module` / `nv_pci_remove`), **not** a chip-state failure — the chip's only role is to make the RM call hang long enough to keep the worker in-flight. Reproduce by: a substrate state that **hangs the chip-touching RM call** (so the F40b worker stays running) **+ an injected `rmmod`/`unbind` during the hang**, built **under KASAN** to catch the freed-`.text`/`nvl` dereference. Requires the F40b bounded-wait worker mechanism present (A6/A7). Phase: v1+ substrate (chip-hang state) + a teardown-during-hang step + KASAN build. Discriminator: does teardown **join** the worker (A7, guarded) or free memory under it (A6, unguarded)?

## Source citations

- SH-3 finding (the latent double-UAF + the `flush_work` guard): `nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/shutdown-hang-ledger.md`; A7 timeout-branch guard `kernel-open/nvidia/nv.c` + `docs/patch-intents/A7-f40b-bounded-wait-shutdown.md`.
- A6 gap + the A9 broadening: `docs/superpowers/specs/2026-05-31-a9-egpu-probe-classify-design.md` (Residual risks), `docs/architecture-v5-deep-review-queued.md` (A6 leak→join follow-up).
- Worker mechanism: A6/A7 intent docs; `nv_f40b_*_work` in `kernel-open/nvidia/nv.c`.
