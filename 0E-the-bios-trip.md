# 0E — The unavoidable BIOS trip

*Why [Chapter 01](01-bios.md) exists as a separate step, why no software
trick eliminates it, what the competition does, and the only path to a
truly-headless-from-power-on experience.*

---

## The claim, stated honestly

The Framework Desktop **cannot** be provisioned fully headlessly out of
the box. There is a one-time, unavoidable, ~5-minute physical session
required with a monitor and keyboard attached, to enable *Network Stack*
and *PXE Boot capability* in the UEFI setup menu. After that one session,
the machine can be fully headless forever — but you cannot skip that
first session.

This is not a limitation the setup guide can engineer around. It is a
property of Framework's firmware: they ship with network-boot disabled
by default, and those settings are only toggleable through the UEFI
setup UI, which requires a display and a keyboard to operate.

Pretending otherwise is how products damage their own credibility.
Accepting it openly is how you keep the rest of the brand honest.

---

## Why no software trick works

Three independent reasons, each sufficient on its own:

### 1. No remote BIOS access on prosumer hardware

The Framework Desktop has no IPMI, no BMC, no serial console, no AMT,
no iDRAC, no Redfish endpoint. These are lights-out-management features
enterprise server hardware has *for exactly this reason*. Framework
Desktop is a prosumer product in the consumer-workstation price band;
it does not include them. No software running on any other machine can
toggle BIOS settings on a Framework that isn't yet running an OS.

### 2. No OS-side tool can change UEFI setup options

`fwupd`, `efibootmgr`, `mokutil`, and similar Linux tools can update
firmware, manipulate UEFI boot entries, and query some firmware state —
but they **cannot change BIOS setup options** like "Network Stack:
Enabled." Those options live in NVRAM variables that the firmware only
modifies when it's running its own setup UI, in response to keyboard
input, on a screen. Once any OS is booted, the setup UI is gone.

