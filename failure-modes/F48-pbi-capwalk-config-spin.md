# F48 — Probe-time PBI capability-walk infinite spin on a disconnected pci_dev

| Field | Value |
|---|---|
| **Status** | OPEN (discovered live 2026-06-13, apnex.32 campaign cycle-3; host-survivable) |
| **Severity** | device-loss + unkillable modprobe (1-CPU spin, device lock held) → reboot to clear; NOT a host wedge |
| **Class** | vendor latent bug (C6-class primitive defect); third I/O class (PCI CONFIG SPACE) |
| **Provenance** | field-bug; captured live via sysrq-l NMI backtrace (`osPciReadByte ← pciPbiReadUuid ← RmGetGpuUuidRaw ← nv_pci_probe`) |

## Mechanism
`src/nvidia/src/kernel/platform/chipset/pci_pbi.c:85-89`: the PBI capability-list walk
`while (cap_base != 0 && pciPbiCheck(...) != NV_OK) cap_base = osPciReadByte(handle, cap_base+1);`
On a `pci_dev_is_disconnected` device every config read returns `0xFF` → `cap_base=0xFF` forever (≠0, never
valid) → unbounded busy-walk, no delay, no TTL, no signal check. The kernel's own `__pci_find_next_cap_ttl`
bounds the walk (TTL=48) and treats 0xFF as terminator; this vendor loop does neither.

## Reachability
Requires probing a **stale marked-disconnected pci_dev** (error_state persists across driver rebind):
e.g. A13's AER early-free marks the device during a failed open, then a modprobe/bind re-probes WITHOUT
re-enumeration. fix-bar1's slot-cycle always creates a fresh pci_dev → historically unreachable; became
reachable once the #292 containment stack (A12/A13/C7) survives long enough to get there. Coverage gap
class: CONFIG-SPACE reads (`osPciRead*`) are covered by NEITHER the os.c MMIO dead-bus short-circuit NOR
C7's poll-reader table (which scoped re-open MMIO/sysmem polls) — cross-ref [[F47]] (the marker/poll
coverage-gap family), [[F04]] (dead-bus sentinel), [[F09]].

## Fix directions (not yet built)
1. TTL-bound the walk + treat 0xFF as terminator in `pciPbiGetCapability` (upstream-worthy primitive fix).
2. `nv_pci_probe` early-bail on `pci_dev_is_disconnected` (kernel-open guard).
3. Consider an `osPciRead*` dead-bus short-circuit mirroring the MMIO one (audit blast radius first —
   config reads run at probe before any marker can exist).

## Full forensics
`nvidia-driver-injector/docs/missions/mission-1-egpu-hot-plug-hot-power/finding-2026-06-13-F48-pbi-capwalk-spin.md`
