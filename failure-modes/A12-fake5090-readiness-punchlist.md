# fake-5090 readiness punch-list — A12 funnel + the F40–F46 wedge family (2026-06-04)

**Purpose:** make F40–F46 *#290-ready* — i.e., contain everything needed to build the vfio-user
substrate scenarios and write the A12-funnel tests **from the docs alone**. Feeds the v5 strategic
review (fake-5090 build is deferred until after it).

**Audit verdict (7-agent, 2026-06-04):** **NOT ready.** All five A12-relevant docs scored
`testing_complete=false`. Three corpus-wide gaps each independently block the goal (§A), plus per-doc
items (§B) and stale/contradictory sections to fix (§C). §D separates the *doc* gaps (this list) from
the *build* gaps (the daemon itself — #290 proper).

---

## A. Corpus-wide additions (shared — no single doc has these; needed across F40–F46)

- [ ] **A1 — Lockdown-poll register model (THE #1 blocker).** A vfio-user substrate fakes at the
  *register* level, but no doc gives it. Add (canonical home: F40, cross-ref from F44/F46):
  - the **BAR0 MMIO offset** the GSP lockdown poll reads (the HWCFG2/MAILBOX0 register behind
    `gpuTimeoutCondWait(_kgspLockdownReleasedOrFmcError)`, `kernel_gsp_gh100.c:~1050`);
  - the **bit encoding** for *still-locked* vs *released* vs *FMC-error* (so the substrate can answer
    "answers but never releases lockdown, no FMC error");
  - the concrete **`NV_GPU_BUS_DEAD_VALUE_U32` sentinel** (`0xFFFFFFFF`) that `osDevReadReg032`/
    `osIsGpuBusDead` returns once `os_pci_set_disconnected` is set — this is the value the substrate
    must return after the marker fires so the worker self-terminates.
- [ ] **A2 — Per-entry isolation trigger recipes (THE #2 blocker — the live-test blocker).** The docs
  enumerate the entries with source lines but give **no** substrate/guest-side recipe to drive each in
  isolation. Specify, per entry, the guest-side trigger:
  - **H-OA1** foreground — `exec </dev/nvidia0` (exists, rung-a10v2 model).
  - **deferred** — `O_NONBLOCK` open of `/dev/nvidia0` (fires on the `open_q` kthread).
  - **`nvidia_dev_get`** (nv.c:5416) — a **modeset** `.open_gpu` op (load `nvidia-modeset` + trigger)
    or a small in-guest kernel shim calling the jump-table entry.
  - **`nvidia_dev_get_uuid`** (nv.c:5530) — a **CUDA UVM** managed-memory op (drives `nvidia-uvm` →
    `nv_uvm_interface.c:151`), or a kernel shim.
  - **`nv_pci_probe`** (nv-pci.c:2239) — bind/unbind, or `NVreg_GpuInitOnProbe=1`.
  - **Family-2 resume** — `echo mem >/sys/power/state` (system-resume → `rm_power_management(RESUME)`)
    and an RTD3/GC6 idle→wake cycle (`rm_transition_dynamic_power`).
- [ ] **A3 — Family-2 resume substrate model (THE #3 blocker — the funnel's novel surface).** No doc
  models the resume GSP bootstrap (`gpu_suspend.c:250` via `gpuPowerManagementResume`). Add: which
  register/poll the resume bootstrap reads, how it stalls on the substrate, the expected
  `-EIO`-equivalent the resume path returns (and whether it can surface an errno to a caller), and what
  "host-alive after a bounded resume stall" looks like. (Belongs in F46 + a new resume sub-entry, or
  its own F-entry.)
- [ ] **A4 — Canonical pass/fail oracle (absent from ALL of F40/F44/F45/F46).** Pin the machine-checkable
  signals every test asserts:
  - **config-vendor** `0x10de` (bus alive) vs `0xffff` (sunk) — the rung-a10v2 authoritative probe;
  - **`tb_egpu_state`** enum: `lost-temporary` = OK (recoverable), `lost-permanent`/`disconnected` =
    fail;
  - **host-liveness** wall-clock probe (no wedge within N s);
  - the **funnel dmesg signature** (now `tb_egpu [F40b/A12]: ...` after the relabel — quote the exact
    strings: `open completed within budget`, `fast-fail, chip NOT sunk`, `GSP lockdown poll ... dead-bus
    marker + sink`).
- [ ] **A5 — Shared substrate-numerics reference.** Centralize the passive-read identity values (today
  only in F40, cross-ref'd inconsistently): `PMC_BOOT_0=0x1b2000a1`, `0x110094` canary, **ReBAR cap @
  ext-offset `0x13c`**, `WPR2` = `0x07f4a000` (set) / `0x00000000` (cleared), AER status, config-vendor.
  Make F40 the canonical home and have F44/F45/F46/F42 reference it explicitly (DRY is fine *if* the
  referenced doc actually carries the value).
- [ ] **A6 — Write the per-entry trigger YAMLs.** F40–F46 have **zero** `triggers/F4x.yaml` (F01–F39 all
  follow the `trigger.primitive/args/preconditions/config/assertions` schema). Author one per failure
  mode, binding the §A4 oracle into `assertions` and the §A1/A3 substrate state into `config:` knobs.

---

## B. Per-doc punch-list

### F46 — H-OA2 (the most incomplete; it's currently a 4-line contract)
- [ ] Add a self-contained (or explicitly-referenced) substrate-state block (→ A5/A1).
- [ ] Add the **per-entry test matrix** with the §A2 triggers (it enumerates entries but gives no how).
- [ ] Add the **Family-2 resume substrate model** (→ A3) — its unique surface.
- [ ] Add an **Assertion-shape section** (Bug-present / Fix-verified / Observability) + the §A4 oracle.
- [ ] Add a **regression-guard arm** (healthy cold init / dGPU still succeeds; WPR2-fast-fail twin still
  fast-fails) — present in F44/F45, absent here.
- [ ] Restate the A12 SHALL chain mechanically (marker → `osIsGpuBusDead` → self-terminate → `flush_work`
  → C5 sink → `-EIO`) instead of only pointing at the design-of-record.

### F40 — lockdown arm (good prose, stale substrate sections)
- [ ] Add the **lockdown-poll register model** (→ A1) — it names only the C function today.
- [ ] Add the **A12/A10 funnel SHALL-behavior** for the arm under test (currently only A6 predecessor
  behavior is concrete).
- [ ] Add the §A4 oracle (config-vendor; `tb_egpu_state` temporary-vs-permanent; the discriminator
  decision logic for slow-healthy vs stuck — named but not specified).
- [ ] **STALE — see §C1/§C2.**

### F44 — A10 lock-inversion (strongest on *behavior*; thin on numerics/oracle)
- [ ] Restate or firmly cross-ref the passive-read numerics + the **value HWCFG2/MAILBOX0 returns during
  the stuck poll** + the `NV_GPU_BUS_DEAD_VALUE_U32` value (→ A1/A5).
- [ ] Add the **AER/rmmod second-contender timing recipe** (how to land the contender *inside* the stuck
  `gpuTimeoutCondWait` window — this distinguishes instant-wedge from finite-soft-block).
- [ ] Add the §A4 config-vendor + `tb_egpu_state`-enum oracle (it has the latency/return oracle only).

### F45 — A11 cold-init deadlock (strong behavior; needs deterministic injection)
- [ ] Add a **deterministic GSP-heartbeat-fail injection hook** (today it's a stochastic "injected PCIe
  transient" — specify how the substrate forces `SET_GUEST_SYSTEM_INFO failed 0xf` reproducibly).
- [ ] Add the **concurrent-writer scheduling** mechanism (how the test deterministically holds
  `g_RmApiLock` when the deferred-open worker reaches `rmapiLockAcquire`, osapi.c:2745).
- [ ] Add the **residual-discriminator drgn expression** (rwsem owner/count for "permanently held" vs
  transient) + a concrete `complete_all`-never-fires programmatic check.
- [ ] Add the §A4 oracle + an A12-funnel bridge (it covers A11+C6 only; connect to the C5-sink/`-EIO`
  arm + funnel observability).

### F42 — leaked-worker UAF (good UAF oracle; predates A12)
- [ ] **Update to the SHIPPED A12 funnel** (→ §C3): it describes the A6 fix as "queued (v5) / make the
  worker provably self-terminating" — the shipped reality is the funnel's *kept* flush-join + dead-bus
  marker.
- [ ] Add the **race-window scheduling** mechanism (how the test makes rmmod/unbind land inside the hang
  window deterministically — the worker reliably still-in-flight at teardown).
- [ ] Parameterize the "hang the RM call" requirement (which poll never completes; the RM `gpuTimeout`
  bound) + add the §A4 oracle (config-vendor / host-not-wedged / `-EIO`).

---

## C. Stale / contradictory fixes (do regardless of #290)

- [ ] **C1 — F40's superseded substrate model.** The labeled "fake-5090 mechanism" row (line 35) +
  "fake-5090 mechanism mapping" (lines 568–579) + the Assertion-shape 30s-wedge oracle (lines 416–443)
  all model the **retracted CTO/destructive-teardown-withhold** chip ("does not respond to the chip-init
  TLP → PCIe Completion Timeout") — directly contradicting F40's own current line-65 understanding ("no
  CTO; the chip *answers* the lockdown poll but never releases"). Rewrite these to the lockdown busy-poll
  model so a builder doesn't synthesize the wrong chip.
- [ ] **C2 — A8 counters "broken" note (F40, F42).** Both cite the A8 sysfs counters as runtime-broken
  (the old `dev_groups` no-op). A8 migrated to `sysfs_create_group` (#273, A8 v2). Update the
  observability notes to the shipped v2 attribute paths.
- [ ] **C3 — F42 predates A12.** Re-point its "A6 fix queued for the v5 review" framing to the shipped
  A12 funnel (kept `flush_work` join + dead-bus self-termination).

---

## D. Build gaps (the substrate itself — #290 proper, NOT this doc list; flagged so scope is honest)

Even with §A–§C done, the **testing tool does not exist yet**. fake-5090 is design-only (README/SURVEY;
"no code yet"). #290 must still build:
- the **vfio-user daemon** (Rust crate, main loop, vfio-user message handlers, config-space/cap-chain/
  BAR-MMIO model) + vendored **libvfio-user** + bindgen FFI;
- the **scenario-DSL loader** (parse the trigger YAMLs; bind `config:` knobs to daemon registers);
- the **QEMU guest harness** (launch script, guest image, QMP `device_del` for surprise-removal, fast
  reboot/snapshot) + the **assertion engine** (parse dmesg/sysfs, evaluate timing);
- the **captured real-5090 `lspci -vvxxx` ground-truth** config-space fixture (SURVEY Phase 0);
- resolution of the two MVP blockers in SURVEY (does the stock driver accept the chosen generic device
  ID; does E1 key on a TB VSEC the QEMU x3130 model doesn't advertise).

---

## E. Suggested sequencing for the v5 review
1. **§C** stale-fixes first (cheap; removes the actively-misleading F40 CTO section).
2. **§A1 + §A5** — the register model + numerics (unblocks the substrate chip-state; mostly recoverable
   from the open-driver source + a real-5090 register dump).
3. **§A2 + §A4** — the per-entry triggers + oracle (the live-test blocker; defines the test surface).
4. **§A3** — the resume substrate model (the genuinely-novel research bit).
5. **§A6 + §B** — author the trigger YAMLs + bring each doc to the F01–F39 standard.
6. Then **§D** (#290 build) has a complete spec to execute against.
