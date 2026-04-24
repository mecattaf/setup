# 0A — Skills need a policy layer

*The technical thesis underneath this setup guide. If `00-why.md` is the
commercial case for building the pipeline at all, this is the architectural
case for why the pipeline is shaped the way it is — specifically, why the
`leger` CLI and the runbook/policy layer exist as distinct primitives from
the capabilities they enable.*

---

Every useful AI skill has two halves. The **invocation** — what Claude Code
does when you ask it to ship something — is the half the ecosystem currently
models. The **setup** — the one-time account wiring, token minting, credential
placement that has to happen before the invocation works at all — is the half
that has no name and no home.

Right now that setup half ends up in one of three places, all wrong:

1. **Inside the skill itself**, behind an `if (!configured) { runSetup() }`
   branch the LLM hits on first invocation.
2. **In a README** the user is expected to follow manually, usually with an
   LLM pair-programming the setup as a one-off.
3. **Nowhere** — the skill assumes configuration exists and fails cryptically
   when it doesn't.

All three are bad for the same reason: **setup is deterministic, durable, and
has memory. LLM-driven execution is stochastic, transient, and memoryless.**
Running deterministic work inside a stochastic runtime is the category error
the skills ecosystem hasn't addressed yet.

---

## The canonical example

A team wants Claude Code to ship durable artifacts — dashboards, internal
tools, generated reports — to a Cloudflare Access–gated Pages site.

The invocation they want is one command:
`npx claude-deploy my-dashboard/`. The artifact lands at
`https://dashboards.acme.internal`, gated to their Google Workspace domain,
with audit logs in Cloudflare. Fast, repeatable, no drama.

For that one command to work, the following must already be true:

- A Cloudflare Pages project exists at `acme-dashboards`
- A Cloudflare Access application protects the project's hostname
- That Access application is bound to Google Workspace as the IdP
- An email-domain policy allows `@acme.com` addresses
- A scoped API token exists with `Pages:Edit` + `Access:Read`
- The token is stored somewhere the skill can read (Vault, 1Password, a
  config file under `~/.config/`)
- A `wrangler.toml` in a known path references the project and the token

None of that is hard. All of it is deterministic. **None of it should happen
inside the skill.**

If it lives inside the skill, every invocation re-checks account state (3-5
wasted tool calls), different Claude Code sessions reason about partial state
differently (inconsistent UX), half-done setups produce the worst failure
modes (the LLM improvises, often wrongly), and the user cannot audit what
their account looks like from chat transcripts. Also: the skill ends up 500
lines of error handling around the Cloudflare API instead of the 20 lines of
actual functionality.

---

## Separating the two halves

**Policy** (runs once per team, durable state, resumable, auditable):

```bash
leger policy apply claude-deploy --team=acme
```

This is a Terraform module, or a small Go function calling the Cloudflare
provider library directly, or even a shell script with idempotency checks.
Doesn't matter. What matters is: it has a state file, it completes or it
doesn't, it can resume from where it stopped, and when it succeeds it writes
a manifest to a known location:

```jsonc
// ~/.config/leger/policies/claude-deploy/state.json
{
  "status": "ready",
  "team": "acme",
  "pages_project": "acme-dashboards",
  "hostname": "dashboards.acme.internal",
  "token_ref": "vault://acme/cloudflare/pages-deploy",
  "wrangler_config": "~/.config/leger/policies/claude-deploy/wrangler.toml",
  "applied_at": "2026-04-24T18:30:00Z",
  "applied_by": "operator@acme.com"
}
```

**Skill** (runs every invocation, reads policy output, does the one thing):

```bash
npx claude-deploy my-dashboard/
```

Internally, ~20 lines:

```
1. Read ~/.config/leger/policies/claude-deploy/state.json
2. If missing or status != "ready": exit with
   "Run `leger policy apply claude-deploy` first."
3. wrangler pages deploy ./my-dashboard/
     --project-name=<state.pages_project>
     --config=<state.wrangler_config>
4. Print the URL.
```

No Cloudflare API calls. No authentication logic. No retry handling for
account provisioning. The skill trusts its preconditions because the
preconditions are a file on disk, verifiable in O(1).

---

## The error message is the API

The line that does the work:

> `Run leger policy apply claude-deploy first.`

That's the contract between the policy layer and the skill layer. The skill
declares its dependency; the policy layer fulfills it; neither knows how the
other works internally. Standard interface segregation, applied to AI tooling
instead of OO classes.

This makes skills composable in a way the current ecosystem doesn't support.
A skill author writes their deploy logic assuming a stable environment. A
policy author maintains the playbook that establishes that environment. They
can ship independently, version independently, be authored by different
people. The skill's `requires:` declaration is metadata, not code. An
orchestrator (Claude Code itself, or a wrapper) can check policy state
*before* invoking the skill, surfacing missing prerequisites as
human-addressable actions instead of runtime errors deep in a tool call.

