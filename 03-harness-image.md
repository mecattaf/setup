# 03 — The harness bootc image

**Goal:** build the OS image the target will boot into. This is the slow-moving
substrate described in [`0C`](0C-harness-vs-dynamic-layer.md) — everything
dynamic lives above it.

**Prerequisite:** a Fedora machine with `podman` to build the image, and a
public container registry (GHCR, Quay, Docker Hub) to host it. `bootc install`
does not yet support authenticated registries cleanly — your image must be
publicly pullable, or you must bake `/etc/ostree/auth.json` into it.

---

## What a bootc image is

A regular OCI container image that happens to be a bootable operating system.
It's built with a `Containerfile` (Dockerfile), pushed to a registry, pulled
and laid down on disk by `bootc install`. Once installed, `bootc upgrade`
pulls new digests from the same registry, stages them as the next deployment,
and reboot activates. Atomic, transactional, rollback-able.

Upstream: https://containers.github.io/bootc/ and the Fedora docs at
https://docs.fedoraproject.org/en-US/bootc/.

If you've used Silverblue, uBlue, or Bazzite before, this is the same model,
just expressed as "you build the image yourself" instead of "you consume
someone else's."

---

## The Containerfile skeleton

Start with this scaffold, in a fresh git repo called `harness` (or whatever
you want to name your image):

```dockerfile
# Containerfile
FROM quay.io/fedora/fedora-bootc:44

# ───── Hardware enablement (Strix Halo / AMD Ryzen AI Max 300) ─────
# Kernel arguments that have to be present at install time.
COPY <<'EOF' /usr/lib/bootc/kargs.d/10-strix-halo.toml
kargs = [
  "iommu=pt",
  "amdgpu.gttsize=126976",
  "ttm.pages_limit=32505856"
]
EOF

# ───── Install-time configuration ─────
COPY <<'EOF' /usr/lib/bootc/install/50-harness.toml
[install]
root-fs-type = "btrfs"

[install.filesystem.root]
type = "btrfs"
EOF

# ───── Signed-pull enforcement for upgrades ─────
# Pubkey used to verify bootc upgrade pulls.
COPY cosign.pub /usr/share/pki/containers/harness.pub

COPY <<'EOF' /etc/containers/policy.json
{
  "default": [{"type": "reject"}],
  "transports": {
    "docker": {
      "ghcr.io/<your-org>/harness": [{
        "type": "sigstoreSigned",
        "keyPath": "/usr/share/pki/containers/harness.pub",
        "signedIdentity": {"type": "matchRepository"}
      }],
      "": [{"type": "insecureAcceptAnything"}]
    }
  }
}
EOF

COPY <<'EOF' /etc/containers/registries.d/harness.yaml
docker:
  ghcr.io/<your-org>/harness:
    use-sigstore-attachments: true
EOF

# ───── Baseline tools (the slow-moving loadout) ─────
RUN dnf install -y \
      git jq yq \
      tailscale \
      podman-compose \
    && dnf clean all

# ───── OpenTofu from the official repo ─────
RUN dnf config-manager addrepo --from-repofile=https://get.opentofu.org/rpm/fedora.repo \
    && dnf install -y tofu \
    && dnf clean all

# ───── The leger binary (placeholder; replace with your release artifact) ─────
# COPY leger-linux-amd64 /usr/local/bin/leger
# RUN chmod 0755 /usr/local/bin/leger

# ───── Base services ─────
RUN systemctl enable sshd bootc-fetch-apply-updates.timer

# ───── Validate ─────
RUN bootc container lint
```

A few decisions worth understanding, not copying blindly.

---

## What belongs in the image

Apply the decision tree from
[`0C — Harness vs dynamic layer`](0C-harness-vs-dynamic-layer.md). Specifically:

### Must be in `/usr`

- **Kernel modules and firmware** for the target hardware. `fedora-bootc:44`
  ships `linux-firmware` by default; verify Strix Halo firmware blobs are
  present (`rpm -qa | grep linux-firmware` inside the image).
- **Install-time config**: anything under `/usr/lib/bootc/install/*.toml`.
  Filesystem choice, partitioning hints, kargs.
- **Kernel arguments** that have to be set at install time: `/usr/lib/bootc/kargs.d/*.toml`.
  The three lines above are required for `gpt-oss-120b` at 65k context on
  Strix Halo — without them, the model OOMs. If you're not running large
  MoE models, trim.
- **Core CLI tools that the provisioning runbook depends on**: `gh`,
  `cloudflared`, `tofu`, `mise`, `just`, `tailscale`, and your `leger`
  binary. These exist so that a cold-booted machine with nothing but this
  image can already run the first `leger policy apply` without bootstrapping
  tooling at runtime — which is the whole point of an immutable OS.
