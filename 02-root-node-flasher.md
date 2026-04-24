# 02 — Root node flasher stack

**Goal:** turn an existing Fedora machine (the "root node") into a one-cable
PXE server that will install a bootc container image onto the target Strix
Halo box over direct ethernet. After this chapter, powering the target on
triggers an unattended install of the OS image you point it at.

**Time:** ~30 minutes of setup. The stack stays live indefinitely; you run
`just activate` (or equivalent) each time you want to flash, and leave it off
the rest of the time.

**Prerequisites:**
- A Fedora machine (workstation, IoT, or bootc — all work) that will act as
  the root node.
- One spare ethernet port on the root node, physically cabled directly to the
  target's NIC. No switch required. If you only have one ethernet port, use
  a USB-ethernet adapter for this link and keep the built-in port for upstream.
- The target has already completed Chapter 01 (Network Stack + PXE Boot
  capability enabled in BIOS).
- A bootc container image, publicly pullable from a registry, that you want
  installed on the target. If you don't have one yet, use
  `quay.io/fedora/fedora-bootc:44` directly — the stack works end-to-end
  against stock Fedora bootc.

---

## Architecture

```
  root node (Fedora)                          target (Strix Halo, headless)
  ────────────────────                        ──────────────────────────────
  ethernet: 10.42.0.1/24 ◄─── direct cable ──► ethernet: DHCP from root
  NetworkManager "shared" mode
    ├── built-in dnsmasq on :67 (DHCP)
    │     + drop-in pxe.conf on :69 (TFTP)
    └── podman nginx on :80 (HTTP)
  /srv/netboot/
    tftp/
      x86_64-sb/ipxe-shim.efi     ◄── served over TFTP to UEFI firmware
      x86_64-sb/ipxe.efi          ◄── chained from shim, still Secure-Boot OK
    http/
      boot.ipxe                   ◄── iPXE reads this, loads kernel+initrd
      ks.cfg                      ◄── Anaconda reads this, drives the install
      fedora44/vmlinuz            ◄── Fedora 44 netinst kernel (cached or proxied)
      fedora44/initrd.img
```

The three protocols (DHCP, TFTP, HTTP) are exactly the minimum PXE+Secure-Boot
flow needs. No netboot.xyz, no matchbox, no Foreman, no Cobbler — those are
fleet-orchestrator tools for hundreds of machines. One cable, one target,
~20 lines of configuration.

---

## Step 1 — Bring up the direct-ethernet link

On the root node:

```bash
# Find the spare ethernet NIC (the one plugged into the target).
ip -br link show
# Note the name — example: enp191s0 or enx00e04c...
```

Create a NetworkManager connection on that interface with shared mode:

```bash
sudo nmcli connection add \
  type ethernet \
  con-name flasher \
  ifname enp191s0 \
  ipv4.method shared \
  ipv4.addresses 10.42.0.1/24 \
  ipv6.method disabled
sudo nmcli connection up flasher
```

**"Shared mode"** does three things in one flag: assigns `10.42.0.1` to the
interface, starts a lightweight dnsmasq on that interface serving DHCP in
`10.42.0.0/24`, and NATs traffic from the target out your upstream interface.
Your regular internet stays unaffected.

Verify:

```bash
ip addr show dev enp191s0       # should show 10.42.0.1/24
sudo ss -ulnp | grep :67         # should show dnsmasq listening on :67
```

**Gotcha**: NetworkManager's shared-mode dnsmasq only activates once the link
is up. If the target is off, the interface is "no-carrier" and no dnsmasq
runs. Either power the target just long enough to assert link (then power
off), or plug a USB-ethernet dongle at the target end temporarily, or use
`nmcli con modify flasher 802-3-ethernet.auto-negotiate yes` and accept that
the interface stays down until the target powers on.

---

## Step 2 — Fetch the signed iPXE bundle

iPXE is the second-stage bootloader. The firmware's own PXE ROM is
primitive (TFTP only). iPXE adds HTTP, scripting, and — critically — a
Secure-Boot-compatible path via a Microsoft-signed shim binary. Fetch the
official pre-signed release, don't build from source:

```bash
sudo mkdir -p /srv/netboot/tftp
cd /srv/netboot/tftp
sudo curl -L -o ipxeboot.tar.gz \
  https://github.com/ipxe/ipxe/releases/latest/download/ipxeboot.tar.gz
sudo tar xzf ipxeboot.tar.gz
# Unpacks to x86_64-sb/ipxe-shim.efi and x86_64-sb/ipxe.efi (and other arches).
ls x86_64-sb/
```

