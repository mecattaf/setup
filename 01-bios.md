# 01 — BIOS configuration

**Goal:** enable UEFI network boot on a factory-fresh target so that the root node
can take over remotely from here on.

**Time:** ~5 minutes. This is the only chapter that requires physical access
(monitor + USB keyboard). After it, the target is headless for good.

**Prerequisite:** box is factory-fresh (no OS installed). Ethernet is **not**
plugged in yet — we verify link registration separately, later.

---

## Target end state (applies to any Strix Halo box)

Regardless of BIOS vendor, after this chapter the target's firmware must be in
this state:

| Setting | Value | Why |
|---|---|---|
| UEFI Network Stack | **Enabled** | Master toggle. Without this, PXE ROM never loads. |
| PXE IPv4 Boot | **Enabled** | Actual boot method the flasher uses. IPv6 not needed. |
| HTTP / HTTPS Boot | Disabled | iPXE handles HTTP itself once loaded; native UEFI HTTP Boot is a separate path we don't use. |
| Secure Boot | **Enabled** | Factory default. iPXE ships a Microsoft-signed shim — SB stays on end-to-end. |
| Boot order | Disk first, unchanged | We trigger PXE explicitly via `efibootmgr --bootnext` from the primary. Putting PXE first is dangerous: every power-on would re-install. |
| Automatic Failover | Enabled | Factory default. Disk-boot failure falls through to PXE, which is the desired first-install behavior. |

Everything else stays at factory default.

---

## Framework Desktop (Insyde BIOS) — worked example

### Before you sit down

- A **wired USB keyboard** in a rear USB-A port. Wireless keyboards and USB hubs
  have been reported not to register in the Insyde BIOS by the Framework
  community — don't fight it, use a wired keyboard in a direct port.
- A monitor on one of the rear HDMI or DisplayPort outputs.
- Mouse optional but works. Keyboard is more reliable for everything below.
- **Ethernet not plugged in yet.** Link registration is verified at the end of
  this chapter in a deliberate order.

### Key reference

