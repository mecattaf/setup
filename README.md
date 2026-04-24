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

### Why (read first if you want the argument)

| # | Title | What it covers |
|---|-------|----------------|
| 00 | [Why this guide exists](00-why.md) | The commercial case. On-prem AI economics, the ledger principle, the 90-day falsifiable wager. |
| 0A | [Skills need a policy layer](0A-skills-need-a-policy-layer.md) | The technical thesis. Why setup and invocation must be separated. Why Leger is for this. |
| 0B | [ISO vs PXE vs Tunnel](0B-iso-vs-pxe-vs-tunnel.md) | Why this guide standardizes on PXE, what the Cloudflare-Tunnel upgrade path looks like, when each approach is right. |
| 0C | [Harness vs dynamic layer](0C-harness-vs-dynamic-layer.md) | What belongs in the slow-moving bootc image vs the fast-moving Leger layer vs operator dotfiles. |

### How (the mechanical pipeline)

| # | Title | Where it runs | Physical access needed |
|---|-------|---------------|------------------------|
| 01 | [BIOS configuration](01-bios.md) | on the target | yes (monitor + USB keyboard, once) |
| 02 | [Root node flasher stack](02-root-node-flasher.md) | on the root node | no (after Ch. 01 unplug) |

More chapters to come: harness image composition, first flash + headless
handoff, account chain (Google Workspace → Cloudflare → GitHub org →
Tailscale), and cloud provisioning via `leger bootstrap` verbs.

## Conventions

- **"target"** = the fresh Strix Halo box being provisioned (headless after this guide).
- **"primary"** / **"root node"** = the Fedora machine orchestrating provisioning.
- **"flasher"** = the primary-side PXE stack (separate repo).
- **"harness"** = the bootc container image laid down on the target (separate repo).

## License

MIT unless stated otherwise.