`ipxe-shim.efi` is Microsoft-signed (via shim). It loads `ipxe.efi`, which is
signed by iPXE's own cert (trusted by the shim). Both combined keep Secure
Boot enabled end-to-end — no BIOS trips required.

---

## Step 3 — Fetch the Fedora 44 netinstall kernel and initrd

These are the artifacts Anaconda needs to bootstrap itself. You can either
proxy them from the public Fedora mirror at boot time (less to maintain) or
cache them locally (more reliable if your upstream is slow or flaky). Local
cache:

```bash
sudo mkdir -p /srv/netboot/http/fedora44
cd /srv/netboot/http/fedora44
base=https://dl.fedoraproject.org/pub/fedora/linux/releases/44/Everything/x86_64/os/images/pxeboot
sudo curl -LO $base/vmlinuz
sudo curl -LO $base/initrd.img
```

If you prefer the proxy path, skip this and adjust `boot.ipxe` in Step 5.
The cached path is what this chapter uses by default — ~100 MB, one-time
download, makes reflashes predictable.

---

## Step 4 — Drop the dnsmasq PXE configuration

NetworkManager's shared-mode dnsmasq reads drop-ins from
`/etc/NetworkManager/dnsmasq-shared.d/`. We add one file:

```bash
sudo tee /etc/NetworkManager/dnsmasq-shared.d/pxe.conf >/dev/null <<'EOF'
enable-tftp
tftp-root=/srv/netboot/tftp

# Match by UEFI client-arch value per RFC 4578:
#   7 = EFI byte code   9 = EFI x86-64   0 = BIOS (not used here)
dhcp-match=set:efi64,option:client-arch,7
dhcp-match=set:efi64,option:client-arch,9

# Break the chain-reboot loop: once iPXE is loaded it re-DHCPs identifying as
# "iPXE" in the user-class; we then hand it the HTTP script instead of the
# TFTP binary it just ran.
dhcp-userclass=set:ipxe,iPXE

# Stage 1: firmware → signed iPXE shim (TFTP)
dhcp-boot=tag:efi64,tag:!ipxe,x86_64-sb/ipxe-shim.efi

# Stage 2: iPXE → our boot script (HTTP)
dhcp-boot=tag:ipxe,http://10.42.0.1/boot.ipxe
EOF
```

Reload NetworkManager so the drop-in takes effect:

```bash
sudo nmcli connection down flasher && sudo nmcli connection up flasher
```

Verify TFTP is now listening:

```bash
sudo ss -ulnp | grep ':69'
```

---

## Step 5 — Write the iPXE boot script

```bash
sudo tee /srv/netboot/http/boot.ipxe >/dev/null <<'EOF'
#!ipxe
set base http://10.42.0.1
kernel ${base}/fedora44/vmlinuz \
  initrd=initrd.img \
  inst.repo=https://dl.fedoraproject.org/pub/fedora/linux/releases/44/Everything/x86_64/os \
  inst.ks=${base}/ks.cfg \
  inst.text ip=dhcp
initrd ${base}/fedora44/initrd.img
shim https://dl.fedoraproject.org/pub/fedora/linux/releases/44/Everything/x86_64/os/EFI/BOOT/BOOTX64.EFI
boot
EOF
```

The `shim` line is what keeps Secure Boot working: iPXE chainloads Fedora's
own signed shim, which then vouches for the kernel+initrd. Documented at
https://ipxe.org/secboot. The pattern is verbatim from the Fedora example
there.

---

## Step 6 — Write the kickstart

This is the file Anaconda reads to run the install unattended. Replace
`REPLACE_ME_SSH_PUBKEY` with the output of `cat ~/.ssh/id_ed25519.pub` from
the machine you'll SSH into the target from. Replace the image reference
with whatever bootc image you want installed:

```bash
sudo tee /srv/netboot/http/ks.cfg >/dev/null <<'EOF'
text --non-interactive
zerombr
clearpart --all --initlabel --disklabel=gpt
reqpart --add-boot
part / --grow --fstype=btrfs --label=root

network --bootproto=dhcp --device=link --activate --onboot=on

timezone Etc/UTC --utc
lang en_US.UTF-8
keyboard us

rootpw --lock
user --name=core --groups=wheel --lock
sshkey --username=core "REPLACE_ME_SSH_PUBKEY"

firewall --disabled
selinux --enforcing
services --enabled=sshd

bootc --source-imgref=registry:quay.io/fedora/fedora-bootc:44 \
      --stateroot=default

reboot --eject

%post --erroronfail --interpreter=/usr/bin/bash
install -m 0440 -D /dev/stdin /etc/sudoers.d/10-wheel-nopasswd <<SUDO
%wheel ALL=(ALL) NOPASSWD: ALL
SUDO
%end
EOF
```

