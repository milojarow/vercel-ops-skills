# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **vercel-ops-skills** repository — an operating layer over the official Vercel plugin,
covering surface routing, Web Analytics end to end, and the v0 landing bridge.

**Repository**: https://github.com/milojarow/vercel-ops-skills

## Repository Structure

```
vercel-ops-skills/
├── .claude-plugin/          # Claude Code plugin configuration
├── .mcp.json.example        # Vercel MCP server config (the skill's primary surface)
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT License
├── docs/                    # Design specs
├── evaluations/             # Test scenarios for the skill
└── skills/
    └── vercel-ops/
        ├── SKILL.md          # Entry point: surface routing + runbooks
        └── reference/        # tool-hierarchy, web-analytics, v0-bridge
```

## The skill

### vercel-ops
Picks the right surface for a Vercel task (official skills for concepts, MCP for live state, CLI for
on-disk mutation, REST as last resort), runs Web Analytics and Speed Insights end to end without the
dashboard, wires custom events with `track()`, assembles dashboard links from runtime-derived
identity, covers Toolbar comment threads, and bridges v0 output into an existing codebase.

## Skill Activation

Activates when operating a Vercel project rather than writing framework code — enabling or reading
analytics, answering traffic questions, building a dashboard link, reading runtime logs or errors,
handling preview comments, choosing between the official skills / MCP / CLI / REST, or landing v0
output.

## Design constraints

- **No hardcoded account identity.** Team slug comes from `list_teams`; project identity comes from
  `.vercel/project.json`. Never write a real team slug, project ID, or team ID into this repo — it
  breaks portability and leaks operator data into a public repo.
- **The corrections section is perishable.** It documents where the official plugin's text is wrong.
  Re-verify on each major upgrade of that plugin; if upstream fixes something, delete the correction
  rather than leaving a stale contradiction.
- **v0 content is a bridge, not a manual.** v0 ships changes monthly. Document how its output lands
  in a codebase, not how to drive its UI. The pricing table is dated in place — re-check before
  quoting it.

## Updating this skill

After any session that discovers a new wall. Keep entries generic — patterns and examples, never
client data. The git log of this repo is the diary.