---

## Why LLMs can't be the runtime for this

The naive objection: *Claude Code can just run setup when it needs to.*

It can. It shouldn't. Three properties a runtime needs that LLM execution
can't guarantee:

**Resumability.** Setup phase 5 fails halfway through — token created, but
the Vault write returned a network error. The next session must start from
"token exists, finish Vault write," not "let me re-check the whole account."
An LLM reading state from disk is not the same as a state machine advancing
a state file. The LLM can be wrong about what the state implies; a state
file is the source of truth by construction.

**Idempotency guarantees.** Cloudflare's API is mostly-but-not-entirely
idempotent. Creating a Pages project twice gives you an error, not two
projects. Creating an Access application twice might give you two. A
deterministic runner knows which calls tolerate re-execution and which
don't. An LLM has to infer this every time, and gets it wrong enough to
matter.

**Auditability.** When a customer asks *"what did you do to my Cloudflare
account on Tuesday?"* the answer needs to be a diffable artifact, not a chat
log. Terraform state is a diffable artifact. A JSON manifest written by a
policy runner is a diffable artifact. A 400-message Claude transcript is not.

None of this is a critique of LLMs. It's a critique of using LLMs for the
wrong layer. LLMs are excellent at the parts that require *judgment* —
choosing what to build, debugging weird failures, recovering from novel
situations. They're poor at the parts that require *exactness* — apply these
8 API calls in this order with these inputs, record the outputs here, don't
do it again if it already happened.

Policy is the exactness layer. Skills are the judgment layer. Splitting them
is the whole point.

---

## What this looks like as a product

The skill layer is already forming — Claude Code plugins, pi-mono's npm
collection, openclaw, the emerging MCP ecosystem. That's the capability
marketplace. It doesn't need to be rebuilt.

The policy layer is the missing piece. What it needs:

- A **format** for declaring a policy (inputs, phases, outputs, dependencies).
  Terraform is adequate for the subset of work that's pure cloud provisioning;
  richer formats become necessary when policies include manual gates (DNS
  migration) or imperative steps (register a device with a backend).
- A **runner** that executes policies with durable state, resumability, and
  observability. Can be a thin wrapper over `tofu apply -target=...` for the
  Terraform cases; needs more for the general case.
- A **state convention** — where policies write their output manifests, how
  skills read them, how to rotate credentials without breaking dependent
  skills.
- A **dashboard** showing what policies exist in a team's environment, which
  are applied, which are stale, which have drift. Not optional; this is what
  turns "a collection of shell scripts" into "infrastructure an operator can
  manage."

This is being built as [Leger](https://github.com/mecattaf/setup). The
runbook format is HCL-or-YAML over Terraform modules plus imperative steps.
The runner is a Go binary that spawns `tofu` as a subprocess and writes
manifests to `~/.config/leger/policies/<name>/state.json`. The dashboard is
a Cloudflare Workers app that reads state over HTTP from each customer's
self-hosted Leger instance. Everything open source; the revenue is in the
service of implementing it for SMBs, not in the software itself.

The thesis is narrow and testable:

> **Skills become fast, composable, and trustworthy only when setup is
> separated from invocation. The policy layer is what separates them. Today
> that layer doesn't exist as standard infrastructure. Building it is the
> missing primitive in the LLM-ops stack.**

Everything else — the Framework Desktop hardware specificity, the local-LLM
ledger in [`00-why.md`](00-why.md), the Cloudflare-maximalism, the bootc OS
substrate in [`0C`](0C-harness-vs-dynamic-layer.md) — is implementation
detail in service of that thesis for one specific customer segment (SMBs
on-prem). The primitive itself is general. The first well-executed
implementation becomes the reference.

---

## The part worth discussing

Two things genuinely uncertain:

**Is this a product or a protocol?** The state-file-on-disk contract between
policies and skills could be a spec anyone implements — like OpenTelemetry
for AI tooling. Or it could be a product — one runtime, one dashboard, one
commercial implementer. Product-first is the current bet, because specs
without reference implementations go nowhere. But the end state might be
standardization. Worth getting right early because it changes what gets
open-sourced versus built as commercial offering.

**Where does Terraform end and Leger begin?** A lot of what's described here
is *"Terraform with a progress UI and a skill-facing manifest output."* At
some scale of ambition, forking or extending OpenTofu to emit these
manifests natively might be cleaner than wrapping it. The decision right now
is against that — keep OpenTofu as a subprocess — but as the policy format
richens (imperative steps, manual gates, non-Terraform capabilities), the
wrapper may accumulate enough logic that the boundary starts mattering.

Both are the kind of questions worth a half-hour whiteboard with someone
who's built something comparable.
