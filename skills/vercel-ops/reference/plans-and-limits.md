# Plans and limits — what the free tier actually allows

Check this **before** promising a capability, not after wiring it. Several features documented
elsewhere in this skill are plan-gated, and the failure mode is silence: the code runs, nothing
errors, no data appears.

Figures verified against Vercel's published limits pages, **2026-07-25**. Pricing pages move —
re-verify with `search_vercel_documentation` before quoting a number to anyone.

## The commercial-use rule

This is the highest-consequence line in Vercel's docs and the easiest to be unaware of.

> Hobby teams are restricted to non-commercial personal use only. All commercial usage of the
> platform requires either a Pro or Enterprise plan.

Commercial usage is defined as any Deployment used for the financial gain of **anyone** involved in
**any part of the production** of the project — explicitly including a paid employee or consultant
writing the code. The documented examples include:

- Requesting or processing payment from visitors
- Advertising the sale of a product or service
- **Receiving payment to create, update, or host the site**
- Affiliate linking as the site's primary purpose
- Carrying advertisements
- Soliciting donations

**The consequence for agency-style work is direct:** a site built for a paying client is commercial
under this definition even if the site itself sells nothing and shows no ads — being paid to build or
host it is sufficient. Hobby is the wrong plan for that work, and the exposure is not theoretical:
the deployments in question are the ones a client depends on.

Raise this once, factually, when the situation appears. Do not moralize and do not repeat it every
session — state the rule, name the cost of compliance, and let the operator decide.

## Hobby ceilings

| Resource | Hobby |
|---|---|
| Web Analytics events | 50,000 / month, **shared across every project in the account** |
| Web Analytics reporting window | **1 month** |
| Web Analytics custom events | **not available** |
| Speed Insights events | 10,000 |
| Speed Insights projects | **1** |
| Active CPU | 4 CPU-hrs / month |
| Provisioned memory | 360 GB-hrs / month |
| Function invocations | 1,000,000 |
| Edge requests | 1,000,000 |
| Runtime log retention | **1 hour** |
| Projects | 200 |
| Deployments per day | 100 |
| WAF custom rules / IP blocks | 3 each |
| Deployment protection | Vercel Authentication only — no password protection, no shareable links |

## Web Analytics by plan

| | Hobby | Pro | Pro + Analytics Plus | Enterprise |
|---|---|---|---|---|
| Included events | 50,000 / mo | none (usage-billed) | usage-billed | none |
| Additional events | **cannot purchase** | $0.03 / 1K | $0.03 / 1K | custom |
| Reporting window | 1 month | 12 months | 24 months | 24 months |
| **Custom events** | **–** | included | included | included |
| Properties per custom event | – | **2** | 8 | 8 |
| UTM parameters | – | – | included | included |

The Plus add-on is $10/month per team on top of Pro.

### Consequences to state up front

- **`track()` does nothing useful on Hobby.** The custom-events section of
  [web-analytics.md](web-analytics.md) is a Pro-and-above capability. Wiring `track()` on a Hobby
  account produces no error and no data — the worst combination. Check the plan before proposing it.
- **Two properties per event on Pro**, not unlimited. Design the event shape around that cap; the
  eight-property tier requires the Plus add-on.
- **A one-month window makes historical comparison impossible on Hobby.** "How did this quarter
  compare to last" has no answer. Say that rather than returning a partial series that reads as a
  collapse in traffic.
- **The 50,000 events are pooled across every project.** One busy site can exhaust the allowance for
  all the others, and nothing about the quiet project will indicate why it stopped collecting.
- **Exceeding it pauses collection**: a three-day grace period, then capture stops, and it does not
  resume for seven days. Data for that window is gone, not delayed.
- **The analytics script itself costs data transfer and edge requests**, which draw on separate
  Hobby allowances.

## Containers and services on the free tier

Plan gating for Dockerfile builds, the Container Registry, and Services is **not documented** — do
not assert that any of them is Pro-only, and do not assert that it is free.

What *is* documented is the ceiling they run into: on Hobby, **Active CPU is capped at 4 CPU-hrs per
month** and provisioned memory at 360 GB-hrs. A container backend that stays warm consumes both
continuously rather than per-request. Treat that as the practical constraint and verify actual
consumption before recommending the architecture — this is an inference from a published limit, not
a published statement about containers.

## How to check a live account

Plan is account state, so read it rather than assuming:

- `get_project` and `list_teams` over MCP for project and team context.
- The account's usage dashboard for consumption against these allowances.
- When a plan-gated feature is about to be proposed, say which plan it requires **in the same
  sentence** as the proposal. Discovering the gate after the code is written wastes the work and
  reads as a bait-and-switch.