- **Signed-pull policy and key**: `containers-policy.json`,
  `registries.d/*.yaml`, and the cosign pubkey at a stable path under
  `/usr/share/pki/containers/` or `/etc/pki/containers/`.
- **Base systemd units** enabled at build time: `sshd`, the bootc timer,
  NetworkManager (inherited from base).

### Must NOT be in `/usr`

- **Any secret of any kind.** Not authkeys, not tokens, not SSH private
  keys, not cloud credentials. `/usr` is world-readable in the container
  registry; anything there is public. Secrets live in `/etc` (deployed by
  the kickstart or by Leger), or in the on-device Vault.
- **Claude Code.** Install last-mile via `npx @anthropic-ai/claude-code` or
  a `mise` pin. Anthropic ships updates faster than your harness rebuild
  cadence; baking it means every Anthropic release forces an image rebuild.
- **Language runtimes** (Node, Python, Go). Install via `mise` on first
  operator session. Same argument as Claude Code.
- **Application quadlets** — Twenty CRM, LiteLLM, Qdrant, Caddy, whatever.
  These are the Leger-managed layer, updated via `leger upgrade <service>`
  without requiring a reboot. Baking them in means every app update needs
  an OS reboot.
- **Per-customer or per-deployment configuration.** Anything that would
  differ between two installations belongs in the kickstart (for install-
  time state) or in Leger state (for runtime state).

### The test: would it force a rebuild?

If a change to X would require you to rebuild and re-publish the harness
image, then X is an image-layer concern. If X can be updated independently,
it doesn't belong here.

---

## First-boot services

Any systemd unit that should run the first time the target boots and then
never run again belongs here. The pattern that actually works across bootc
upgrades:

```ini
# /usr/lib/systemd/system/harness-firstboot.service
[Unit]
Description=Harness first-boot setup
After=network-online.target
Wants=network-online.target
ConditionPathExists=!/var/lib/harness/firstboot.done

[Service]
Type=oneshot
ExecStart=/usr/libexec/harness-firstboot
ExecStartPost=/bin/sh -c 'install -D /dev/null /var/lib/harness/firstboot.done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

The `ConditionPathExists=!` gate is the canonical idempotency pattern. `/var`
is preserved across `bootc upgrade`, so the sentinel survives — the unit
will NOT re-run after the first successful execution, even if you rebuild
and ship a new image. `systemd` documents this pattern directly
(`systemd.unit(5)`). `systemctl disable` is the wrong tool here because
`/usr` is read-only on bootc; the condition gate is the right one.

Enable it at build time:

```dockerfile
RUN systemctl enable harness-firstboot.service
```

---

## Building the image

Two common approaches; pick whichever matches your existing workflow.

### Option A — `podman build` (simple)

```bash
podman build -t ghcr.io/<your-org>/harness:dev .
podman images ghcr.io/<your-org>/harness
```

Lint it:

```bash
podman run --rm ghcr.io/<your-org>/harness:dev bootc container lint
```

Tag and push:

```bash
podman tag ghcr.io/<your-org>/harness:dev ghcr.io/<your-org>/harness:latest
podman push ghcr.io/<your-org>/harness:latest
```

### Option B — mkosi (more structured)

`mkosi` is Fedora's image-builder-of-choice for more complex images —
separate config files per concern, reproducible builds, CI-friendly. Upstream
at https://mkosi.systemd.io/. Use this if the Containerfile is accumulating
too much COPY+RUN noise.

---

## Signing with cosign in CI

The `policy.json` above requires signatures on every pull. You need to sign
each published digest. Standard GitHub Actions pattern (excerpted — full CI
lives in a separate file in your harness repo):

```yaml
- uses: sigstore/cosign-installer@v3
- name: Sign the image
  env:
    COSIGN_EXPERIMENTAL: 1
  run: |
    cosign sign --yes \
      ghcr.io/<your-org>/harness@${{ steps.build.outputs.digest }}
```

Keyless signing (OIDC via GitHub Actions → Fulcio → Rekor) is the
low-friction option. If you go keyless, adjust `policy.json` to match the
Fulcio identity rather than `keyPath` — see
https://github.com/sigstore/cosign/blob/main/doc/cosign_policy.md.

Verify from a third machine before committing to it:

```bash
cosign verify \
  --key cosign.pub \
  ghcr.io/<your-org>/harness:latest
