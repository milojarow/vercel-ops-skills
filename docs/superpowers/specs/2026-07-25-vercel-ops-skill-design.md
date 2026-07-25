# Design — `vercel-ops-skills`

**Date:** 2026-07-25
**Status:** approved, pending implementation plan

## Problem

The official Vercel plugin (`claude-plugins-official/vercel`, v0.45.1) ships **30 skills**, 4 commands
and 3 agents. It is comprehensive on framework and platform *concepts*, and it is the right place for
Next.js, AI SDK, eve, Workflow, firewall, storage, shadcn and microfrontends.

It has three structural gaps that no upstream release will close, because they are properties of the
*operator's* environment rather than of Vercel:

1. **It is blind to the MCP server.** A repo-wide grep for `mcp__vercel__` returns **zero matches**.
   All 30 skills are written for the CLI and the dashboard. Its own `.mcp.json` describes the server
   as *"Read-only in initial release: search docs, list projects/deployments, inspect logs"* — which
   is stale: the live server exposes 31 tools including `deploy_to_vercel`, `buy_domain`,
   `get_web_analytics` and `reply_to_toolbar_thread`.
2. **Two of its statements are wrong in a way that costs work.** `commands/status.md` lists
   *"Web Analytics data | **Dashboard only**"*. It is not dashboard-only: `get_web_analytics` answers
   over MCP, and `vercel project web-analytics` enables it from the CLI.
3. **It cannot hold the operator's own conventions** — the deployment gate, the CDN/`public/` rule,
   the sensitive-env-var trap, the branch-review-domain REST-only path.

Two topics are absent outright: **v0** (mentioned only as a shadcn registry namespace, `@v0/…`) and
**Vercel Toolbar comment threads** (zero coverage, despite three MCP tools for them).

## Goal

A thin **operating layer**: route to the official pack, fix what it gets wrong, and cover the gaps.
Never re-document what upstream already covers well.

## Non-goals

- Re-documenting Next.js, AI SDK, eve, Workflow, firewall, storage, shadcn, microfrontends.
- A v0 manual. v0's surface moves monthly; only the *bridge into this stack* is durable.
- Anything requiring a paid v0 plan to verify.

## Naming

| | |
|---|---|
| Repo | `vercel-ops-skills` |
| Plugin | `vercel-ops` |
| Skill | `skills/vercel-ops/` → invoked `vercel-ops-skills:vercel-ops` |

The obvious `vercel-skills:vercel` is **rejected**: the official plugin already occupies the `vercel`
namespace in every installed session, and the naming convention forbids colliding with an installed
skill's vocabulary. `-ops` names the function (operating layer) rather than the product.

## Privacy constraint — shapes the architecture

This repo is public. No client or business names, no real project/team IDs, no team slugs, no
hostnames, no literal `/home/<user>`.

This is not a limitation to work around; it forces the better design:

> **Every account-specific value is derived at runtime, never written down.**
> Team slug comes from `list_teams`. Project identity comes from the repo's own
> `.vercel/project.json`. Dashboard links are *assembled*, not stored.

Consequence: the skill works on any machine, any Vercel account, any team — including a client's.
The operator's own inventory (which projects have analytics, which clients they map to) lives in
**memory**, never in this repo.

## Architecture

One cohesive skill. The domain is a single decision surface — "which Vercel tool answers this?" —
so it does not split into sub-skills.

```
vercel-ops-skills/
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json
├── skills/vercel-ops/
│   ├── SKILL.md
│   └── reference/
│       ├── tool-hierarchy.md
│       ├── web-analytics.md
│       └── v0-bridge.md
├── evaluations/vercel-ops/eval-*.json
├── .mcp.json.example
├── README.md
├── LICENSE
└── CLAUDE.md
```

### Optional components — decided explicitly

| Component | Decision | Reason |
|---|---|---|
| `.mcp.json.example` | **include** | The skill's whole premise is the MCP server; a reader must be able to wire it. |
| `hooks/` | **skip** | Triggers here are conversational ("add analytics to this site"), not path-shaped. A `pathPatterns` match on `app/layout.*` would fire on every unrelated layout edit, and the matcher never sees the MCP calls that carry most of this skill's value. |
| `build.sh` + `dist/` | **skip** | Marketplace distribution only. |
| `docs/` | **include** | Holds this spec. |
| `.gitignore` | **skip** | No build artifacts. |
| `skills.png` | **skip** | Cosmetic. |

## SKILL.md — four sections

### 1. Tool hierarchy

| The task is… | Use |
|---|---|
| Conceptual — "how does X work" | the matching official skill |
| Live state — projects, deployments, logs, runtime errors, analytics, agent runs, toolbar threads | **MCP** `mcp__vercel__*` |
| Mutating the local repo — link, env, deploy from disk, enable a product | **CLI** |
| What neither reaches | **REST** |
| Local production build | **never** — the platform builds |

### 2. Routing table to the official pack

Task → which of the 30 skills to load, with the standing caveat that the pack assumes CLI/dashboard
and will not mention an MCP tool even when one is faster.

### 3. Named runbooks

Phrase-triggered actions, not prose:

