# 0C — Harness vs dynamic layer

*Why the bootc image is deliberately slow-moving, what gets installed
last-mile, and where chezmoi belongs in a world that already has Leger and
quadlets and systemd.*

---

The single sharpest design decision in this pipeline is a split nobody
coming from a traditional Linux background would draw by default:

- **The bootc "harness" image is an LTS substrate.** Updated monthly-ish,
  reboot-gated, deliberately conservative. Contains kernel, systemd,
  firmware, core tools that the provisioning runbook depends on.
- **Leger-managed services are a dynamic product layer.** Updated without
  reboot, health-checked, auto-reverted on failure. Contains the
  business-critical quadlets the customer actually uses (Twenty CRM, local
  LLM stack, email server, etc.).
- **chezmoi is operator dotfiles only.** Never customer-facing. Your shell,
  your editor, your aliases. If it breaks, you fix it. It never runs on a
  customer's device except maybe during a hands-on debug session.

This split is what makes the LTS-harness assumption work. Without it, you
end up with the trad-Linux problem: every customer's box is a unique
snowflake because every operator's dotfiles ended up in the provisioning
path, and nobody knows what changed between any two deployments.

---

## The LTS-harness assumption

The harness image is the OS. It's a Fedora bootc container that lives at
some public registry tag like `ghcr.io/your-org/harness:latest`. It gets
rebuilt when, and only when, something in its slow-moving contents needs to
change:

**Changes when:**

- Kernel CVE requires a bump
- Strix Halo firmware or driver changes matter
- A core tool (OpenTofu, cloudflared, the Leger binary itself) gets a
  breaking version you need
- Security patches to anything in the immutable layer

That's maybe monthly, maybe quarterly. Fast enough to stay secure, slow
enough that customers don't see constant reboots.

**What goes in the image:**

- Kernel, systemd, firmware, kernel modules for Strix Halo
- Core OS: podman, systemd timers, NetworkManager, base utilities
- Tools that must be present on a cold-booted machine with no network:
  opentofu, gh CLI, cloudflared, mise, just, tailscale binary (if you want
  it baked)
- The Leger binary itself
- Vault (once secrets-era setup lands)
- Signed CA roots, cosign pubkey for the image's own signature chain
- Any `/usr/lib/bootc/install/*.toml` install-time configuration
- Any `/usr/lib/systemd/system/*.service` that's part of the OS baseline

**What is deliberately NOT in the image:**

- Node, Python, Go runtimes → installed via `mise` post-boot
- Claude Code → `npx @anthropic-ai/claude-code` or a `mise` pin
- Language-specific tooling that version-drifts weekly (wrangler, specific
  Terraform providers)
- Anything that would force a harness rebuild for a minor version bump
- Application quadlets for Twenty CRM, AI models, etc. → managed by Leger
- Per-team or per-user config → managed by Leger, not baked

This split is what makes "the harness is LTS" honest. If claude-code lived
in the image, every Anthropic npm release would force an image rebuild.
Instead it gets installed once, on first operator session, via `npx`. That's
where its updates live.

---

## The dynamic Leger-managed layer

Business-critical quadlets never live in chezmoi. They live in Leger's own
state directory (e.g., `/var/lib/leger/quadlets/` or
`~/.config/containers/systemd/` managed by Leger), installed by
`leger deploy install`, health-checked by Leger, rolled back by Leger.
chezmoi doesn't know they exist.

This means the SMB admin who uses Leger *never touches dotfiles for
business-critical stuff*. They click "install Twenty CRM" in the webapp
(or the admin runs `leger deploy twenty-crm`), Leger writes a quadlet,
Leger monitors it, Leger reports status.

### The decision tree

Where does a given configuration artifact belong? Ask in this order:

1. **Is it part of Leger's curated store catalog?** → Leger manages it.
   Never goes in chezmoi. Ships with auto-revert, health monitoring,
   dashboard visibility.

2. **Is it infrastructure for Leger itself** (Vault, monitoring agent,
   update timer, the leger daemon)? → Ships in the harness image as default
   systemd units + quadlets, updated via `bootc upgrade`.

3. **Is it operator dev environment on your own machine** (your shell, your
   editor, your aliases, your personal podman containers)? → chezmoi. You
   use chezmoi because you know it, it works, and it's never customer-facing.

4. **Is it "I want to run this specific container on my customer's machine
   for my own reasons"?** → Stop and reconsider. Either it's business-critical
   and should be a Leger store item (add it to the catalog), or it's not
   and shouldn't exist on the customer's device.

5. **Is it a systemd unit that's not containerized** (a backup timer, a
   log rotation)? → If Leger needs it, ship in the harness image. If it's
   personal, chezmoi. The customer device should never have "operator's
   personal systemd units" — that's operator-environment contamination.

### How auto-revert works per layer

