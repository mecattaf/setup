# 0B — ISO vs PXE vs Tunnel

*Why this guide standardizes on PXE even though USB ISOs are simpler, what
the Cloudflare Tunnel upgrade path looks like, and when each approach is
actually right.*

---

Three install mechanisms exist for a bootc-on-Strix-Halo setup. They look
similar on the surface ("get OS bits onto disk") but occupy genuinely
different positions in the design space.

---

## 1. USB ISO — the universal fallback

You build an ISO that contains the Anaconda installer plus an embedded copy
of the bootc image. Burn to a USB stick. Plug into the target. Boot. Install.
Remove. Reboot into installed OS.

**Unique property: zero network dependency at install time.** The ISO
contains everything. The target can be on a network with no DHCP, on
captive-portal wifi, in a Faraday cage. Network only becomes relevant *after*
install when the running system wants to `bootc upgrade`.

This is the "it works in a bunker" property, and it is not overrated. For
provisioning a machine at a new site where you don't know the network state,
this is the most reliable path that exists.

**Cost:**

- 11.8 GB of USB stick (an 8GB image plus Anaconda + Fedora baseline)
- Physical trip to insert the stick
- Rebuild the ISO every time the bootc image changes meaningfully

That third cost is less bad than it looks. See
[`0C-harness-vs-dynamic-layer.md`](0C-harness-vs-dynamic-layer.md): the
harness image is deliberately slow-moving, so ISO rebuilds are monthly-ish,
not daily.

---

## 2. PXE + local primary — the current plan

A Fedora machine nearby (the "root node" — see Chapter 02) serves DHCP +
TFTP + HTTP on a direct ethernet link to the target. Target powers on, PXE
ROM requests network boot, installer kicks off, kickstart drives the whole
thing unattended, `bootc install` pulls the image from a public registry,
reboot.

**Unique property: zero physical intervention per-flash after the first BIOS
trip.** You plug a cable once. You press a button once (or you don't — see
Chapter 02 on `efibootmgr --bootnext`). Reflashing becomes a pure software
operation.

**Cost:**

- Requires a local machine on the same L2 network as the target, running
  DHCP + TFTP + HTTP
- Requires you (or an agent on your behalf) to be physically at that local
  machine to operate it
- Doesn't work if the target is somewhere else

This is what Chapter 02 covers. For a home/office with one or a few targets,
and a developer iterating on the bootc image frequently, this is the right
trade.

---

## 3. PXE + Cloudflare Tunnel — the upgrade path

Same stack as (2), but the HTTP-serving side moves behind `cloudflared`. iPXE
fetches `boot.ipxe` and `ks.cfg` from `https://flasher.yourdomain.com` via
Cloudflare's edge; target machine never talks directly to your primary over
the open internet.

**Unique property: reflashing a machine that isn't local.** The target can be
at another site, on a LAN you don't control, and you can still push an OS
update to it by `git push` + power cycle. The local side requires only a
tiny PXE handoff appliance serving DHCP + TFTP (~1 MB of signed iPXE binary).
Everything substantial comes from Cloudflare.

The "tiny PXE appliance" is genuinely small — a Raspberry Pi 3 running
dnsmasq with a fixed config. No nginx, no kickstart rendering, no artifact
caching, no maintenance, no updates. It's boot-bootstrap hardware. You ship
it to a new site, someone plugs it in, the Framework on that LAN network-boots
off Cloudflare.

**Cost:**

- Cloudflare account + tunnel + DNS config (one-time)
- cloudflared becomes a moving part on the primary; if it dies, no flashes
- Still requires DHCP + TFTP on the target's LAN because firmware can't
  do HTTPS before iPXE runs — that bootstrap gap doesn't disappear
- Cloudflare free-tier egress is fine for text (`boot.ipxe`, `ks.cfg`), but
  if you served the 11.8 GB ISO over a free-tier tunnel you'd hit their
  acceptable-use policy. Fine for what tunnel is for; not for hosting ISOs.

For artifacts, **R2 is the right tool** — S3-compatible object store with
no egress fees, fronted by Cloudflare's CDN. Store `vmlinuz`, `initrd.img`,
and potentially `boot.ipxe` itself in R2; iPXE fetches over HTTPS directly
from the CDN. Tunnel only carries dynamic content (per-machine rendered
kickstarts).

---

## The "works when..." matrix

