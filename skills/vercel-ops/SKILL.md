---
name: vercel-ops
description: Use when operating a Vercel project rather than writing framework code — enabling or reading Web Analytics or Speed Insights, answering "how many visitors" or "where is traffic coming from" without opening the dashboard, wiring custom events with track(), building a project's dashboard link, reading runtime logs or errors, or handling Toolbar comment threads. Also use when asking whether a Vercel feature is available on the free tier, what the Hobby plan allows, or whether client work can run on it; when choosing which surface answers a Vercel task — the official Vercel skills, the Vercel MCP server, the CLI, or the REST API; when working with Services, a Dockerfile, or the Container Registry; and when landing v0 output into an existing codebase.
---

# vercel-ops

> **📡 ACTIVE-SKILL MARKER:** Prefix your reply with 📡 **only on turns where the work touches the
> `vercel-ops` domain** — reading or enabling analytics, querying live account state (visitors,
> deployments, logs, runtime errors, agent runs, Toolbar threads), assembling a dashboard link,
> deciding which surface answers a Vercel task, or landing v0 output. What matters is whether *this
> turn* touches the domain.
>
> **Writing framework code is not this domain.** A turn spent authoring a component, a route, a
> Server Action, or a Tailwind class is Next.js work even in a project that deploys to Vercel —
> **omit 📡**. The marker means "I operated the platform", not "this project is hosted there".
> Likewise omit it on plain git operations, typechecks, and edits in unrelated domains, even if the
> skill loaded earlier in the session. If other active skills also apply to the same turn,
> **stack their emojis** in the prefix.

An **operating layer** over the official Vercel plugin, not a replacement for it.

The official plugin (30 skills) is the right answer for framework and platform *concepts* — Next.js,
AI SDK, eve, Workflow, firewall, storage, shadcn, microfrontends. Send conceptual questions there and
do not re-derive them here.

This skill covers what that plugin structurally cannot: **which surface to reach for**, the places
where its text is out of date, and two topics it never covers — **Web Analytics as an end-to-end
operation**, and **v0 output landing in a real codebase**.

## The core insight

The official plugin never references a single MCP tool. Every one of its 30 skills is written for the
CLI and the dashboard. When a live-state question arrives — visitors, logs, runtime errors, agent
runs, review comments — following that plugin sends you to a browser for something an MCP call
answers in one round trip.

**Reach for the MCP first on anything that reads live account state.**

## Which surface answers this?

| The task is… | Use | Why |
|---|---|---|
| Conceptual — "how does X work", "how do I build Y" | the matching official skill | It is deeper and maintained upstream |
| **Live state** — projects, deployments, logs, runtime errors, analytics, agent runs, Toolbar threads | **MCP** `mcp__vercel__*` | One call, no browser, no auth dance |
| **Mutating the local repo or project settings** — link, env, deploy from disk, enable a product | **CLI** | It owns the on-disk `.vercel/` state |
| What neither reaches (e.g. a branch-scoped review domain) | **REST** | Last resort, documented per case |
| A local production build | **never** | The platform builds; a local build proves nothing about the deployed artifact |

Full routing table to the 30 official skills, plus the MCP tool inventory and the documented
corrections: **[reference/tool-hierarchy.md](reference/tool-hierarchy.md)**

## Runbooks

These are phrase-triggered actions. When the user says something in the left column, execute — do not
explain the option space first.

