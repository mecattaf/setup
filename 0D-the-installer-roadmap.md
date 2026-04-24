# 0D — The installer roadmap

*Why the thing the customer unboxes is not the same thing the customer uses,
why the provisioning gesture is worth thinking about as a standalone product
surface, and the V0 → V1 → V2 arc from "scaffolding we tolerate" to "the
actual product."*

---

The chapters so far have been about the mechanical pipeline: BIOS, flasher,
harness image. That pipeline works, but only for operators who already know
what a kickstart is. For an SMB admin buying Leger as a packaged product,
the real question is *how does the thing arrive and how does the customer
turn it on?*

This chapter is about that — the physical installer the customer touches,
and the three-stage roadmap from "we ship what we can today" to "we ship
what the product deserves."

---

## The observation that started this

An SMB admin unboxing an AI appliance should be able to configure it from
their phone. They should not need a keyboard, a monitor, a USB stick from
their laptop, or knowledge of what a kickstart is. The Roomba sets this
bar: plug it in, open the app, allow Wi-Fi sharing, done.

Today's setup guide doesn't meet that bar. Today's setup guide requires:

- A root node on the same LAN as the target
- A direct ethernet cable
- Someone who knows how to run `nmcli connection add`
- A one-time BIOS trip with monitor + keyboard

That's fine for operators (you, future FDEs) provisioning hardware for
customers. It's the wrong shape of experience for customers provisioning
their own box.

The installer roadmap below is how you close that gap, staged so each phase
ships on its own and each phase informs the next.

---

## V0 — Captive-portal installer on a Framework MicroSD Expansion Card

### What it is

Bulk-buy **Framework's existing MicroSD Expansion Card** (~€20 retail,
probably €12–15 at wholesale), flash the embedded MicroSD with a thin
Fedora installer image you maintain, ship it in a small Leger-branded box
with a QR card.

You are not manufacturing hardware. You are reusing Framework's SKU. Zero
PCB design, zero firmware, zero certification, zero inventory risk for the
casing. The only custom element is the 2 GB installer image on the micro
SD card inside the expansion card — which is just a file you flash.

### What ships to the customer

- 1 × Framework MicroSD Expansion Card, preloaded with your installer
- 1 × QR code card (printable insert) with activation URL
- 1 × single-page quickstart

### The provisioning flow

1. Customer scans the QR code → lands on `leger.dev/activate/<token>`, creates
   a Leger account (or logs in), sees on-screen instructions.
2. Customer inserts the Expansion Card into the Framework Desktop's
   expansion slot. Powers on.
3. Framework boots from the card. A thin Anaconda installer comes up.
4. Installer brings up the Wi-Fi adapter in **AP mode**, broadcasts an open
   network named `Leger-Setup-<token>`, and serves a captive portal at
   `http://192.168.100.1`.
5. Customer disconnects their phone from their normal Wi-Fi, connects to
   `Leger-Setup-<token>`. iOS/Android auto-detects the captive portal and
   opens it.
6. Captive portal asks: *"Enter the Wi-Fi network your Leger box should
   join, and the password."* Customer types.
7. Installer tears down AP mode, switches to client mode, connects to the
   real Wi-Fi, authenticates to `leger.run` with the activation token, pulls
   the latest harness image from Cloudflare/R2, installs, reboots.
8. Phone auto-reconnects to the real Wi-Fi. Customer lands in the Leger
   dashboard.

~10 minutes wall-clock, dominated by the harness image download.

### Why this works

- **No custom hardware.** Framework already manufactures, certifies, and
  distributes the expansion card.
- **No app required.** The captive portal is the standard IoT-onboarding
  pattern. Every smart plug, smart bulb, and printer setup in the last
  decade uses it. iOS and Android both auto-open captive portals on
  networks without internet.
- **Hardware-verified.** `iw phy` on a live Strix Halo Framework Desktop
  confirms AP mode is supported on all bands. The Wi-Fi stack can
  broadcast its own network in the pre-Anaconda environment. (See the
  [verification in later-evening-physical-object.md](../sodimo/secondweek/friday/later-evening-physical-object.md)
  if you want the exact output.)

### What it's honest about

- **The customer types their Wi-Fi password once.** Every install. The
  captive portal can't auto-fill from the phone's saved networks because
  neither iOS nor Android exposes saved Wi-Fi passwords to web pages (for
  good security reasons).