| Scenario | USB ISO | PXE + local | PXE + tunnel |
|---|---|---|---|
| Target on your LAN with primary | ✓ | ✓ | ✓ (overkill) |
| Target on a remote LAN with DHCP you don't control | ✓ | ✗ | ✓ (needs tiny PXE appliance at the site) |
| Target on captive-portal wifi | ✓ | ✗ | ✗ |
| Target on no network at all | ✓ | ✗ | ✗ |
| Per-machine custom config (per-MAC kickstart) | Hard (rebuild ISO) | Easy (rendered kickstart) | Easy (Worker) |
| Dev loop: "I changed the OS image, flash :latest" | Slow (rebuild ISO) | Fast (`git push` → power cycle) | Fast (`git push` → power cycle) |

The three approaches are not redundant. They serve genuinely different
scenarios, and the right answer at any given time is *probably multiple of
them simultaneously*:

- **USB ISO** for the initial bootstrap at a new site when you don't know
  what the network will be. First install only. Possibly the only way to
  reach initial state in some environments.
- **PXE + local** as your current dev setup — fastest iteration, least
  infrastructure.
- **PXE + tunnel** once a second physical location exists. Same architecture
  minus the local-only constraint.

---

## HTTPS — what changes at each layer

HTTPS end-to-end for the PXE flow reveals a lot about what HTTPS actually is.
The current flow, simplified:

```
Framework firmware (PXE ROM)
  → DHCP           (UDP, no encryption possible)
  → TFTP ipxe-shim.efi   (UDP, no encryption possible — but signed)
  → iPXE runs
  → HTTP boot.ipxe       (plaintext, local-only today)
  → HTTP ks.cfg          (plaintext, local-only today)
  → HTTP kernel+initrd   (plaintext, local-only today)
  → bootc pulls ghcr.io/your-org/harness       (HTTPS, already)
```

The bootc image pull is already HTTPS — the registry only serves over TLS.
The plaintext layers are all local-only: primary → target over your direct
ethernet cable. Attack surface is "someone with physical access to your
cable," i.e., your house.

### What becomes HTTPS when tunnel-ified

```
Framework firmware (PXE ROM)
  → DHCP          (still local, still plaintext — unavoidable)
  → TFTP ipxe-shim.efi   (still local, still TFTP — but payload is Microsoft-signed,
                          so plaintext ≠ tamper-able)
  → iPXE fetches boot.ipxe  from  https://flasher.yourdomain.com
  → iPXE fetches kernel/initrd from Fedora mirror over HTTPS (native)
  → Anaconda fetches ks.cfg  from  https://flasher.yourdomain.com
  → bootc pulls ghcr.io     (already HTTPS)
```

All internet-crossing fetches are TLS. DHCP and TFTP stay local because they
happen before iPXE is running — the firmware can't do HTTPS, that's literally
why iPXE exists.

### iPXE trusts Cloudflare's cert by default