| Said | Do |
|---|---|
| "add analytics to this site" | The six-step runbook in [reference/web-analytics.md](reference/web-analytics.md), ending with the dashboard link |
| "add speed insights too" | Same shape, via `vercel project speed-insights` |
| "how's traffic on X?" | `get_web_analytics` in `count` mode, then hand back the link |
| "where are visitors coming from?" | `aggregate` by `referrerHostname`, `country`, or `route` |
| "track the WhatsApp button" (any element) | **Check the plan first** — custom events are Pro-only ([reference/plans-and-limits.md](reference/plans-and-limits.md)) — then the `track()` wiring |
| "analytics is broken / shows zero" | The diagnosis order in [reference/web-analytics.md](reference/web-analytics.md#when-it-reads-zero) — check the expected causes before debugging anything |
| "the client commented on the preview" | `list_toolbar_threads` → `get_toolbar_thread` → `reply_to_toolbar_thread` |
| "how did that agent run go?" | `list_agent_runs` → `get_agent_run` → `get_agent_run_trace` (eve framework observability) |

## Identity is derived, never stored

Every account-specific value is read at runtime. Nothing about a team or project is hardcoded, which
is what lets this skill work on any account — including a client's.

| Value | Source |
|---|---|
| `teamSlug` | `list_teams` |
| `projectId`, `orgId`, `projectName` | the repo's `.vercel/project.json` |

> **`orgId` is the `teamId`.** `.vercel/project.json` calls it `orgId`; every MCP tool asks for
> `teamId`. They are the same `team_…` value — pass it straight through. Nothing surfaces this, and
> looking for a separate `teamId` is a dead end.

**When the user names no directory** ("our site", "the client's page"), there is no
`.vercel/project.json` to read. Resolve through the API instead: `list_teams` → `list_projects` →
match by name, and ask the user to disambiguate rather than guessing when more than one plausibly
fits. Acting on the wrong project is worse than one clarifying question.

Dashboard links are then **assembled**:

```
https://vercel.com/<teamSlug>/<projectName>/analytics
https://vercel.com/<teamSlug>/<projectName>/speed-insights
```

> **The local directory name is not the project name.** A repo checked out as `foo` is routinely
> named something else on Vercel — `website-foo`, `foo-prod`, a name chosen years ago. Building a link
> from the directory name yields a 404 that reads to the user as "the feature didn't work". Always
> read `projectName` from `.vercel/project.json`. If the file is absent, run `vercel link` first
> rather than guessing.

## Common mistakes

| Mistake | Reality |
|---|---|
| Proposing a plan-gated capability without saying so | Custom events, a reporting window beyond one month, password-protected deployments and more are Pro-and-above. Name the required plan **in the same sentence** as the proposal: [reference/plans-and-limits.md](reference/plans-and-limits.md). |
| Assuming client work can live on the free tier | Being **paid to build or host** a site is commercial use, which Hobby forbids outright — regardless of whether the site itself sells anything. |
| Sending the user to the dashboard for visitor counts | `get_web_analytics` answers it. The official plugin's claim that this is dashboard-only is wrong. |
| Reporting "analytics isn't working" when it reads zero | Web Analytics collects **in production only**. Zero right after enabling is expected — say so. |
| Building the dashboard link from the folder name | Use `projectName` from `.vercel/project.json`. |
| Treating the Vercel MCP as read-only | Its own bundled config says so and is stale — the live server deploys, buys domains, and posts replies. |
| Committing v0 output as-is into an established repo | v0 emits shadcn/ui and client-heavy components. See [reference/v0-bridge.md](reference/v0-bridge.md). |
| Declaring an env var unknowable — or regenerating it — because `env pull` shows `[SENSITIVE]` | Display masking, not data loss. Verify in the deployed artifact: [reference/tool-hierarchy.md](reference/tool-hierarchy.md#deployment-conventions). |
| Running a local production build to "check the deploy" | Build on the platform; inspect with `get_deployment_build_logs`. |

## Reference

| File | Holds |
|---|---|
| [reference/tool-hierarchy.md](reference/tool-hierarchy.md) | Routing table to the 30 official skills, MCP tool inventory, corrections with citations, deployment conventions |
| [reference/web-analytics.md](reference/web-analytics.md) | Enabling from the CLI, the six-step runbook, querying modes and dimensions, custom events |
| [reference/plans-and-limits.md](reference/plans-and-limits.md) | The commercial-use rule, free-tier ceilings, which capabilities are plan-gated |
| [reference/v0-bridge.md](reference/v0-bridge.md) | When v0 wins and loses, landing rules, design-system import, cost |
| [reference/ship-2026-gaps.md](reference/ship-2026-gaps.md) | Services, Dockerfile and the Container Registry — the silent failures only. Dated; delete when upstream covers them |