| Key | Action |
|---|---|
| F2 | Enter BIOS during POST (hold if first tap doesn't register — known Framework quirk; can take up to two minutes on a cold box) |
| F12 | One-shot boot menu at POST |
| F5 / F6 | Change the value under the cursor |
| F9 | Load defaults — **do not press** |
| F10 | Save and exit |
| Esc | Back out of a submenu / cancel |
| ↑ ↓ ← → | Navigate |
| Enter | Select / enter submenu |

### Step 1 — Enter BIOS

Power on. Tap **F2** repeatedly during POST until the Insyde setup screen
appears. The first tab you land on is **Main**; write down the BIOS version
shown there. You will need this when triaging the 3.04 slow-boot regression
post-install (see `pre-launch-items` in the flasher repo).

### Step 2 — Enable Network Stack

The Framework Desktop Boot tab has **11 items** (this is different from the
Framework Laptop 13, which has 12 — the Desktop omits the inline *EFI Boot
Order* submenu and adds two toggles Laptop 13 lacks):

1. Power on AC attach
2. Quick Boot
3. Quiet Boot
4. **Network Stack** ← change this
5. **PXE Boot capability** ← change this
6. HTTP/HTTPS Boot capability
7. USB Boot
8. Timeout
9. Automatic Failover
10. New Boot Device Priority
11. *(no `EFI Boot Order` submenu — the Desktop manages boot order via OS-side `efibootmgr` only)*

Arrow right to **Boot**. Arrow down to **Network Stack**. Press **F5** or **F6**
to flip `Disabled` → `Enabled`.

### Step 3 — Enable PXE Boot capability

Still on the Boot tab, arrow down to **PXE Boot capability**. Press F5/F6 to
flip `Disabled` → `Enabled`.

> **This is the non-obvious one.** On the Framework Laptop 13, enabling
> Network Stack is sufficient and the only PXE toggle. On the Framework
> Desktop, Network Stack is gated by a second switch that defaults off.
> Missing this is the single most common reason the target appears to "PXE
> correctly" in BIOS but never actually transmits DHCPDISCOVER at boot.

### Step 4 — Leave the rest of the Boot tab alone

- **HTTP/HTTPS Boot capability**: leave `Disabled`. We use iPXE-over-PXE, not
  native UEFI HTTP Boot.
- **USB Boot**: leave `Enabled` (default). Useful for recovery.
- **Automatic Failover**: leave `Enabled` (default). Helpful for the PXE retry
  behavior on first install.
- **Boot order / New Boot Device Priority**: leave untouched. There is **no**
  inline EFI Boot Order submenu on the Desktop — don't go looking for one.

### Step 5 — Verify Secure Boot is on (don't change anything)

Arrow right to **Security**. Arrow down to **Secure Boot**, press **Enter** to
enter the submenu. Confirm **Enforce Secure Boot** shows `Enabled` (factory
default). Press **Esc** to back out.

Do not touch PK, KEK, DB, DBX, DBT, DBR, "Enroll PK", "Delete PK", "Erase all
Secure Boot Settings", or "Restore Secure Boot to Factory Settings".

### Step 6 — Save and exit

Press **F10**. Confirm **Yes**. The machine reboots. It will land on a QR-code
"no boot device" screen — this is expected on a fresh disk with nothing
serving PXE yet.

---

## Verification (still with monitor + keyboard)

Do not unplug peripherals until these two checks pass.

### Verify the settings persisted

Power cycle, press F2, go back to the Boot tab. Confirm:

- **Network Stack**: `Enabled`
- **PXE Boot capability**: `Enabled`

If either reverted to `Disabled`, redo Step 2 / Step 3 and save again. If it
reverts a second time, that's a CMOS / NVRAM issue — stop and open a Framework
support ticket; don't continue.

### Verify the NIC registers and transmits

This requires ethernet link on the target's rear port. Any device on the other
end of the cable that asserts link will do — it does **not** need to be the
root node, and does **not** need to be serving PXE yet.

1. Plug ethernet between the target and any live device (your laptop, a
   switch, the root node, anything with link LEDs).
2. Power cycle the target. Tap **F12** during POST.
3. The one-shot boot menu should include an entry labelled **UEFI PXE IPv4**
   (exact wording varies: "UEFI: IPv4 Network", "EFI PXE 0 for IPv4", etc.).
   If it does not appear, Network Stack + PXE Boot capability did not
   register the NIC. Verify ethernet link LEDs, reboot, re-enter F12.
4. Select the IPv4 entry — **not** IPv6, **not** a combined "PXE" if separate
   entries exist. The screen should show:

   ```
   Start PXE over IPv4. Press ESC to exit.
   ```

   This means the firmware's PXE ROM is transmitting DHCPDISCOVER. Exact
   confirmation that the target side works.

5. Press **Esc** to abort, or let it time out (~30-60 seconds) and fall
   through to the QR-code screen. Either is fine.

#### Optional — prove it over the wire

If the device on the other end is a Linux machine, you can confirm the target
is actually transmitting:

```bash
ip -br link show                          # find the NIC name (e.g. enp6s0)
sudo tcpdump -i <that-nic> -n -e 'port 67 or port 68'
```

While `tcpdump` is running, retrigger the target's F12 → UEFI PXE IPv4. You
should see lines like:

```
... BOOTP/DHCP, Request from <target-mac>, length ...
```

That is conclusive: NIC alive, PXE ROM loaded, DHCPDISCOVER transmitted on
the right wire. If tcpdump sees nothing across 60 seconds, something is wrong
on the target side (wrong port, cable, or a missed BIOS toggle) — stop and
retrace.

---

## Power down and go headless

1. Hold the power button ~5 seconds to force off (the target may be stuck at
   the QR screen waiting for a boot source — that's fine).
2. Unplug monitor. Unplug keyboard. Unplug any stray USB devices (a forgotten
   bootable USB can hijack the boot order later).
3. Plug the ethernet cable into the root node's port that will serve the
   flasher stack. Do not power the target back on yet — the flasher repo's
   preflight runs first.

Keep the monitor and keyboard within reach of this machine for one more
power-on's worth of safety margin. If the target's firmware does not
automatically fall through from a failed disk boot to PXE on first
power-on (uncommon but possible), you will briefly need F12 access again.
After the first successful install, `efibootmgr --bootnext` over SSH
replaces F12 forever.

---

## Generalizing to other Strix Halo boxes

Other AMD Ryzen AI Max 300 platforms ship different BIOS vendors — typically
AMI Aptio (MS-A2, EVO-X2, GTR9 Pro) or a vendor-customized Insyde (HP Z2 Mini
G1a). The target end state is identical; only menu paths differ. Translate
by name:

- **AMI Aptio**: `Advanced → Network Stack Configuration → Network Stack =
  Enabled`, then `Ipv4 PXE Support = Enabled`, `Ipv6 PXE Support = Disabled`.
  Secure Boot lives under `Security → Secure Boot`.
- **HP Z2 Mini G1a**: most of the network-boot config is gated behind an HP
  setup password on some SKUs. Check `F10 BIOS → Advanced → Boot Options`
  and `Secure Boot Configuration`.
- **Insyde (other vendors)**: similar to Framework Desktop but the two-toggle
  split (`Network Stack` + `PXE Boot capability`) is not universal — some
  Insyde builds collapse both into `Network Stack` alone.

The verification procedure above (F12 → UEFI PXE IPv4 → "Start PXE over
IPv4" → optional tcpdump) is vendor-independent and is the real signal that
the BIOS side is done. If that works, you can ignore any disagreement
between what the vendor's menus are named and what this chapter says.

---

## What changed, what did not

Changed on the target:

- `Boot → Network Stack`: Disabled → Enabled
- `Boot → PXE Boot capability`: Disabled → Enabled

Explicitly **not** changed:

- Boot order
- Secure Boot (left at factory `Enabled`)
- HTTP/HTTPS Boot capability (left at factory `Disabled`)
- Any Secure Boot key store (PK/KEK/DB/DBX)
- CPU / memory / storage / TPM / fan / power settings

Next chapter: primary-side flasher stack (iPXE signed shim + dnsmasq drop-in +
nginx docroot + kickstart). Lives in the flasher repo, not this one.