The signed-shim iPXE bundle from the GitHub releases *includes a default CA
trust store* (derived from Mozilla's). Fetching
`https://flasher.yourdomain.com/...` just works, assuming Cloudflare hands
out a publicly-valid cert (which the tunnel does automatically).

If you were using a self-signed cert or a private CA, you'd have to embed
your CA into a custom-built iPXE, which means compiling iPXE yourself, which
means losing the Microsoft-signed shim, which means Secure Boot has to come
off. **"Just use Cloudflare" is dramatically simpler than "run my own CA"**
— the former uses CAs iPXE already trusts.

### The TFTP bootstrap gap is unavoidable

TFTP cannot be made HTTPS. It's the bootstrap gap. The first ~200KB of code
the Framework executes is delivered in plaintext UDP. That code is
Microsoft-signed (`ipxe-shim.efi`), so tampering would break the signature
and Secure Boot would reject the binary. In practice the TFTP layer is
**tamper-evident even in plaintext**, because the payload is signed. It's
not *confidential* — an observer sees which binary was fetched. That's fine;
it's a public binary.

This is the same pattern in every network-boot architecture ever. You cannot
avoid the plaintext bootstrap. You can only shrink it and make sure the
handoff to the encrypted layer is secure. iPXE's built-in CA trust store is
the secure handoff.

---

## "How is this different from `bootc switch ghcr.io/ublue/bazzite`?"

A reasonable question: if the endpoint is "OS pulled from a registry URL,"
why not just USB-boot a generic live image and run `bootc switch` at the
CLI?

They look similar at the end. They're architecturally different at the
beginning.

**`bootc switch`** runs inside an already-booted OS. You've got a working
kernel, initramfs, network stack, DNS resolver, TLS, the `bootc` binary,
etc. It's an application-level operation. The "hard problems" are solved
because you already have a Linux system in front of you.

**PXE + tunnel** runs in an environment where the target has *nothing* yet.
No OS, no userspace, no TLS, no DNS — just firmware with a PXE ROM. Every
layer of the stack is solving a piece of the bootstrap:

- DHCP → "how do I get an IP with no userspace"
- TFTP → "how do I fetch a file with no TCP stack, only UDP"
- `ipxe-shim.efi` → "a signed bootloader Secure Boot accepts"
- iPXE → "now I have TCP + TLS + DNS, I can do HTTPS"
- Fedora kernel + initrd → "now I have a Linux kernel and early userspace"
- Anaconda → "now I can partition, create users, chroot the target"
- bootc verb in kickstart → "now pull the container, identical to `bootc switch`"

The final step is identical in both flows. Everything upstream of that final
pull is what PXE has to solve and `bootc switch` gets for free.

**What Cloudflare Tunnel changes:** in the USB-and-switch flow, nothing.
Bazzite is already hosted at `ghcr.io/ublue/bazzite` — public HTTPS,
Cloudflare or not. You don't need your own HTTPS endpoint. In the PXE flow,
Cloudflare Tunnel replaces the local HTTP server on your primary. The
question becomes: *if both paths ultimately `bootc pull` from ghcr, what are
you actually serving from your tunnel?*

Answer: only the bootstrap artifacts that get the target to `bootc`. Roughly
20 KB of text files per provisioning (`boot.ipxe` + `ks.cfg`), plus maybe
the cached 100 MB Fedora netinst kernel/initrd if you don't want to proxy
to Fedora's mirror. Everything else — the 8 GB OS payload — still comes
from ghcr.

The tunnel serves kilobytes. That's it.

---

## The "most advanced" shape, for reference

The end state worth building toward, once you're past first-flash and ready
for fleet-level thinking:

1. **R2 bucket** hosts static artifacts — `boot.ipxe`, `vmlinuz`,
   `initrd.img`. Unchanging, public, served by Cloudflare's global CDN.
   Zero egress cost.
2. **Cloudflare Worker** generates `ks.cfg` dynamically per-request.
   Reads the iPXE UUID header or MAC in a query param, renders a kickstart
   with the right SSH pubkey / hostname / site-specific config for that
   specific machine. Free tier covers everything at this scale.
3. **`ghcr.io/your-org/harness`** unchanged — where the OS payload lives.
4. **Tiny PXE appliance at each site** — Raspberry Pi or a spare mini-PC
   running dnsmasq with a fixed config that hands out `ipxe-shim.efi` over
   TFTP. That shim embeds an instruction to fetch `https://cdn.yourdomain.com/boot.ipxe`.
   That's all it does. Zero maintenance.

The Framework at any site boots, gets DHCP from the local appliance,
fetches iPXE shim over TFTP, then everything else is Cloudflare + ghcr.
A kickstart change is a `git push` to whatever repo the Worker reads. An
OS change is a `git push` to the harness repo. A new site just needs a Pi
plugged into the LAN the target will sit on.

That's the architecture major tech companies use for fleet provisioning,
minus some corporate paranoia around internal PKI. At your scale, a single
person can operate it.

---

## What to do, in order

**Tonight** — ignore all of this. [Chapter 02](02-root-node-flasher.md)
works end-to-end locally. Get the first flash working before adding layers.

**Once the local flow is proven** — mirror the static flasher artifacts
(`boot.ipxe`, `vmlinuz`, `initrd.img`) to R2. Private bucket, unique paths,
public HTTPS endpoints. Test fetching from R2 via `curl`.

**Once R2 works** — flip `boot.ipxe` to fetch from Cloudflare URLs instead
of `10.42.0.1`. Reflash. The install should complete identically, but now
doesn't depend on your primary's nginx at all. Everything post-iPXE goes
over internet-delivered HTTPS.

**Once geographic independence matters** — move dynamic content (per-machine
rendered kickstart with embedded SSH pubkey) to a Cloudflare Worker. Ship
a tiny-PXE-appliance Pi to remote sites.

**Keep the USB ISO in your back pocket** for the "brand new site, I don't
know the network" case. That's its sweet spot and nothing else replaces it.

Three tools, three use cases, no redundancy. Build them in that order as the
scope demands, not before.
