# 00 — Why this guide exists

Before the mechanical chapters (BIOS, flasher, image, install) make sense, you
need the argument that motivates building the pipeline in the first place.
That argument is economic, and it is falsifiable. This chapter is that argument.

---

## The default is backwards

The current default pattern for a team adopting AI tooling is: pay per seat for
a hosted frontier model (Claude Pro, Team, Max; ChatGPT Plus; Gemini Advanced),
route every request to the most capable available tier, and hit the rate
limits that come with it. A knowledge worker reviews a generic internal email
with Claude Opus — a model with legal-contract-grade reasoning — and eats 70%
of their monthly allocation on throwaway work. A team of 10 hits Team tier
ceilings mid-month and upgrades three users to Max to unstick them. Monthly
bill creeps from €200 to €500 to €1,000.

The economics of this are not inherently wrong — frontier cloud models are
genuinely the right tool for some tasks. But *starting* at the top tier and
falling back to local when cost bites *inverts* the routing decision the
infrastructure should enforce.

The alternative: **every AI invocation defaults to local inference on
dedicated hardware; cloud is explicit per-invocation escalation.** A daily
customer-commentary run lands on a local model by default; a legal-contract
review that genuinely requires Opus is flagged in the tool invocation
(`escalate: "cloud-heavy"`). The ledger captures both, with cost attribution
on each side.

If local coverage is even 30% of invocations, the math works. If it hits 70%,
the box pays for itself in ~2 months.

---

## The ledger principle

Every AI invocation — regardless of which surface (chat UI, IDE agent, MCP
tool, scheduled job, email autoreply) — emits one record to a single local
ledger. Minimum schema:

| Field | Why |
|---|---|
| `timestamp` | When |
| `surface` | Which product/agent made the call |
| `user_id` | Who (needed for per-seat quotas) |
| `model` | Which model actually served it |
| `tokens_in`, `tokens_out` | Consumption |
| `latency_ms` | Quality-of-service tracking |
| **`cost_eur`** | **Actual cost — `0` for local, real cost for cloud** |
| **`cost_eur_if_cloud`** | **Counterfactual — what the frontier cloud equivalent would have charged** |
| `outcome` | Success / error / refused / timeout |

The savings number is one SQL query:

```sql
SELECT SUM(cost_eur_if_cloud) - SUM(cost_eur) AS savings_eur
FROM run_ledger
WHERE timestamp >= date('now', '-30 days');
```

That's the only number that matters. Not a marketing claim, not a pitch deck
assertion — a query result against a table the customer owns.

**Discipline around the counterfactual.** `cost_eur_if_cloud` has to be
honest. Three numbers a rigorous ledger distinguishes:

1. *All-frontier counterfactual* — if every call had gone to Opus. Worst case,
   marketing number, pessimistic.
2. *Realistic-mix counterfactual* — weighted by what the user actually would
   have done (60% Sonnet, 40% Opus for a typical knowledge worker). More
   honest.
3. *Real spend replaced* — the subscriptions the customer actually cancels
   or downgrades. Hardest to defend, most honest.

Lead with #3, show #2 as "additional efficiency," keep #1 for pedagogy. Don't
blur them.

---

## The headline math

Concrete numbers from a real 2026 deployment (10-30 person distributor, four
departments: admin ops, sales ops, accounting, planning):

| Category | Value | Source |
|---|---|---|
| 1 × Strix Halo box (Framework Desktop 395+ 128GB) | €3,583 HT | vendor |
| Electricity (Strix Halo at typical inference load) | ~€210/month | Paris rates |
| Cloud counterfactual for the same workload | €1,640–2,500/month | realistic tool-call volume |
| **Payback (single box, cloud-spend basis)** | **~2 months** | |
| Existing SaaS cancellations (email, CRM, e-commerce) | €3,158/year | concurrent |
| Time savings (co-estimated with team) | €54,600/year at full confidence | 162.5 hrs/mo × €28/hr blended |
| Time savings at 50% confidence discount | €27,300/year | |
| **Payback (two boxes + implementation, time-savings basis)** | **<6 months** | |
| **P&L net after 5-yr amortization** | **+€758/year from day one** | SaaS cancellations alone |

The 70% local-coverage target is the single variable that keeps all of these
numbers load-bearing. Ledger shows you that number in real time. If it holds,
the argument holds. If it doesn't, you know before the quarter ends.

---

## Who this actually pays off for (honestly)

Not every team. Three tiers, by current cloud spend:

| Current team spend | Payback | Sell on |
|---|---|---|
| Basic Pro seats (€20–30/user) | 12–24 months | Privacy, compliance, capability ceiling — **not** savings |
| Team tier (€30+/user, mid-sized team) | ~12 months | Savings + value + unlock |
| Power users on Max or Enterprise (€100+/user) | **3–6 months** | Savings, full stop — ROI writes itself |

The third category is the obvious first target. A team where 3–5 people are
burning through Max tiers is a pre-qualified customer. The hardware pays off
before the renewal of their next annual subscription cycle.

For the first category, the pitch isn't savings — it's *unlock*. Unlimited
local inference means employees stop mentally rationing their tool use.
"Should I really ask the AI for this?" goes away. Measured team AI-usage
volume typically **triples** after migration, because the friction drops to
zero. That behavior change is worth something even when the savings don't
pencil out.

---

## The falsifiable wager

The argument above is not asserted, it is *proposed* with a pre-committed
evaluation window. The shape that actually lands with skeptical SMB owners
and CFOs:

> Give me the budget for one box plus two weeks of implementation. I will
> instrument everything — every invocation, every token, every counterfactual.
> In 90 days we look at the ledger together. If the savings don't speak for
> themselves, we undo it.

That's the trust architecture. Not a sales pitch, not a case study, not a
promise — a bet with a public scoreboard. The ledger is that scoreboard. The
setup guide in this repo is how the scoreboard gets built.

---

## What this guide does

This repo is the externalized, reusable, generic version of what was already
done once for one deployment. It ends where the Leger layer begins. Scope:

- **Chapter 01 — BIOS.** The one-time physical access on the target.
- **Chapter 02 — Root node flasher.** The primary-side PXE stack.
- **Chapter 03 — Harness bootc image** *(next)*. What goes in the slow-moving
  OS layer.
- **Chapter 04 — First flash + headless handoff** *(next)*.
- **Chapter 05+ — Account chain** *(pending)*. Google Workspace → Cloudflare
  → GitHub org → Tailscale.
- **Chapter 06+ — Cloud provisioning** *(pending)*. The `leger bootstrap`
  verbs against Terraform recipes.

Before any of those make sense, three companion chapters explain the deeper
WHY of how this guide is shaped:

- **[`0A — Skills need a policy layer`](0A-skills-need-a-policy-layer.md)** —
  the technical thesis. Why setup and invocation must be separated. Why this
  is what `leger` is for.
- **[`0B — ISO vs PXE vs Tunnel`](0B-iso-vs-pxe-vs-tunnel.md)** — why this
  guide standardizes on PXE even though USB ISOs are simpler, and what the
  Cloudflare Tunnel upgrade path looks like.
- **[`0C — Harness vs dynamic layer`](0C-harness-vs-dynamic-layer.md)** — why
  the bootc image is deliberately slow-moving, what gets installed last-mile,
  what belongs in chezmoi vs Leger vs the image itself.

Read 00 → 0A → 0B → 0C if you want the argument for why the pipeline is
shaped this way. Skip to 01 if you just want to provision the box.