- **There's a Wi-Fi disconnect-reconnect dance.** Phone switches from home
  Wi-Fi → `Leger-Setup-<token>` → back to home Wi-Fi. Takes 60–90 seconds.
  Universal pattern, slightly awkward, rarely fails.
- **The UX is not premium.** It feels like setting up a router. Which, at
  this stage, is honest — you're installing a headless Linux server.

### Who it's for

**V0 is not the product. V0 is scaffolding.** It exists specifically to
enable operator-guided pilot installs — the first 3-5 paying customers
where you or an FDE is physically or remotely present, walking them
through setup. At that scale the captive-portal friction is absorbed by
the FDE; the customer may not even see it.

V0 does not go on the marketing site. No screenshots of the captive
portal. No "easy setup in minutes" copy. The website during the V0 period
says *"Leger is in managed rollout — contact us to schedule an
implementation."* No self-serve signup.

### BOM / cost at V0 scale

- Framework MicroSD Expansion Card at wholesale: ~€12–15
- 4GB micro SD card (you barely use 1 GB): ~€1
- Small retail box + QR insert + quickstart: ~€2
- Labor to flash and pack: ~€2

All-in per unit at pilot quantity (50–100): ~€20. Trivial relative to the
€5–10K implementation fee.

---

## V1 — Native app + BLE provisioning (the Roomba pattern)

### What it is

A Leger-branded native iOS and Android app that handles the first-boot
provisioning via Bluetooth Low Energy, plus ongoing device management,
push notifications, and (in V2) hardware-key authentication. **This is
the product.** V0 was the scaffolding; V1 is what the customer actually
experiences.

### What changes for the customer

1. Scan the QR code → App Store / Play Store deep-link → **Leger app**
   installs on the phone.
2. Customer opens the app, creates or logs into their Leger account. Enters
   their Wi-Fi password **once, in the app**. Saved to the app's local
   keychain.
3. Customer inserts the Expansion Card, powers on the Framework.
4. Framework boots, installer starts advertising over **BLE** (not AP
   mode). iOS detects the advertisement via Apple's Accessory Setup Kit;
   Android detects via Bluetooth.
5. Customer taps "Add device" in the app. Phone shows *"Set up [Leger
   Device]?"* — the same prompt pattern as pairing AirPods or a HomePod.
   Customer taps **Allow**.
6. App transfers Wi-Fi credentials to the installer over encrypted BLE.
   No disconnect-reconnect dance; the phone stays on real Wi-Fi the
   whole time.
7. Installer connects to Wi-Fi, fetches harness, installs. App displays
   live install progress on the phone.
