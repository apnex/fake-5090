# F35 — `bolt` authorization gap — TB device fails to auto-authorize, GPU never enumerates

| Field | Value |
|---|---|
| **Sources** | *(no injector patch — user-space `boltd` daemon root cause; documented as known-mitigated-elsewhere)* |
| **Confidence** | community-evidence |
| **Predecessor evidence** | Distribution bug reports of TB GPU enumeration depending on `bolt` policy; aorus-5090-egpu setup-guide §`boltctl` configuration; community reports of "eGPU works on user A's host but not user B's" traced to `bolt` policy differences |
| **fake-5090 mechanism** | **User-space-class only.** Substrate cannot reproduce; failure-index entry documents the boundary and operator mitigation |

## Symptom (driver view)

Modern Linux distributions ship `boltd` (the Bolt thunderbolt management daemon) which enforces a per-device authorization policy. By default (since GNOME 3.30+ era), the policy is:

- Devices connected at boot time (with auto-authorize enabled): authorized automatically.
- Devices connected at runtime (hot-plug): require explicit user authorization via `boltctl authorize <UUID>` OR via GNOME/KDE settings UI OR via a `bolt` policy file marking the device as "ALWAYS auto-authorize."

For headless systems (servers, lab rigs, anything without a GUI), the GUI authorization flow is unavailable; if the policy is "user-prompt" and no user is present, the device sits in "unauthorized" state and never reaches PCI enumeration. The driver never sees the device; `nv_pci_probe` is never called; the operator sees `boltctl list` showing the device in "connected, not-authorized" state.

The driver-visible symptom is "GPU connected but not detected" — but the cause is upstream of the driver. `nvidia-smi -L` returns "No devices found." `lspci -nn -d 10de:` returns nothing (the GPU has not been added to the PCI hierarchy because `bolt` has not authorized the underlying TB link). The injector cannot fix this — `bolt` is user-space, runs before PCI enumeration, and the gate is policy-level.

This is structurally distinct from F33 (signal-loss vs CC-pin — F33 is a physical-layer disconnect after enumeration; F35 is an authorization-layer block before enumeration) and from F29/F30/F31 (kernel-side allocation failures — those run AFTER `bolt` authorizes and the kernel attempts PCIe BAR assignment).

## Trigger sequence

**User-space-class only.** Reproducing F35 requires:

1. A real TB/USB4 device (the substrate cannot drive `boltd`'s authorization flow because the substrate's transport-adapter exists post-authorization).
2. A real `boltd` daemon running on the host.
3. A `bolt` policy configured to require explicit authorization (the default on most distributions for hot-plug devices).
4. A hot-plug-attach event with no user present to authorize.

The fake-5090 v1 substrate cannot reproduce F35:
- The substrate sits BELOW `boltd` in the stack. By the time the substrate is in play, the kernel has already accepted the device and is running `nv_pci_probe`. The `boltd` authorization gate is at a higher level.

F35 routes to operator-procedure documentation (the apnex setup guide includes a "set `bolt` policy to auto-authorize" step for new TB attachments).

## Expected post-patch behaviour

**No injector patch addresses F35.** Operator-level mitigations:

1. **`boltctl enroll --policy=auto <UUID>`** — enrol the device with auto-authorize policy. After enrolment, subsequent hot-plug events authorize automatically.
2. **`boltctl --auth-mode=auto`** (varies by `bolt` version) — set the host-global default for new devices to auto-authorize.
3. **`boltd` configuration file** — set the policy via `/etc/bolt/config` or distribution-equivalent.
4. **Disable `boltd`** (`systemctl disable --now boltd`) on headless systems where TB authorization is not needed; the kernel's TB controller will accept devices without `boltd` mediation.

The injector's relationship to F35 is observational: A4's telemetry surfaces the "device never reached `nv_pci_probe`" state via the absence of any per-device sysfs surface. Operator tooling can correlate this absence with `boltctl list` output to detect F35 explicitly.

## Assertion shape

**fake-5090 v1 cannot assert F35 directly.** The failure-index entry exists to:

1. Document the coverage boundary (no fake-5090 assertion).
2. Provide operator triage: "GPU connected but not in `lspci`" → check `boltctl list` for unauthorized state → F35.
3. Surface the `boltctl` authorization commands in a discoverable form.

**Real-hardware-only assertions (apnex setup-guide validation):**

- Cold-boot baseline: GPU attached at boot; `bolt` policy auto-authorize-at-boot; `lspci -nn -d 10de:` shows the device.
- Hot-plug case (negative): connect the GPU at runtime with `bolt` policy "user-prompt"; without explicit `boltctl authorize`, `lspci` does NOT show the device.
- Hot-plug case (positive): same setup but with policy `auto`; `lspci` DOES show the device within bounded time (≤ 5 s after hot-plug-attach).

**Negative assertion (what F35 is NOT):**

- F35 is NOT F33 (post-enumeration disconnect).
- F35 is NOT F29/F30/F31 (post-`bolt` allocation failures).
- F35 specifically: the device never appears in `lspci`; cause is at the `bolt` layer.

## fake-5090 mechanism mapping

- **User-space-class only.** The substrate exists below `boltd` in the stack. Reproducing F35 requires actually running `boltd` against a real TB controller with a real policy configuration.
- **Boundary-documentation role.** F35 is in the index so operators triaging "GPU connected but not detected" have a name for the `bolt` policy class.
- **Future-work hook.** Out of scope for v2 — F35 is a user-space policy issue, not a driver-stack issue.

## Source citations

- `boltctl(1)` man page; `boltd(8)` man page
- GNOME Thunderbolt security model documentation
- apnex/nvidia-driver-injector setup guide — §`bolt` configuration for TB GPUs (operator procedure)
- Distribution bug trackers — class of "TB device requires authorization" issues
- F33 entry (signal-loss vs CC-pin — sibling transport-layer issue)
- F29/F30/F31 entries (post-`bolt` BAR-allocation failures — operator-triage workflow needs to distinguish all of these from F35)