```

If this fails with `no signatures found`, the CI isn't actually signing.
Fix that first — nothing downstream (`--enforce-container-sigpolicy`,
`bootc upgrade` signature checking) works until signing works end-to-end.

---

## Testing the image before committing to it

Do not publish `:latest` without running it at least once.

### Sanity: does it boot in a VM?

```bash
# Build an anaconda installer ISO from the image
sudo podman run --rm -it --privileged --pull=newer \
  --security-opt label=type:unconfined_t \
  -v ./output:/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  --type anaconda-iso \
  --use-librepo=True \
  ghcr.io/<your-org>/harness:dev
```

Boot the resulting ISO in QEMU. Confirm it installs, reboots, comes up with
the services you expected enabled.

### Deeper: does it pass `bootc container lint`?

```bash
podman run --rm ghcr.io/<your-org>/harness:dev bootc container lint
```

Lint catches common bootc-specific mistakes — missing kernel packages,
broken symlinks, badly-configured systemd units, unsupported filesystem
choices. Expected output: `no errors`. Anything else, fix before pushing.

### Practical: does it pull cleanly as an anonymous user?

```bash
podman logout ghcr.io
podman pull ghcr.io/<your-org>/harness:latest
```

If this 401s, the registry repository is private. Either make it public, or
bake `/etc/ostree/auth.json` into the image for authenticated pulls. The
former is what Chapter 02's kickstart assumes.

---

## Tagging strategy

Two tags worth maintaining:

- `:latest` — what `bootc upgrade` pulls by default. Move it on every
  successful CI run.
- `:YYYYMMDD-<short-sha>` — one immutable tag per build, for pinning and
  rollback. The digest is the real immutable identifier, but the dated tag
  is human-readable.

`:dev`, `:pr-N`, or similar transient tags for CI testing, but never for
anything a customer's `bootc-fetch-apply-updates.timer` could pick up.

---

## The update cadence question

How often should you publish a new `:latest`?

The honest answer from [`0C`](0C-harness-vs-dynamic-layer.md): monthly-ish.
Fast enough to stay secure, slow enough that customers don't see constant
reboots. Bump when:

- Kernel CVE requires it
- Strix Halo firmware or driver update matters
- A core tool (OpenTofu, cloudflared, the leger binary) gets a breaking
  version you need
- Security patches to anything in the image

Do NOT bump on every Claude Code / wrangler / model-of-the-week release —
those live outside the image for exactly this reason.

The default `bootc-fetch-apply-updates.timer` fires weekly. Match your
publish cadence to the update cadence customers will feel. Weekly publishes
on a weekly check is fine; daily publishes on a weekly check means most
publishes are silently superseded before customers ever see them, which is
also fine but wasteful.

---

## What you end up with

A public image at `ghcr.io/<your-org>/harness:latest`, with:

- A known-good digest, signed by your cosign key
- A documented `policy.json` the image itself enforces on upgrades
- A baseline loadout of tools the provisioning runbook can assume are present
- Hardware tuning for the specific targets you deploy to
- First-boot services that run once and self-disable
- Reproducible CI that rebuilds it from source on every push

The kickstart in [Chapter 02](02-root-node-flasher.md) now has a real image
to install. Swap `quay.io/fedora/fedora-bootc:44` for
`ghcr.io/<your-org>/harness:latest` in the `bootc --source-imgref=` line
and reflash.

---

## Common pitfalls

**`autopart` fails at install time.** Don't use it. Use the explicit
`reqpart --add-boot` + `part /` form from Chapter 02's kickstart. Autopart
on bootc + `/boot/efi` has a known-bad interaction.

**Forgetting to enable services at build time.** `systemctl enable` during
`RUN` is required; `[Install] WantedBy=...` in the unit file alone is not
enough — `systemctl preset` wouldn't necessarily catch new units unless
you ship a preset file. Explicit enable is safest.

**Baking Claude Code / wrangler / model files into the image.** Everyone
does this the first time and regrets it the first time they have to rebuild
the image because `npm` shipped a patch release. Keep last-mile tools out.

**Unsigned `:latest` in production.** Means `bootc upgrade` either fails
(if the policy is enforced) or silently drops signature verification
(if you were lenient). Both are worse than knowing. Verify `cosign verify`
passes from a third machine before marking a build "done."

**Private registry without `auth.json`.** `bootc install` does not yet
handle authenticated registries cleanly. Public registry is the low-friction
answer; if you must be private, bake `/etc/ostree/auth.json` into the image
(the image needs to be able to auth to pull *itself* on upgrade).

---

Next chapter (pending): **04 — First flash & headless handoff.** Captures
the post-install MAC reservation, the `efibootmgr -v` hex capture, the SSH
`~/.ssh/config` pattern, and the `reflash.sh` one-liner that closes the
dev loop.