| Layer | Failure handling |
|---|---|
| systemd (unit level) | `Restart=on-failure`, `RestartSec=10s`, `StartLimitBurst=3`, `StartLimitIntervalSec=60s`. If a service flaps 3 times in 60 seconds, systemd stops. Built-in. |
| podman (container level) | Healthchecks in the quadlet: `HealthCmd=`, `HealthInterval=30s`, `HealthOnFailure=kill` or `stop`. Container that fails gets reaped. Built-in. |
| Leger (service level) | Watches for "stopped after repeated failures" state. Reverts the quadlet to the last-known-good snapshot. Reports to dashboard. **This is custom.** |
| bootc (OS level) | `bootc rollback` reverts to previous image deployment. Manual or triggered by a health-gate policy. |

The Leger layer is the one that earns its keep. Every `leger deploy install`
snapshots the previous quadlet to `/var/lib/leger/snapshots/<service>/<timestamp>.quadlet`.
On repeated systemd failures, Leger copies the snapshot back,
`systemctl daemon-reload`, starts the service, and POSTs an event:

```json
{
  "customer": "acme",
  "service": "twenty-crm",
  "event": "auto-reverted",
  "from_version": "v2.4.1",
  "to_version": "v2.4.0",
  "timestamp": "2026-05-03T14:22:03Z"
}
```

Dashboard renders: *"Twenty CRM was rolled back from v2.4.1 to v2.4.0 due to
repeated failures."*

**Circuit breaker.** Auto-revert is dangerous if taken too far. If the
snapshot being reverted to also fails (e.g., underlying infrastructure
problem, not code problem), you can ping-pong. After 2 auto-reverts within
an hour, Leger stops auto-reverting and switches to *"manual intervention
required"* on the dashboard. Admin sees a red card: *"Twenty CRM has failed
three times in an hour, we've stopped trying, click to investigate."*

---

## Why this matters for the setup guide

The setup guide builds the harness side. Chapters 01–04 produce a running
OS image on the target. Everything after that — the Leger layer, the
customer-facing webapp, the playbook catalog, the per-customer quadlets —
is *outside* the scope of this guide, because it's outside the scope of
the harness.

Which is the whole point of the split. The setup guide becomes tractable
because it only has to answer questions about the slow-moving layer.
Questions like *"which CRM should we include?"* or *"which LLM models
ship by default?"* or *"how do customers configure AI Gateway?"* all
belong to the Leger layer, not the setup pipeline.

If those questions were answered inside the image, every customer would
need a custom harness image and the monthly-ish update cadence would
collapse. The split preserves the update cadence.

---

## The update loop is reboot-gated (and that's the point)

`bootc upgrade` is a transactional, atomic, rollback-able operation. It
stages the new image as the next deployment, and `systemctl reboot`
activates it. You can't hot-patch the OS. You accept that in exchange for:

- **Atomicity.** The new OS either comes up fully or you boot the old one.
  No half-updated state.
- **Rollback.** `bootc rollback` returns to the previous deployment. First
  pass at a regression is a reboot, not a reinstall.
- **Auditability.** Every deployment is an image digest you can name.
  "Which OS is this box running?" is a one-line answer.

This is the model from Fedora Silverblue, uBlue, Bazzite, CoreOS. It's
deliberate, not a limitation. The trade is no hot-patching for everything
else being cleaner.

The daily operational reality:
`bootc-fetch-apply-updates.timer` fires on whatever cadence you configure
(weekly is typical), pulls the new image if the digest changed, stages it
as the next deployment. Reboot applies it. That's the entire OS update
loop.

---

## The install mechanism does not change any of this

[`0B-iso-vs-pxe-vs-tunnel.md`](0B-iso-vs-pxe-vs-tunnel.md) covers three
install mechanisms — USB, PXE+local, PXE+tunnel. The update loop is
identical after install regardless of which you chose. USB-installed boxes
and PXE-installed boxes both `bootc upgrade` from the same registry on the
same timer. The first-mile choice has no bearing on ongoing operations.

So: choose the install mechanism based on install-ergonomics. The update
loop is the bootc property, not the installer property.

---

## What belongs where, in one table

| Thing | Layer | Update mechanism | Failure handling |
|---|---|---|---|
| Kernel / firmware / drivers | Harness image | `bootc upgrade` + reboot | `bootc rollback` |
| OpenTofu, gh, cloudflared, leger binary | Harness image | `bootc upgrade` + reboot | `bootc rollback` |
| Claude Code | Last-mile (`npx` / `mise`) | `npm`-style version pin update | Pin to last-known-good |
| Language runtimes (node, python, go) | Last-mile (`mise`) | `mise install`/`mise use` | Pin to last-known-good |
| Twenty CRM, local LLM stack, etc. | Leger-managed quadlets | `leger upgrade <service>` | Auto-revert to snapshot |
| Vault (secrets store) | Harness image | `bootc upgrade` + reboot | `bootc rollback` |
| Operator's shell/editor/aliases | chezmoi | `chezmoi apply` | Manual |
| Per-customer playbook outputs | Leger state dir | `leger policy apply` | Terraform state restore |

---

## What this means for `00-why.md`'s economics

The two-month payback in [`00-why.md`](00-why.md) assumes the operator's
time isn't spent firefighting image rebuilds every week. That assumption
holds only because of this split. If every new Claude Code version, every
new model, every new wrangler release forced a harness rebuild, the
operational overhead would swamp the savings.

The LTS-harness is the property that makes the economics sustainable,
not just the hardware.
