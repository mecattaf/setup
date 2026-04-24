# setup

Step-by-step guide for provisioning an **AMD Ryzen AI Max 300 ("Strix Halo")** box
as a headless bootc node, driven remotely by a Fedora primary ("root node") over
a direct ethernet link.

Written against the **Framework Desktop** (Insyde BIOS, RTL8126 5 GbE NIC), which
is the primary target. Other Strix Halo boxes (MINISFORUM MS-A2, GMKtec EVO-X2,
HP Z2 Mini G1a, Beelink GTR9 Pro, etc.) use the same CPU family but ship
different BIOS vendors — the high-level requirement ("UEFI network stack on,
PXE IPv4 on, Secure Boot on") is identical, only the menu paths differ. Use the
Framework chapter as the worked example and translate by name.

## Scope

This repo is the **fresh-box side** of the pipeline. It ends at "box is powered
off, ethernet plugged in to primary, will network-boot on next power-on." From
that point forward, the primary's flasher stack takes over; that's a separate
repo.

Assumes:

- The root node is an existing Fedora machine (bootc or workstation — either works).
- The root node will run the PXE / DHCP / HTTP stack to network-install the target.
- Secure Boot is desired **on** end-to-end (via iPXE's Microsoft-signed shim).
- The target has **no operating system installed** yet (factory-fresh).

## Chapters

| # | Title | Where it runs | Physical access needed |
|---|-------|---------------|------------------------|
| 01 | [BIOS configuration](01-bios.md) | on the target | yes (monitor + USB keyboard, once) |

More chapters will be added as the pipeline solidifies (primary-side flasher
stack, first install, reflash loop, tailnet enrollment, fleet scaling).

## Conventions

- **"target"** = the fresh Strix Halo box being provisioned (headless after this guide).
- **"primary"** / **"root node"** = the Fedora machine orchestrating provisioning.
- **"flasher"** = the primary-side PXE stack (separate repo).
- **"harness"** = the bootc container image laid down on the target (separate repo).

## License

MIT unless stated otherwise.
