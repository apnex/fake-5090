# fake-5090

> A synthetic PCIe device that lies convincingly enough to a real
> NVIDIA driver that the driver's enumeration, hotplug, and error-
> handling code paths run as if a real RTX 5090 were attached —
> without any GPU hardware, and without risking the host.

## Why this exists

Modern GPU drivers carry a lot of code that nobody can reliably
exercise. Specifically:

- **Hotplug paths** that fire when a Thunderbolt-attached eGPU
  appears or disappears.
- **Error-recovery paths** that fire when the PCIe link drops,
  retrains, throws AER events, or returns all-ones from a dead
  device.
- **Defensive teardown paths** that fire during surprise removal
  mid-DMA.

These paths exist because the failure modes are real — anyone running
an eGPU has watched their host hang at least once — but they are
*structurally hostile to test against real hardware*:

1. You can't reliably crash a 5090 on demand without bricking it.
2. Surprise-removal mid-DMA against a real card is a great way to
   wedge the host you're trying to keep healthy.
3. CI fleets don't have eGPUs; even if they did, every test run
   would burn warranty.
4. The code paths often hide subtle bugs that only surface across
   a specific sequence of link-flap → recovery → re-enumeration —
   sequences that take minutes to set up by hand and seconds to
   destroy.

The result is that error-handling code in GPU drivers is, in
practice, written, shipped, and prayed over. Bugs survive in it for
years because nobody can repeatably trigger them.

fake-5090 is a way out of that. It is a **userspace daemon** that
implements the PCIe-substrate-facing slice of an NVIDIA GPU —
config space, BARs, capability chains, MSI/MSI-X, AER, reset — over
the [vfio-user](https://github.com/nutanix/libvfio-user) protocol.
A QEMU/KVM guest connects to the daemon as if it were a real PCIe
device. The guest runs the driver-under-test. The driver thinks it
has a 5090.

Then we make the daemon misbehave on purpose, in scripted,
reproducible, version-controllable ways, and watch what the driver
does.

## What this is for

- **Validating NVIDIA driver forks** that add error-handling,
  recovery, or eGPU/Thunderbolt-aware logic — most immediately
  `apnex/nvidia-driver-injector`, but the design is downstream-
  agnostic.
- **Regression-testing those patches in CI** so a kernel upgrade or
  driver-version bump doesn't silently regress the error paths.
- **Reproducing field bugs** — capture a real failure scenario as a
  scripted sequence of daemon state transitions; replay it any time.
- **Exploring driver behaviour** under conditions that would
  otherwise require deliberately damaging hardware (link errors,
  partial enumeration, capability corruption).

## What this is *not* for

- Running CUDA. There is no compute model and there never will be.
- Display output, video encode/decode, or any actual GPU
  functionality.
- Replacing real-hardware testing. fake-5090 *complements* tests
  against real hardware (and `aer-inject` against real cards); it
  does not pretend to fully model a 5090. Surprise-removal mid-
  USB4-tunnel-renegotiation will always need a real card and a real
  cable.

## Design principles

1. **Repo-sovereign.** fake-5090 knows nothing about its consumers'
   internals. It publishes a stable interface (socket protocol +
   container image + version tag). Consumers (`nvidia-driver-
   injector` first; others later) own their own scenarios and
   assertions in their own `tests/` directories. No two-way
   coupling.
2. **Zero host blast radius.** Daemon crashes don't take the host
   down. Driver crashes don't take the host down. Only the guest
   VM ever dies, and it dies on demand.
3. **Scriptable failure injection.** Every interesting failure mode
   is a state the daemon can be told to enter from outside, over a
   control channel. No recompilation to trigger a new scenario.
4. **CI-friendly.** Headless, deterministic, containerised. A
   downstream consumer should be able to wire fake-5090 into a
   GitHub Actions job and have it Just Work.
5. **Patch-set agnostic.** fake-5090 doesn't care which NVIDIA
   driver fork, which patch, which version. It models the
   substrate; the driver lives in the guest; the test scenarios live
   in the consumer repo.

## Status

Survey only — no code yet. The decision to build is made; the build
order, language, and packaging questions are still open.

- [`SURVEY.md`](SURVEY.md) — full findings: approaches considered,
  host capability check, recommended build order, risks, open
  questions.
- [`references/`](references/) — background on vfio-user, aer-inject,
  and the QEMU topology needed to model a Thunderbolt-attached eGPU.
- [`notes/`](notes/) — working notes per build phase (empty).
- [`scripts/`](scripts/) — harness scripts (empty).

## Audience

You are likely reading this because you are one of:

- **An NVIDIA driver fork maintainer** who wants to actually
  *test* the error paths in your fork. fake-5090 is for you. Skip
  to SURVEY.md §3 for how patches map to test surfaces, then §4 for
  the recommended build order.
- **A CI engineer** trying to gate driver merges on something other
  than "did it compile." fake-5090 is for you. Skip to SURVEY.md
  §4 Phase 3 for the consumer-integration story.
- **A kernel/PCIe developer** curious whether vfio-user is mature
  enough for non-trivial device emulation. fake-5090 is a worked
  answer in progress. SURVEY.md §1.1 and `references/vfio-user.md`
  have the protocol-level details.
- **A future version of `apnex`** trying to remember why this repo
  exists. Hi. The reason is in the four paragraphs of §1 above.

## Key design choices (locked)

- **License:** MIT.
- **Default device ID:** generic Ada/Blackwell-class NVIDIA ID,
  chosen for broad compatibility with currently-shipping drivers.
  The device ID is configurable from the command line and from the
  scenario fixture file — fake-5090 is built to be reprogrammable
  to any PCI vendor/device/class triple at runtime, not just NVIDIA.
- **Implementation language:** Rust, with `bindgen`-generated FFI
  over a vendored `libvfio-user`. Rationale: memory safety in a
  test harness is non-negotiable (a flaky harness erodes trust in
  real failures), and the orchestration surface area —
  control socket, scenario DSL, CLI, container packaging — dwarfs
  the vfio-user-facing code and is where Rust wins decisively.
- **`aer-inject` real-hardware harness:** lives inside this repo
  under `scripts/`, not as a sibling. Single home for the full
  spectrum of NVIDIA-driver-test tooling, fake and real.
- **Real-hardware baseline:** stood up *after* fake-5090 v1 ships,
  not before.

## License

MIT. See [`LICENSE`](LICENSE).
