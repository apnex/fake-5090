# vfio-user reference

vfio-user is a protocol that lets a userspace process serve a PCI
device to QEMU (or any other consumer) over a UNIX socket. QEMU's
`vfio-user-pci` device acts as the client. The daemon owns config
space, BARs, IRQs.

## Why this fits fake-5090

- **No QEMU patches** — `vfio-user-pci` is in mainline QEMU since 7.x
  and is **present in obpc's QEMU 10.2.2** (verified
  `qemu-system-x86_64 -device help | grep vfio-user`).
- **Out-of-process** — daemon crash is contained, not a QEMU crash
  and certainly not a host crash.
- **Scriptable** — failure injection is just "the daemon decides to
  return 0xFFFFFFFF for this config-space read." Trivial to wire to
  a control socket and orchestrate from a test harness.
- **Language-agnostic** — any language with UNIX sockets and the
  protocol can implement it. Reference impl is C.

## Protocol summary (just enough to design)

- Transport: UNIX socket (SOCK_STREAM).
- Wire format: header + payload, little-endian, message-framed.
- Message types: `VFIO_USER_VERSION`, `VFIO_USER_DMA_MAP`,
  `VFIO_USER_DMA_UNMAP`, `VFIO_USER_DEVICE_GET_INFO`,
  `VFIO_USER_DEVICE_GET_REGION_INFO`, `VFIO_USER_REGION_READ`,
  `VFIO_USER_REGION_WRITE`, `VFIO_USER_DEVICE_GET_IRQ_INFO`,
  `VFIO_USER_DEVICE_SET_IRQS`, `VFIO_USER_DEVICE_RESET`, ...
- Regions: index 7 = PCI config space; indices 0–5 = BARs;
  index 6 = ROM.
- The interesting work for fake-5090 happens in:
  - `REGION_READ` / `REGION_WRITE` on config space — implement
    capability chain here
  - `REGION_READ` / `REGION_WRITE` on BAR0 — implement MMIO state
    machine here (only the registers the driver actually touches)
  - `DEVICE_RESET` — needed for A3-recovery patch testing

## Implementations

| Project          | Language | Notes                                        |
|------------------|----------|----------------------------------------------|
| `libvfio-user`   | C        | Reference. Originally Nutanix, now community-maintained under SPDK org. |
| `vfio-user-rs`   | Rust     | Independent Rust crate. Smaller surface, more ergonomic for new daemons. |
| QEMU client      | C        | `hw/remote/vfio-user-obj.c` (server side, for QEMU-as-server) + `hw/vfio/user.c` (client side). |

For fake-5090, **`libvfio-user` (C)** is the safer pick:
- Matches the driver/kernel idiom apnex is already in
- Largest set of working examples (gpio, nvme samples ship in-tree)
- Stable API

## Links to capture (TODO when online)

- libvfio-user repo and protocol spec
- QEMU docs page for `vfio-user-pci`
- vfio-user-rs crate
- A worked example daemon (the in-tree `gpio` or `lspci` samples are
  good starting points)

## Build notes (Fedora 44)

`libvfio-user` is **not packaged** in Fedora 44 (`dnf info
libvfio-user` returned nothing). Build from source:

```bash
git clone https://github.com/nutanix/libvfio-user
cd libvfio-user
meson setup build
meson compile -C build
sudo meson install -C build  # or stay local + LD_LIBRARY_PATH
```

Dependencies: `meson`, `ninja-build`, `json-c-devel`, `cmocka-devel`
(for the test suite), `python3-cffi` (for the Python bindings, if
wanted).
