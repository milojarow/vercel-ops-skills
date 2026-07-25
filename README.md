# vercel-ops-skills

**An operating layer over the official Vercel plugin — the surface it doesn't cover.**

## What is this?

The official Vercel plugin ships ~30 skills and is the right answer for framework and platform
concepts: Next.js, AI SDK, eve, Workflow, firewall, storage, shadcn, microfrontends. This repo does
not duplicate any of that.

It covers what that plugin structurally cannot: **which surface answers a given task**, the places
where its text is out of date, and two topics it never covers — Web Analytics as an end-to-end
operation, and landing v0 output in an existing codebase.

### Why this skill exists

- **The official plugin never references an MCP tool.** A repo-wide grep for `mcp__vercel__` returns
  zero matches across all 30 skills. Every one is written for the CLI and the dashboard, so live-state
  questions get routed to a browser for something one MCP call answers.
- **It says Web Analytics is "Dashboard only".** It isn't. `get_web_analytics` reads it, and
  `vercel project web-analytics` enables it — there is no dashboard step in the whole workflow.
- **It describes the MCP server as read-only.** Its bundled config still says "read-only in initial
  release"; the live server deploys, buys domains, and posts Toolbar replies.
- **Toolbar comment threads are undocumented.** Six MCP tools exist for reading and answering
  comments left on a preview deployment. Nothing upstream mentions them.
- **v0 is absent.** It appears only as a component-registry namespace. Nothing addresses what has to
  happen to v0's output before it can live in a repo with settled conventions.

Everything account-specific — team slug, project identity — is derived at runtime from `list_teams`
and the repo's own `.vercel/project.json`. Nothing is hardcoded, so the skill works on any Vercel
account.

## The skill

| Skill | Description |
|-------|-------------|
| **vercel-ops** | Surface routing (official skills / MCP / CLI / REST), Web Analytics end to end including custom events, dashboard-link assembly, Toolbar threads, and the v0 landing bridge. |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/vercel-ops-skills
```

Then install:

```
/plugin → Discover → vercel-ops-skills → Install
```

## Requirements

- The **Vercel MCP server** connected — see [`.mcp.json.example`](.mcp.json.example). Most of this
  skill's value is MCP calls.
- The **Vercel CLI**, for enabling products and anything that mutates on-disk project state.
- The **official Vercel plugin** installed alongside it. This skill routes to it and assumes it is
  present; it is not a replacement.

## License

MIT