A few things to understand, not to copy blindly:

- `bootc` (not `ostreecontainer`) is the current kickstart verb for bootc
  installs. Fedora 43+.
- `part /` is explicit — **do not** use `autopart`; it silently breaks on the
  `/boot/efi` layout bootc expects.
- `--fstype=btrfs` matches the fedora-bootc default. If you switch to an
  image that pins xfs, change this line to match.
- `rootpw --lock` and `user --lock` mean the `core` user has no password —
  SSH-key-only. The sudoers drop-in grants passwordless sudo over wheel,
  which `core` is a member of. That lets `efibootmgr --bootnext` and
  `systemctl reboot` run non-interactively from remote SSH (see the reflash
  loop below).
- Replace `quay.io/fedora/fedora-bootc:44` with your own image when you have
  one. The image must be publicly pullable — `bootc install` does not yet
  handle authenticated registries cleanly.
- No `%post` writes to `/etc` beyond the sudoers drop. Tailscale, WireGuard,
  secrets, etc. belong in the **image** (`/usr/lib/systemd/system/…`), not
  the kickstart. Kickstart is stateless-per-flash; anything you write here
  gets re-deployed by `bootc upgrade` unless it lives under `/etc` (which
  is preserved with three-way merge) — and even there, baking into the
  image is cleaner.

---

## Step 7 — Run nginx on 10.42.0.1:80

HTTP serves `boot.ipxe`, `ks.cfg`, and the cached kernel/initrd. Any static
HTTP server works; we use nginx in a podman container because the root node
may be bootc (read-only `/usr`) and we don't want to layer packages:

```bash
sudo podman run -d \
  --name flasher-nginx \
  --restart=unless-stopped \
  -p 10.42.0.1:80:80 \
  -v /srv/netboot/http:/usr/share/nginx/html:ro,Z \
  docker.io/library/nginx:alpine
```

For a more permanent setup, write a systemd Quadlet under
`/etc/containers/systemd/flasher-nginx.container` — left as an exercise.

Verify:

```bash
curl -I http://10.42.0.1/boot.ipxe      # 200 OK
curl -I http://10.42.0.1/ks.cfg         # 200 OK
curl -I http://10.42.0.1/fedora44/vmlinuz   # 200 OK
```

---

## Preflight — all five must pass before you power the target

```bash
sudo ss -ulnp | grep ':67'                            # DHCP listening
sudo ss -ulnp | grep ':69'                            # TFTP listening
curl -I http://10.42.0.1/boot.ipxe                    # HTTP boot script
curl -I http://10.42.0.1/ks.cfg                       # kickstart
ls -la /srv/netboot/tftp/x86_64-sb/ipxe-shim.efi      # shim present
```

If any of these fail, **stop and fix it**. Powering the target on before the
stack is green just means the firmware transmits DHCPDISCOVER, hears nothing,
times out, and you learn nothing about which layer is broken. Debug with
one-variable-at-a-time cycles.

Also useful: `sudo tcpdump -i <flasher-iface> -n -e 'port 67 or port 69 or
port 80'` in a second terminal while the target boots. You should see DHCP
offer → TFTP get of `ipxe-shim.efi` → HTTP GET `/boot.ipxe` → HTTP GET
`/ks.cfg` → HTTP GET of the kernel and initrd, in that order.

---

## First flash

Target is powered off at the end of Chapter 01, BIOS is configured, ethernet
is cabled to the root node's `enp191s0`.

1. Confirm the root node's flasher stack passes preflight above.
2. Power the target on. Don't touch anything.
3. The target's firmware tries disk first (nothing there on a fresh box) →
   Automatic Failover kicks in → PXE → DHCP from root node → signed iPXE
   shim over TFTP → `boot.ipxe` over HTTP → Anaconda boots → reads kickstart
   → `bootc install` pulls the image from your registry → reboot.

Total time, typical: ~6 minutes from power-on to the target booting into the
installed OS. Dominated by the `bootc` image pull (~8 GB at whatever your
upstream gives you).

If Automatic Failover doesn't kick in on the very first power-on (rare but
possible on some firmware), plug the monitor+keyboard back briefly, tap F12
at POST, select **UEFI PXE IPv4**. One-shot, never needed again after this.

---

## After the first flash — two captures

Once the target boots into the installed OS, two pieces of state need
capturing so subsequent reflashes are hands-off.

### Capture the MAC for a static DHCP reservation

On the root node:

```bash
sudo journalctl -u NetworkManager | grep DHCPACK | tail -5
# Look for a line like:
#   DHCPACK(enp191s0) 10.42.0.47 a1:b2:c3:d4:e5:f6 harness
```

