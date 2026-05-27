# fake-5090 — Survey

**Date:** 2026-05-26
**Author:** hermes (agent)
**Question asked:** *Is there a way to mock/emulate a PCIe-attached
eGPU device purely for driver code-path testing (no working CUDA
needed), that appears as a real PCIe GPU like a 5090?*

**Short answer:** Yes — and the host (`obpc`) already has the
foundational pieces installed. Recommended path: **vfio-user fake
GPU daemon + QEMU/KVM guest running the driver-under-test**,
complemented by **`aer-inject` against a real device** for end-to-end
error-path validation when an eGPU is attached.

---

## 1. Approaches surveyed

Five options, ordered from highest fidelity / highest effort to
lowest. Reasoning given for each pick or reject.

### 1.1 vfio-user daemon — *recommended primary path*

Out-of-process device emulation over a UNIX socket. QEMU is the
client (`-device vfio-user-pci,socket=...`); the fake-5090 daemon
is the server. The daemon owns config space, BAR memory regions,
MSI/MSI-X, and capability chains.

- **Fidelity:** highest of any non-hardware option for the paths we
  care about (enumeration, config space, capability walking, MMIO
  register state machine, MSI delivery, AER-via-config-space).
- **Effort:** medium-high. ~2–4 weeks for a respectable v1.
- **Iteration speed:** good — daemon restarts in milliseconds; only
  the guest reboot is slow (mitigate with kexec or VM snapshots).
- **Crash blast radius:** zero — daemon crash kills the guest at
  worst; host untouched.
- **CI-friendly:** yes. Headless, scriptable, deterministic.
- **Language:** C reference (`libvfio-user`), Rust bindings exist
  (`vfio-user-rs`), Python bindings are thin.