8. When install completes, app deep-links into the dashboard (which the
   customer's Leger account is already authenticated for).

### What the app is (beyond provisioning)

Provisioning is one feature. The app is the admin's ongoing interface to
their infrastructure:

- **Monitoring + push**: disk pressure, failed health checks, certificate
  expirations, unusual outbound traffic — all surfaced natively, glanceable,
  one-tap-to-act.
- **Usage summaries**: *"Your team ran 47K LLM requests this week, saved
  ~€340 vs comparable cloud API spend."* The ledger story from
  [`00-why.md`](00-why.md) rendered for the admin's pocket.
- **Ops**: restart a quadlet, trigger a backup, approve a pending config
  change, invite a team member.
- **Multi-device**: SMBs that grow to 2–3 locations add boxes from the
  same app. After the first device's Wi-Fi password is saved, **device
  #2 onwards is truly tap-to-pair with no typing**.
- **The relationship**: direct channel to the Leger support team, release
  notes in plain language, onboarding checklist progress for new
  deployments.
- **Hardware-key authentication** (V2+, see below).

This is why V1 is 3–6 months of focused engineering, not 2 weeks. The
app is not a provisioning tool; it's the customer's primary infrastructure
interface, with provisioning as one gesture.

### What becomes technically feasible

- **BLE provisioning on iOS** is solved, not hypothetical. CoreBluetooth
  (central mode) is rock-solid on iOS 13+; Accessory Setup Kit (iOS 18+)
  provides the Apple-native "tap to pair" system prompt that matches
  AirPods / HomePod UX.
- **Android BLE** works via the standard BluetoothAdapter + BluetoothLeScanner
  APIs. Web Bluetooth also works in Chrome/Edge on Android as a lighter-weight
  fallback for early iterations.
- **NEHotspotConfiguration** on iOS lets the app programmatically add
  networks to the phone's known list and confirm reachability — useful for
  the "verify the box got online" confirmation step.

The iOS limitation that *does* apply: you **cannot** silently read the
phone's saved Wi-Fi passwords from the system keychain. The user enters
the password once, in your app, and your app stores it for subsequent
device setups. This is how Roomba, Nest, and Ecobee all work — the
"magical" part is that the user enters it only once per app install, not
never.

### Why the app is worth the investment

You were already going to need an app. An SMB admin's ongoing interface
to their infrastructure can be:

- **SSH + CLI.** Works for operators, fails for admins whose technical
  ceiling is the Google Workspace admin console.
- **Web dashboard only.** Works. Commodity feel. Doesn't support glanceable
  push, proximity-aware behavior, or native BLE interactions.
- **Native mobile app + web dashboard.** The premium default for B2B
  infrastructure products at your price point. Tailscale, Fly, Railway,
  Vercel all have native apps — they built them because the audience
  expects to manage infrastructure from their phone.

If the app has to exist for ongoing operations, building it now (informed
by what the first 3–5 V0 customers taught you) is not premature — it's the
right time. V0 is scaffolding; V1 is product.

### Engineering scope

- Custom Fedora installer image with BlueZ + a provisioning daemon baked
  in (the harness's installer counterpart — small, focused, written in Go
  or Rust for size)
- Pre-Anaconda BLE GATT server that accepts Wi-Fi credentials from the
  app, writes them into the kickstart path, hands off to Anaconda
- Native iOS app (Swift/SwiftUI) — 2–3 months of focused work
- Native Android app (Kotlin/Compose) — 2–3 months, can run in parallel
  with iOS if you have two engineers
- Shared backend work in `leger.run` for activation tokens, push
  notification infrastructure, device lifecycle
- End-to-end testing on actual Framework Desktop hardware

Realistic calendar: 4–6 months for a single engineer to ship both
platforms + backend, or 2–3 months with a small team.

### BOM

No hardware changes from V0. Same Framework MicroSD Expansion Card, same
2GB embedded MicroSD. The engineering investment is entirely in software
+ mobile apps. The reused Framework form factor continues to carry the
product's physical presence.

---

## V2 — The YubiKey pivot

### What it is

After the installer completes its initial job, the same physical device
**changes role**: from "thin installer" to "hardware security key
permanently bound to this customer's Leger deployment." The secure element
on the card now holds:

- The Vault unseal key (required to boot the Framework's encrypted services)
- A FIDO2 / WebAuthn authenticator credential (for logging into
  `dashboard.leger.run`)
- The encryption key for Leger's optional cloud backup of the customer's
  API keys and configuration secrets

### What this unlocks

- **Vault unseal on boot is physical.** Framework reboots → Vault starts
  sealed → queries for unseal key → device (if present) provides it via
  secure channel → Vault unsealed. If the device is removed (stolen
  machine, unauthorized relocation), Vault stays sealed on the next boot.
  Model: *"the key is in the drawer with the keys to the office."* SMB
  owners understand this immediately.
- **WebAuthn login replaces passwords.** Customer logs into
  `dashboard.leger.run` → WebAuthn prompt → device provides the signed
  assertion. No passwords, no TOTP, no password manager.
- **Cloud backup is customer-encrypted.** Leger offers opt-in encrypted
  backup of the customer's secrets and configuration to R2. The encryption
  key lives on the device; Leger cannot decrypt the backup even
  server-side. If the customer's Framework fails and they get a
  replacement, they plug in the device → it authenticates to Leger →
  decrypts the backup → restores to the new Framework.
- **API-key-rotation attestation.** When Leger rotates an external API
  key on the customer's behalf, the rotation event is signed by the
  device. Audit trail: *"this customer's key was rotated on this date,
  authorized by this hardware."*

### What makes this genuinely a moat

**The software is OSS. The device is not.** Anyone can fork Leger, run
harness on their own hardware, skip the services business entirely.
They cannot replicate the signed hardware that a specific customer
deployment is bound to. For an OSS-first business model
(see [`0A`](0A-skills-need-a-policy-layer.md)'s closing discussion of
product-vs-protocol), having one non-forkable component provides
strategic durability that pure OSS lacks.

### Engineering scope

- Custom PCB with **secure element** (ATECC608B or similar) and an NFC
  dual-interface chip (NXP NT3H2211), potentially in the Framework
  Expansion Card form factor via partnership
- Firmware implementing CTAP2 (FIDO2/WebAuthn) — open-source starting
  points exist (solokeys/solo2)
- Vault plugin for "HSM-backed unseal via USB-attached device" — does
  not exist today in Vault upstream; build as a plugin or a sidecar
  service
- Integration with `leger.run`'s WebAuthn flow and the encrypted-backup
  design
- FIDO L1 certification (~$10–30K, optional but credibility-relevant for
  institutional customers)
- Firmware-update path that handles security-patched firmware over the
  device's lifecycle

Realistic calendar: 9–18 months from V1 ship. This is the "real hardware
product" phase. Do not start it before V1 is shipping and 10–20 paying
customers have validated the demand.

### BOM at V2 production scale

~1000-unit run, per-unit:

- USB 3.2 Gen 2 controller + 128GB flash: €8–12
- NXP NT3H2211 NFC dual-interface: €1.50
- ATECC608B secure element: €1.50
- STM32L0-family MCU: €1
- PCB + antenna: €1–2
- Custom casing (Framework-matched design language): €2–5
- Assembly + test + programming: €2–4
- Total: ~€18–25 per unit

With NRE (firmware development, PCB design, tooling, certs) amortized
across the first 1000 units: ~€35–50 all-in for the first production run.
Subsequent runs drop to near-BOM once NRE is paid down.

---

## Staging, in one view

| Phase | Timeline | Ship | Scope |
|---|---|---|---|
| V0 | weeks 1–4 after first flash | Framework MicroSD Expansion Card + embedded installer image + QR card | Internal scaffolding; FDE-guided pilots only |
| V1 | months 3–9 | Native iOS + Android apps, BLE provisioning, web dashboard, push, ongoing monitoring | **The product.** Self-serve purchase opens at V1 ship. |
| V2 | months 9–18+ | Custom PCB with secure element + FIDO2 firmware + Vault integration + encrypted backup | Hardware moat; justified after V1 validates demand |

Each phase is shippable on its own. Each phase informs the next with real
customer data. None of the phases require committing to the phases after.

---

## The Framework partnership angle

Running through all three phases: Framework Computer Inc. has been
actively looking for compelling business narratives for the Framework
Desktop (their first non-modular-repairable product; they need
justifications for it that match their brand). A Leger-branded expansion
card SKU in Framework's retail store is a partnership that delivers
concrete value to both sides:

**Leger gets**: manufacturing (Framework's supplier relationships),
retail distribution (framework.computer has pre-existing traffic),
brand halo (buyers see your card next to official Framework accessories
and infer legitimacy), and — in V2+ — the possibility of Framework
Desktops shipping with harness pre-installed at the factory, which is
the only path to *truly* eliminating the one-time BIOS trip (see
[`0E`](0E-the-bios-trip.md)).

**Framework gets**: a concrete "what you can do with the AI Max SKU
besides gaming" story, enterprise/SMB case studies they're missing,
developer-ecosystem credibility, a validated third-party use of the
expansion-card thesis beyond storage cards.

Realistic partnership timeline: 6–12 months of relationship-building and
negotiation before anything ships co-branded. The partnership is an
accelerator on top of a standalone product, not a dependency. V0 and V1
both work without Framework's cooperation. V2 with Framework integration
is the end state if it materializes.

---

## What this changes about everything else

- **[`00-why.md`](00-why.md)'s economics still hold** — the hardware math
  and 90-day wager are about the Framework Desktop itself, not the
  installer. V0 doesn't change the ROI story for pilot customers; V1
  makes the story legible to self-serve customers.
- **[`0A`](0A-skills-need-a-policy-layer.md)'s policy/skill separation
  applies to the app** — the app is a skill *consumer*, not a policy
  runner. The runbooks and state files still live on the device and in
  `leger.run`; the app reads them.
- **[`0C`](0C-harness-vs-dynamic-layer.md)'s LTS-harness is what makes
  the V0-to-V1-to-V2 arc sustainable** — if the OS were fast-moving, no
  installer strategy would work because `:latest` would mean something
  different every week. The harness being slow-moving is the property
  that lets a multi-month hardware roadmap be credible.

---

## The discipline this chapter exists to enforce

Do not confuse V0 with the product. V0 is scaffolding for FDE-guided
pilots, and marketing it as the product damages the brand in ways that
are hard to recover from. V1 is the product: the native app, the BLE
provisioning, the ongoing mobile interface. V2 is the hardware moat that
makes the OSS-with-services business durable.

Ship V0 fast. Ship V1 properly. Ship V2 when the data justifies the
investment. Never the reverse.