Add the MAC to the dnsmasq drop-in so the target always gets the same IP:

```bash
sudo tee -a /etc/NetworkManager/dnsmasq-shared.d/pxe.conf >/dev/null <<'EOF'

# Static reservation — replace the MAC with what you captured.
dhcp-host=a1:b2:c3:d4:e5:f6,10.42.0.2,harness
EOF
sudo nmcli connection down flasher && sudo nmcli connection up flasher
```

Append the matching SSH client config on the root node:

```bash
cat >> ~/.ssh/config <<'EOF'

Host harness
  HostName 10.42.0.2
  User core
  StrictHostKeyChecking accept-new
  UserKnownHostsFile /dev/null
EOF
```

The `/dev/null` known-hosts and `accept-new` are intentional — host keys
regenerate on every reflash by design, so accumulating fingerprints in the
real known_hosts just means "MITM!" screaming on every iteration. For a
reflash loop on a trusted direct link, this is the cleanest shape.

### Capture the PXE boot entry number

From the root node, SSH in and read the UEFI boot menu:

```bash
ssh harness sudo efibootmgr -v
# Look for the line like:
#   Boot0005* UEFI PXEv4 (MAC:a1b2c3d4e5f6)  ...
# Save the hex number — "0005" in this example.
```

---

## The reflash loop

With the MAC reservation and the boot-entry hex captured, any future reflash
is one command from the root node:

```bash
ssh harness "sudo efibootmgr -n 0005 && sudo systemctl reboot"
```

`efibootmgr -n` writes UEFI's `BootNext` variable — a one-shot override
consumed by the next boot. The target reboots, network-boots the flasher
stack exactly once, then the variable self-clears and the machine returns to
its normal disk-first boot order afterward. No BIOS trip, no human, no
babysitting.

To change what gets installed on the next reflash: edit `ks.cfg` (swap the
`--source-imgref` to the new image tag), edit `boot.ipxe` if the kernel URL
changed, and trigger. Nothing on the root node needs restarting between
iterations.

---

## What this chapter does NOT do

- **Any cloud-account provisioning.** This is pure hardware + direct-link
  setup. Cloudflare / GitHub / Google Workspace / domain + DNS come in later
  chapters.
- **Any in-image configuration.** If you need Tailscale, Vault, or a first-
  boot enrollment unit on the target, bake it into the bootc image —
  that's the image maintainer's job, not the flasher's. Kickstart is
  deliberately minimal here.
- **Fleet provisioning.** One root node, one direct-ethernet cable, one
  target. If you're deploying N targets, this same stack works in sequence
  — power one at a time, reflash, move the cable. For parallel / large-
  fleet, add a managed switch and per-MAC kickstart variants; the
  architecture generalizes but this chapter doesn't cover it.

---

## Troubleshooting

**Target never transmits DHCPDISCOVER.** Chapter 01 Part 2 failed — Network
Stack or PXE Boot capability did not register. Re-enter BIOS, re-verify.
Link LEDs on both ends must be solid before the target boots.

**DHCPDISCOVER seen, no reply.** dnsmasq isn't serving on the flasher
interface. Usually `flasher` connection is down (no link at activation
time), or the drop-in file has a typo. `journalctl -u NetworkManager -f`
while the target boots.

**DHCP offer goes out, target never TFTPs.** `dhcp-boot` line is wrong, or
`tftp-root` path doesn't exist, or the shim binary isn't at the expected
relative path. Verify `ls /srv/netboot/tftp/x86_64-sb/ipxe-shim.efi`.

**TFTP succeeds, iPXE loads, but "No such file or directory" on boot.ipxe.**
nginx isn't running, or it's bound to the wrong IP, or SELinux is blocking
the bind-mounted docroot. `sudo podman logs flasher-nginx`. The `:Z` on the
volume mount relabels the directory for SELinux — don't drop it on a
machine with enforcing SELinux.

**Anaconda hangs at "Starting installation."** The image is private or the
registry is unreachable. `podman logout ghcr.io && podman pull
<your-image>` from any third machine to verify public pullability.

**Install completes, box reboots to "no boot device."** The bootc install
may have succeeded but the firmware didn't update `BootOrder` to include
the new disk entry. Plug the monitor back in, boot into BIOS, verify a
`UEFI OS` or similar disk entry exists in the boot menu. If not, file a
bug — this is rare on Framework Desktop but common on some other Strix
Halo boxes with quirky UEFI implementations.

---

Next chapter: the **harness** bootc image — what lives in `/usr`, why image
baking is the right layer for first-boot services, and the image-side
counterpart to the kickstart above.