**Why this wins:** the driver code paths of interest sit at the
PCIe-substrate layer (config-space reads, AER masks, link-state,
bus-loss detection, recovery sequencing). That is exactly what
vfio-user models well and exactly what real-hardware testing makes
painful (you can't reliably crash a 5090 on demand).

### 1.2 QEMU built-in custom PCI device (in-tree)

Add a new `hw/misc/fake_nvgpu.c` to QEMU itself; register a
`TypeInfo` with NVIDIA vendor/device IDs and BAR layout.

- **Fidelity:** equivalent to vfio-user in principle.
- **Effort:** similar to vfio-user, but you're now patching QEMU and
  carrying that diff forever.
- **Verdict:** **avoid**. vfio-user gets the same fidelity without
  forking QEMU. Only reason to choose this is if vfio-user transport
  overhead matters, which it won't for driver-init testing.

### 1.3 Kernel-level fake PCI device (no VM)

Register a synthetic `struct pci_dev` directly in the host kernel
via `pci_create_root_bus()` + a stub driver, OR use an out-of-tree
`pci-emu` / `fake_pci_bus` project.

- **Fidelity:** medium — config space is real, MMIO is whatever you
  back it with (vmalloc'd memory typically).
- **Effort:** low for a stub, medium for anything stateful.
- **Iteration speed:** excellent — no VM boot.
- **Crash blast radius:** **high** — a driver oops takes down the
  host. Not acceptable when the code under test is explicitly about
  driver crash safety.
- **Verdict:** good as a *secondary* loop for tight unit tests of
  isolated functions once trusted; **not** the primary harness.

### 1.4 `aer-inject` + real device — *recommended complementary path*

Kernel module that synthesises AER (Advanced Error Reporting) events
against a real PCIe device. Already in the running kernel on `obpc`
(`/lib/modules/7.0.9-204.fc44.x86_64/kernel/drivers/pci/pcie/aer_inject.ko.xz`).
Pair with EINJ (ACPI Error Injection) for platform-level errors and
Thunderbolt `authorized=0` writes for controlled detach.

- **Fidelity:** **highest possible** — real card, real link, real
  driver loaded the real way.
- **Effort:** very low — module + small userspace wrapper.
- **Crash blast radius:** medium — a bad AER injection can wedge the
  device or the root port; host generally survives.
- **Verdict:** **use this today**, against a 5090 eGPU when attached.
  It is the highest-confidence validation available, and it costs
  almost nothing to set up. Belongs in a sibling harness, not
  necessarily inside fake-5090 — but documented here because it's
  the natural Phase 0 of any driver-test programme.

### 1.5 virtio-gpu / VirGL / Venus

Paravirtualised GPU. Guest gets OpenGL/Vulkan but **no NVIDIA driver
loads**, because there's no NVIDIA device present.

- **Verdict:** **out of scope** — wrong shape for this problem.

---

## 2. Host capability check (`obpc`)

Run on 2026-05-26.

| Capability                             | Status                                      |
|----------------------------------------|---------------------------------------------|
| QEMU                                   | `10.2.2-1.fc44` ✓                           |
| `vfio-user-pci` device in QEMU         | **present** ✓                               |
| `pcie-root-port` / `x3130-upstream` / `xio3130-downstream` | all present ✓ (model TB switch hierarchy)   |
| `pci-testdev` / `ivshmem` (references) | present ✓                                   |
| `libvirt-daemon-kvm`                   | `12.0.0-3.fc44` ✓                           |
| `/dev/kvm`                             | present ✓                                   |
| `aer_inject` kernel module             | present in `7.0.9-204.fc44` ✓               |
| Kernel source                          | `/root/linux-v6.19/` available ✓            |
| `libvfio-user` headers/lib             | **not installed** — build from source       |
| Real GPU currently attached            | no (Arc Pro 130T iGPU only)                 |
| Thunderbolt domains                    | `domain0`, `domain1` both present ✓         |
| TB root ports                          | `00:07.0`, `00:07.2` ✓                      |

**Net:** every blocker for the recommended path is already cleared
**except** `libvfio-user` itself, which is a vendored build (Fedora
does not currently package it). No QEMU rebuild required.

---

## 3. Driver code-path test surfaces

fake-5090 is downstream-agnostic, but the immediate consumer is
`apnex/nvidia-driver-injector`. The injector's patch manifest is a
useful concrete spec for what fake-5090 v1 must exercise. The patch
set sits at exactly the layer fake-5090 models well:

| Code-path family       | Modelled by fake-5090 via …                                            |
|------------------------|------------------------------------------------------------------------|
| AER capability handling | Config-space capability block at the correct offset; scripted mask state |
| "Device lost" detection | Daemon flips into "all-ones" mode → config reads return `0xFFFFFFFF` |
| `pci_error_handlers`    | Trigger via AER injection (real device) or simulated config-space state change |
| Crash-safety / teardown | `device_del` via QMP; daemon dies mid-MMIO transaction                |
| eGPU / TB detection     | Model `xio3130-downstream` + `x3130-upstream` chain; optionally advertise TB VSEC in daemon |
| PCIe primitives (read/write/recovery) | Hit via normal driver init; exercised end-to-end           |
| Bus-loss watchdog       | Daemon enters "bus lost" state; verify watchdog fires within SLA      |
| Recovery (reset + re-enum) | vfio-user `DEVICE_RESET` protocol; daemon replays init           |
| Surprise removal        | **Three orthogonal mechanisms** (see §3.1)                            |
| Close-path telemetry    | Mechanism B (graceful hot-remove via QMP `device_del`)                |

Stateful MMIO modelling is the long pole — defer the deepest
recovery-flow modelling to phase 2.

### 3.1 Surprise removal — the highest-value test family

Surprise removal is the marquee scenario fake-5090 exists to
exercise, and it splits into three orthogonal mechanisms. Each
models a different real-world failure shape from the driver's
perspective; each lives behind a control-socket trigger.

**Mechanism A — daemon enters `DeviceLost` state mid-operation.**
The daemon's state machine flips into a "device is gone" mode.
Subsequent config-space and BAR MMIO reads return `0xFFFFFFFF`;
writes are silently dropped (no error returned to QEMU, mirroring
how dead silicon behaves); no MSI is ever raised again. The driver
gets *silence and 0xFF*, which is exactly what a real surprise-
removed device produces until the kernel's PCIe layer notices and
synthesises a remove. This is the truest test of bus-loss watchdog
and gpu-lost-retry paths. **No `.remove()` callback fires** — the
driver must detect the loss itself.

**Mechanism B — graceful PCIe hot-remove via QMP.**
`{"execute":"device_del","arguments":{"id":"fake-5090-leaf"}}` over
the QMP socket. QEMU invokes the orderly PCIe hot-remove protocol;
pciehp fires; the driver's `.remove()` callback runs in the *normal*
code path. Tests close-path telemetry and orderly teardown — a
distinct set of paths from Mechanism A.

**Mechanism C — yank the whole switch hierarchy.**
`device_del` against the root port removes the leaf, both switch
ports, and the GPU together. Models an eGPU enclosure power-cut or
host-side cable yank. Stresses ordering assumptions: does the
driver cope when the parent bridge disappears before the child has
finished cleanup?

**Mechanism D — combined race (A then B, microsecond-timed).**
Daemon enters `DeviceLost`; 50ms later QEMU emits `device_del`.
Races bus-loss-watchdog detection against orderly removal — a known
field-bug shape for eGPUs where the driver enters recovery just as
the OS decides to tear down. Real hardware produces this race
randomly; fake-5090 produces it on every run with deterministic
timing. This is the kind of test only fake-5090 can run.

---

## 4. Recommended build order

### Phase 0 — *do today, no new code* (effort: 1–2 days)

1. Lives in a sibling harness (or a `scripts/aer-harness.sh` here, if
   we want everything under one roof). When the 5090 eGPU is next
   attached, use `aer-inject` + `setpci` + Thunderbolt
   `authorized=0` to validate error-handling code paths against the
   real card. **Highest-confidence validation available without
   building fake-5090.**
2. Capture `lspci -vvxxx` from a real 5090 — this is the ground-truth
   config-space layout the daemon will replay. Can be done on `obpc`
   itself the next time the eGPU is connected.

### Phase 1 — *minimum viable fake* (effort: 1–2 weeks)

1. Build `libvfio-user` from source (vendor under
   `fake-5090/vendor/`).
2. Write a Rust or C daemon (~500 LOC) that:
   - Serves the 5090's config space header (vendor/device/class/BARs)
   - Implements capability chain: PCIe, MSI-X, AER
   - Returns sane defaults for BAR MMIO reads (zeros or scripted
     values from a fixture file)
   - Logs every transaction
3. QEMU guest: Fedora 44, driver-under-test loaded, topology:
   ```
   pcie.0 → pcie-root-port → x3130-upstream → xio3130-downstream → vfio-user-pci(fake-5090)
   ```
4. **Success criterion:** driver `probe()` runs, hits the targeted
   code paths, logs visible in dmesg.

### Phase 2 — *failure injection* (effort: 1–2 weeks)

1. Daemon gains a control socket: external test driver flips states
   ("link down", "bus lost", "all reads return 0xFF", "fatal AER
   pending").
2. Daemon supports vfio-user `DEVICE_RESET` for recovery-path
   testing.
3. Test harness (Python or bash) runs scenarios → asserts on dmesg /
   sysfs / module debugfs.

### Phase 3 — *consumer integration* (effort: 1 week)

1. Publish daemon as a container image (matches the
   `nvidia-driver-injector` containerisation pattern).
2. Document the consumer contract: socket protocol, control-socket
   commands, container env vars, version tag policy.
3. Downstream repos (starting with `nvidia-driver-injector`) wire
   fake-5090 into their `tests/` directories — scenarios and
   assertions live there, not here.

---

## 5. Risks and caveats

- **vfio-user is young.** Protocol is stable but ecosystem is thin.
  C reference (`libvfio-user`) is canonical; Rust bindings best-
  maintained for new daemons.
- **Stateful MMIO modelling has a long tail.** Recovery flows
  require the most modelling. Defer to phase 2; ship phase 1 with a
  static-fixture model first.
- **Residual gaps requiring real hardware.** fake-5090 simulates
  surprise removal directly (see §3.1) — but a thin layer of
  peripheral phenomena that *co-occur* with removal on real silicon
  cannot be modelled in pure software:
  - **LTSSM link-training states.** The Link Training and Status
    State Machine lives in silicon. If a bug depends on observing a
    specific transient LTSSM state during recovery, fake-5090 can't
    reproduce it.
  - **Partial in-flight DMA.** The daemon can drop MMIO writes
    mid-burst, but real hardware may have written *some* bytes to
    host memory before the link dropped. Modellable via vfio-user
    `DMA_MAP` (the daemon has access to guest pages) but non-trivial;
    deferred past phase 2.
  - **Thunderbolt-layer races.** TB controller firmware, USB4 tunnel
    renegotiation, side-band channel. If a bug depends on the TB
    stack racing the PCIe stack *outside the driver*, still needs
    real hardware.
  - **AER signal vs link-drop ordering.** Real silicon non-
    deterministically interleaves fatal AER with link drop; QEMU
    injects deterministically. Good for testing, but doesn't
    naturally reproduce spec-ambiguous orderings.
  These are real gaps. None of them are surprise removal itself —
  they're peripheral phenomena that co-occur with it. `aer-inject`
  against a real card (scripts/, Phase 0) covers what fake-5090
  cannot.
- **NVIDIA's driver checks device IDs at multiple points.** Need to
  confirm a 5090 ID is recognised by the *currently shipping*
  consumer driver. If not, fall back to a 4090 ID. Make this
  configurable from day one.

---

## 6. References

See [`references/`](references/) for:

- `references/vfio-user.md` — protocol summary, key links, library
  pointers, build notes
- `references/aer-inject.md` — usage notes for the in-kernel injector
  and complementary tools (setpci, EINJ, TB authorize)
- `references/qemu-topology.md` — exact QEMU CLI for a TB-style
  switch hierarchy with vfio-user-pci leaf
- `references/patch-surface-map.md` — line-level mapping per
  downstream consumer (TODO, Phase 1, populated by each consumer)

---

## 7. Decisions (locked 2026-05-26)

1. **Repo home.** `apnex/fake-5090` on GitHub, **public**.
2. **License.** MIT.
3. **Default device ID.** Generic Ada/Blackwell-class NVIDIA ID for
   broad driver-version compatibility. Device ID is **configurable
   at runtime** via CLI flag and scenario fixture — daemon is built
   to be reprogrammable to any vendor/device/class triple.
4. **Language.** Rust, with `bindgen`-generated FFI over a vendored
   `libvfio-user`. Trade-off table and rationale in chat transcript;
   short version: memory safety in a test harness is
   non-negotiable, and the orchestration surface area (control
   socket, scenario DSL, CLI, container) dwarfs the vfio-user-facing
   code and is where Rust wins decisively.
5. **`aer-inject` harness location.** Inside this repo under
   `scripts/`. Single home for the spectrum of NVIDIA-driver-test
   tooling, fake and real.
6. **Phase 0 ordering.** Real-hardware baseline stands up **after**
   fake-5090 v1 ships, not before.

## 8. Generated LICENSE

Standard MIT, copyright `apnex`. Will land as [`LICENSE`](LICENSE)
when the repo is initialised.