(Technically you can script NVRAM-variable writes at the `efivarfs`
level, but the setup UI is what validates and persists its own writes
— editing vendor-defined NVRAM variables externally is undefined
behavior and can brick the firmware. Don't try.)

### 3. No vendor ships network-boot enabled by default on prosumer hardware

For security reasons. A machine that PXE-boots on any network it's
plugged into is a liability: boot-integrity attack surface on corporate
networks, random reflash loops if something on the LAN is serving
malicious installers, bandwidth abuse. Desktop-class machines
universally ship with PXE and network-stack *disabled* at factory.

This isn't a Framework choice you could lobby them to change. It's an
industry norm for a reason. The one-time opt-in is what keeps the
default safe.

---

## What the competition does

Every local-AI / self-hosted-server product that targets prosumer
hardware has the same problem, and none of them have solved it in the
sense you'd want:

- **Umbrel** ships on a Raspberry Pi with the OS pre-imaged on the SD
  card at factory. Boots directly to the OS. No BIOS step because the
  Pi's boot path is different from a UEFI desktop.
- **Start9 (Embassy)** ships on dedicated hardware with pre-configured
  firmware. The OS is preinstalled; power-on boots directly to their
  stack.
- **Home Assistant Yellow** ships with the OS pre-installed and
  first-boot configuration happens over Ethernet once the user plugs
  it in.
- **Synology NAS** units ship with the OS pre-installed and a network-
  based first-boot wizard. Customer does not open BIOS.

The pattern across all of them: **the vendor ships hardware with the
OS pre-installed at factory.** That's what makes first-boot headless.
If Framework were to offer "Framework Desktop for Leger" as a SKU where
they flash harness at the factory before shipping, the customer's
experience becomes: unbox → plug in → pair phone → done. No BIOS.

That's the V2 end state described in
[`0D — installer roadmap`](0D-the-installer-roadmap.md). It requires a
partnership with Framework, not a software change on your side.

---

## What's acceptable today (V1 reality)

Until the Framework partnership materializes, there are four honest
paths. Pick the one that matches your customer acquisition motion:

### Path A — The FDE does it during the implementation visit

For pilot and early V1 customers, a Leger engineer is on-site (or
remote-hands with a display borrowed from a friendly sysadmin) during
the setup engagement anyway. The BIOS trip is five minutes of that
engineer's time, while they're already there for the two-week
implementation. Customer never touches it.

**Applies to:** the first 20–50 customers, where the business model is
guided implementation (€5–15K setup fee). Captured inside the fee, not
a separate conversation.

### Path B — A printed card with instructions

For customers who arrive via self-serve channels (post-V1 launch) and
genuinely want to avoid a service engagement, ship a single-page card
with the Expansion Card:

> **Before first use — one-time, 5 minutes, one-time-only:**
> 1. Connect a monitor and USB keyboard to the Framework Desktop.
> 2. Power on, press F2 repeatedly until the BIOS menu appears.
> 3. On the **Boot** tab, find **Network Stack** → set to **Enabled**.
> 4. Find **PXE Boot capability** → set to **Enabled**.
> 5. Press **F10** to save and exit.
> 6. Unplug the monitor and keyboard — you won't need them again.

With a QR code linking to a 60-second video walkthrough. This is the
floor for self-serve. Two-thirds of the friction is in the sentence
*"connect a monitor and USB keyboard"* — not the actual BIOS steps,
which are trivial.

**Applies to:** self-serve customers post-V1 launch. Honest marketing
copy acknowledges the step openly: *"Ten-minute setup. Step one is a
one-time keyboard-and-monitor session to enable network boot (just
like setting up a new motherboard in a PC build). After that, every
future interaction is through the Leger app on your phone."*

### Path C — Pre-configured shipping service

Customer ships the Framework Desktop to Leger. Leger opens it, plugs
in a monitor and keyboard, does the BIOS config plus pre-installs
harness, repacks it, ships it back. Customer unboxes a pre-provisioned
box that boots to a working dashboard on first power-on.

**Applies to:** customers paying a premium for zero-friction onboarding.
Adds a week of shipping time and real fulfillment logistics (you now
have a warehouse / office where Framework Desktops arrive and leave).
Real cost, real time, but eliminates the BIOS step from the customer
side entirely.

### Path D — Framework partnership with factory pre-install

The endgame. Framework ships "Framework Desktop for Leger" with harness
pre-installed at the factory. Customer unboxes, powers on, sees the
Leger first-boot UI, pairs phone, done. No BIOS, no keyboard, no
monitor. This is how Dell ships Ubuntu on XPS, how System76 ships
Pop!_OS, how Lenovo ships Fedora on some ThinkPads. The mechanics are
standard; what's non-standard is the commercial agreement.

**Applies to:** V2+ once the partnership is real. 12–18 month
negotiation cycle at minimum. Not in the V1 plan. Not a blocker for
shipping.

---

## What the setup guide does about this

This repo is honest about the friction. [Chapter 01](01-bios.md) walks
through the BIOS trip in precise detail — it's the first chapter, it's
labeled *"physical access needed,"* and its closing paragraph is
explicit that you're doing this once and not again.

The installer roadmap in [`0D`](0D-the-installer-roadmap.md) describes
the ongoing experience after BIOS — V0 captive-portal, V1 BLE+app, V2
hardware pivot — none of which eliminate the BIOS step, all of which
improve everything after it.

A hypothetical future chapter, once the Framework partnership lands,
will document the "factory-preinstalled" variant and what it skips.
That chapter doesn't exist today because the partnership doesn't exist
today.

---

## Honest framing for customers

Do not pretend this is zero-touch. Do not hide it in the fine print.
Do not bury the monitor-and-keyboard requirement in "advanced setup."

Language that works:

> **Ten-minute setup, once. Headless forever.**
>
> Your Framework Desktop needs a one-time 5-minute BIOS configuration
> before it can be managed headlessly — the same step you'd do when
> building any new desktop PC. After that, every future interaction
> happens over Wi-Fi from your phone.

Language that doesn't:

> *"Unbox and go."*
> *"Zero configuration required."*
> *"No technical knowledge needed."*

The first is accurate and sets the right expectation. The latter three
are inaccurate and will damage the brand the first time an SMB admin
hits the BIOS step unprepared. Customers tolerate small amounts of
honest friction. They do not tolerate friction that the marketing
copy said wasn't there.

---

## The principle underneath

Hardware imposes constraints that software can't engineer away.
Recognizing this is necessary; it's what lets you commit product-design
effort to the parts that *are* under your control (the mobile app, the
dashboard, the provisioning runbook, the analytics ledger) and commit
*relationship* effort to the parts that require partners (Framework
factory pre-install).

The one-time BIOS step is the wall. You can route around it via
Framework partnership. You cannot climb it with better code.

Accept the wall; build beautifully in front of and behind it.
