# F46 — H-OA2 un-bounded first-open init wedge (the un-armed twin of A6's bounded H-OA1)

> **Related:** [[F40]] (the open-arm GSP-lockdown stall this bounds), [[F44]]/[[F45]] (the wedge
> class), [[F42]] (the bounded-wait worker UAF lesson). **Fix:** A12 complete GSP-bootstrap funnel
> (design-of-record `nvidia-driver-injector/docs/missions/.../design/A12-init-funnel-design-of-record-2026-06-04.md`).

| | |
|---|---|
| **Class** | Host wedge — un-bounded chip-touching GSP-bootstrap init |
| **Path** | hot-plug / hot-power-on (recovery-race + every bring-up) |
| **Status** | **OPEN** (in-driver gap; design-of-record approved 2026-06-04, A12 pending implementation) |
| **Attribution** | #282 (open-arm), #301 (this catalog entry) |
| **Discovered** | 2026-06-04, by the A10-v2 fastfail validation + the cold-init budget finding |

## Mechanism

Lazy init (`NVreg_GpuInitOnProbe=0`) defers the ~1.3–1.9 s cold `RmInitAdapter`/GSP bootstrap to the
first device open. On a #979-divergent chip that init can stall indefinitely holding `nvl->ldata_lock`
→ silent host wedge (the F40/F44/F45 class). **A6 (`A6-f40b-bounded-wait-open`) bounds only ONE entry
path** — H-OA1, the `/dev/nvidia0` foreground open (`nvidia_open` → `nv_open_device_for_nvlfp_bounded`,
nv.c:1947). The **same cold init has multiple OTHER entry points that run it UN-bounded** ("H-OA2"):

- `nvidia_dev_get` (nv.c:5416) — modeset `.open_gpu` (nv-modeset-interface.c:156), dmabuf, P2P;
- `nvidia_dev_get_uuid` (nv.c:5530) — UVM (nv_uvm_interface.c:151), P2P;
- the O_NONBLOCK deferred path (`nvidia_open_deferred`, nv.c:1792);
- `nv_pci_probe` (nv-pci.c:2239, latent under `GpuInitOnProbe=1`);
- **and a second RM bootstrap family entirely** — power-resume (`rm_power_management(RESUME)` /
  `rm_transition_dynamic_power`), which re-runs GSP bootstrap *without* transiting `nv_start_device`.

These all call `nv_open_device`/`nv_start_device` → `rm_init_adapter` (or the resume bootstrap) on the
caller's thread **holding `ldata_lock`, with no worker, no timeout, no C5 sink**. The
persistence-engage open (`nvidia-smi -pm 1`) at injector bring-up traverses H-OA2 — so the path that
runs the *actual* cold bootstrap at every bring-up is the **un-armed** one. **A stall on H-OA2 is an
unbounded host wedge.**

## Why it is latent in production

Production cold inits via H-OA2 normally *succeed* (~1.3–1.9 s), so the gap is masked. It bites when
the chip is divergent — a hot-replug recovery, a broken-state re-init, an H16 cold-bringup transient,
or a CUDA/modeset consumer racing the injector — exactly the hot-plug/hot-power scenarios this project
targets.

## Provably-closed entry set (the fix's completeness proof)

`kgspBootstrap_HAL` has **exactly two call sites in all of RM** (`kernel_gsp.c:4798` cold,
`gpu_suspend.c:250` resume) → two bootstrap families, **no hidden entry**. The fix (A12) bounds the
closed set with 3 bound points: the `nv_start_device` funnel (Family-1, all 5 limbs) + 2 resume-site
wraps (Family-2).

## Structural-closure verdict (4-pass adversarial design, 2026-06-04)

- The host **wedge is perfectly closeable** in-driver (A12 complete funnel — every bootstrap entry
  bounded to a finite, host-alive outcome).
- **Instant** termination of a genuinely-stuck non-lockdown stall is **NOT** achievable in-driver —
  proven, three fundamental closed-RM reasons: ① the dead-bus marker self-terminates only the
  lockdown poll; ② resume holds the global RM API lock across its poll (no relaxed-locking release);
  ③ a timeout is not proof of loss (a detached worker can run to success → false-dead-bus a healthy
  GPU). So A6's `flush_work`/A10-v2's grace are **load-bearing**, and the sole residual is a **bounded
  recovery latency** (≤ RM `gpuTimeout`), not a wedge — fixable only **upstream in NVIDIA RM**.

## fake-5090 test contract

Substrate must model an init that **stalls** when reached via a NON-`/dev/nvidia0` entry
(`nvidia_dev_get`/`_uuid`, a resume) — verifying A12 bounds each to `-EIO`/host-alive, not a wedge.
Pairs with the [[F44]] lockdown-poll substrate (#290).
