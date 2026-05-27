# QEMU topology for fake-5090

A Thunderbolt-attached eGPU enumerates through a switch hierarchy
roughly like:

```
host pcie root → TB root port → TB upstream (eGPU enclosure) → TB downstream → GPU
```

We can model this faithfully in QEMU using stock device models, with
our vfio-user fake-5090 daemon at the leaf. The injector's E1
(eGPU detection) patch walks the upstream chain looking for
Thunderbolt markers — modelling the right topology is essential or
E1 will never fire.

## Devices used (all present in obpc's QEMU 10.2.2)

| QEMU device                  | Role in the model                        |
|------------------------------|------------------------------------------|
| `pcie-root-port`             | Stands in for the host's TB4 root port   |
| `x3130-upstream`             | Stands in for the eGPU enclosure's upstream switch port |
| `xio3130-downstream`         | Stands in for the downstream port to the GPU |
| `vfio-user-pci`              | The fake-5090 daemon, leaf device        |

The TI x3130 / xio3130 models are *not* literally Thunderbolt
silicon — they're standard PCIe switch silicon. For E1 detection
purposes, what matters is the **shape** of the chain (root port →
upstream → downstream → endpoint) and the **capability presence**,
not the specific vendor IDs. If E1 keys on TB-specific markers
(USB4 device-level config, specific class codes), we may need to
either patch the model's vendor IDs or extend the daemon to advertise
TB markers in its own capability chain.

## Skeleton QEMU CLI

```bash
qemu-system-x86_64 \
  -enable-kvm -cpu host -smp 4 -m 8G \
  -machine q35,kernel-irqchip=split \
  -device intel-iommu,intremap=on,caching-mode=on \
  \
  -device pcie-root-port,id=rp0,bus=pcie.0,chassis=1,slot=0 \
  -device x3130-upstream,id=ups0,bus=rp0 \
  -device xio3130-downstream,id=dns0,bus=ups0,chassis=2,slot=0 \
  \
  -device vfio-user-pci,socket=/tmp/fake-5090.sock,bus=dns0 \
  \
  -drive file=fedora-44-test.qcow2,if=virtio \
  -nographic
```

Pair with the daemon listening on `/tmp/fake-5090.sock`. The guest
boots, the patched driver loads, and `lspci -t` should show the GPU
behind a two-level switch hierarchy hung off the root port — exactly
the topology E1 expects to walk.

## Open items

- Confirm whether E1 keys on the Thunderbolt-specific `Vendor-Specific
  Extended Capability` (VSEC) ID `0x1137` or similar. If yes, the
  fake-5090 daemon must advertise that VSEC in its config-space
  capability chain — the x3130 model alone won't be enough.
- Confirm `vfio-user-pci` works correctly behind a multi-level switch
  in QEMU 10.2 (worth a 30-min smoke test before committing to this
  topology).
- Surprise-removal: QMP `device_del rp0` (entire root port) is the
  cleanest analogue. Test that the close-path telemetry (A4) fires
  end-to-end.