| Said | Done |
|---|---|
| "add analytics to this site" | the six-step runbook below, ending in a link |
| "add speed insights too" | same shape via `vercel project speed-insights` |
| "how's traffic on X?" | `get_web_analytics` + link, no browser |
| "where are visitors coming from?" | `aggregate` by `referrerHostname` / `country` / `route` |
| "track the WhatsApp button" | `track()` wiring + how to query it afterwards |

### 4. Corrections and local conventions

Every correction cites `file:line` in the official pack so it reads as evidence, not opinion.
Local conventions covered: the pre-payment production gate, `public/` gitignored when an external CDN
serves the assets, `--scope` required for non-interactive deploys, and the branch-review-domain path.

## reference/web-analytics.md

**Enabling is fully automatable — no dashboard.** Verified against CLI 56.5.0:

```
vercel project web-analytics  [name]   Enable Web Analytics for a project
vercel project speed-insights [name]   Enable Speed Insights for a project
    -F, --format <FORMAT>              Specify the output format (json)
```

The CLI's own help labels `--format json` as *"non-interactive / agents"* — it is built to be driven
by an agent.

### Runbook: add analytics to a project

1. Read `.vercel/project.json` → `projectId`, `orgId`, `projectName`. If absent, `vercel link` first.
2. `npm i @vercel/analytics`
3. Root layout: `import { Analytics } from '@vercel/analytics/next'`, render `<Analytics />` before `</body>`.
4. `vercel project web-analytics <projectName> --format json`
5. Commit and push; the platform builds.
6. Assemble and hand back the dashboard link.

### Link assembly

```
https://vercel.com/<teamSlug>/<projectName>/analytics
https://vercel.com/<teamSlug>/<projectName>/speed-insights
```

`teamSlug` ← `list_teams`. `projectName` ← `.vercel/project.json`.

> **Wall — the local directory name is not the project name.** A repo checked out as `foo` is
> routinely named `website-foo` (or anything else) on Vercel. Building the link from the directory
> name yields a 404 that reads as "the feature didn't work". Always read `projectName`.

> **Wall — Web Analytics collects in production only.** Immediately after enabling, the dashboard and
> the API both report zero. That is expected, not a failure. Say so instead of debugging it.

### Reading without a browser

`get_web_analytics` in `count` mode returns totals; `aggregate` mode groups by up to two dimensions
(`route`, `country`, `referrerHostname`, `deviceType`, `browserName`, `eventName`, `eventData/<prop>`,
plus time buckets) and accepts an OData `filter`. `aggregate` requires `since`, `until` and `by`.

### Custom events

`track('<event>', { … })` from `@vercel/analytics`, then query with `dataset: 'events'` and
`filter: "eventName eq '<event>'"`. This is what turns analytics into a retention argument for a
client: concrete counts of the action that matters to their business, not raw pageviews.

## reference/v0-bridge.md

Framed as a bridge, not a manual — v0 ships changes monthly; what is durable is how its output lands
in this stack.

- **When v0 wins:** producing a client-facing demo fast, where the demo is a sales instrument.
  **When it loses:** production code in an established repo, where its conventions fight the stack's.
- **Landing rules.** v0 emits shadcn/ui components (stated in the official pack's own `vercel.md`),
  while this stack is DaisyUI + server-first. The reference documents the conversion, purging
  unnecessary `'use client'`, file placement, and writing real accented characters in JSX text.
- **Design Systems 2.0** — importing an existing design system so generations use the stack's real
  components instead of generic shadcn. This is the setup that removes the conversion tax.
- **Cost.** Plan tiers and per-model token pricing, with the recommendation to evaluate on the free
  tier before paying.

## Evaluations

Drawn from the walls the skill documents:

1. Asked to add analytics, the local directory name differs from the Vercel project name — does the
   returned link use `projectName`?
2. Asked how many visitors a site had — is it answered over MCP, or deflected to the dashboard?
3. Analytics just enabled, dashboard reads zero — explained as production-only, or misdiagnosed?
4. v0 output pasted into a DaisyUI repo — converted, or committed as shadcn?
5. An env var reads `[SENSITIVE]` after `env pull` — is the value verified in the deployed artifact
   rather than declared unknowable?

## Verified facts behind this design

| Claim | Evidence |
|---|---|
| Official pack has 30 skills, never references an MCP tool | `grep -roE "mcp__vercel__[a-z_-]+"` → 0 matches |
| Pack calls the MCP read-only | its `.mcp.json` |
| Pack calls Web Analytics dashboard-only | its `commands/status.md` |
| Enabling works from the CLI | `vercel project --help`, CLI 56.5.0 |
| `get_web_analytics` returns real data | live call against a linked project |
| v0 emits shadcn/ui | the pack's own `vercel.md` |
| v0 plan tiers and token pricing | v0.app/pricing, fetched 2026-07-25 |
| Toolbar threads uncovered | grep for toolbar/comment terms → 0 matches |

## Open risks

- **Upstream may close a gap.** If a future release documents the MCP tools, the routing table shrinks
  and the corrections section may empty. That is a good outcome; the skill should be re-checked against
  the pack on each major upgrade rather than assumed still-correct.
- **v0 moves fast.** The bridge is written to survive that; the pricing table is the perishable part and
  is dated in place.
