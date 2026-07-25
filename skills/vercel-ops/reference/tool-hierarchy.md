# Tool hierarchy — which surface answers a Vercel task

Four surfaces exist. They are not interchangeable, and the official plugin only knows two of them.

## The MCP tool inventory

The Vercel MCP server exposes ~31 tools. The official plugin references **none** of them — a
repo-wide grep for `mcp__vercel__` returns zero matches — so nothing downstream will suggest these
unless this skill does.

| Group | Tools |
|---|---|
| Discovery | `list_teams`, `list_projects`, `get_project`, `list_deployments`, `get_deployment` |
| Debugging | `get_deployment_build_logs`, `get_runtime_logs`, `get_runtime_errors` |
| Analytics | `get_web_analytics` |
| Agent observability | `list_agent_run_projects`, `list_agent_runs`, `get_agent_run`, `get_agent_run_trace` |
| Toolbar threads | `list_toolbar_threads`, `get_toolbar_thread`, `reply_to_toolbar_thread`, `edit_toolbar_message`, `add_toolbar_reaction`, `change_toolbar_thread_resolve_status` |
| Deploy / access | `deploy_to_vercel`, `get_access_to_vercel_url` |
| Domains | `check_domain_availability_and_price`, `buy_domain`, `get_domain_order` |
| Billing | `buy_pro`, `buy_addon`, `buy_credits`, `get_purchase_quote` |
| Docs | `search_vercel_documentation`, `web_fetch_vercel_url` |
| Design import | `import-claude-design-from-url` |

Two of these deserve a standing habit:

- **`search_vercel_documentation`** — the anti-staleness tool. Vercel's surface moves faster than any
  model's training cutoff. Before asserting that a flag, endpoint, or product behaves a certain way,
  query it. This is cheaper than being wrong.
- **Toolbar threads** — comments a reviewer or client leaves on a preview deployment. They can be
  read, answered, reacted to, and resolved without a browser. Nothing in the official pack mentions
  this exists.

## Agent observability is about `eve`

`list_agent_runs` and friends are the observability layer for agents built with the **eve** framework
— not a general "AI agent" feature, and not the same thing as **Vercel Agent** (AI code review and
incident investigation). Both are covered by official skills. Route there rather than explaining them
here.

## Routing table to the official pack

Load the official skill; do not paraphrase it.

| Task | Official skill |
|---|---|
| App Router, Server Components, Server Actions, data fetching | `nextjs` |
| Upgrading Next.js, codemods | `next-upgrade` |
| PPR, `use cache`, `cacheLife`, `cacheTag` | `next-cache-components` |
| Chat, generation, structured output, tool calling, embeddings | `ai-sdk` |
| Model routing, provider failover, cost tracking | `ai-gateway` |
| Durable workflows, pause/resume, retries | `workflow` |
| Durable agents, sessions, skills, channels | `eve` |
| AI code review, incident investigation | `vercel-agent` |
| Serverless / Edge Functions, Fluid Compute, cron | `vercel-functions` |
| Request interception, rewrites, redirects | `routing-middleware` |
| Cache hit rate, stale content, revalidation, ISR | `cdn-caching` |
| Ephemeral key-value cache with tag invalidation | `runtime-cache` |
| Blob, Edge Config, Neon, Upstash | `vercel-storage` |
| WAF rules, rate limiting, Attack Mode, bot management | `vercel-firewall` |
| Ephemeral microVMs for untrusted code | `vercel-sandbox` |
| Scoped OAuth tokens for third-party services | `vercel-connect` |
| Clerk, Descope, Auth0 | `auth` |
| Component installation, registries, theming | `shadcn` |
| Multi-zone, path-based routing across projects | `microfrontends` |
| Turborepo SaaS starter | `next-forge` |
| Bundler configuration, HMR | `turbopack` |
| Third-party integrations, commerce, payments, CMS | `marketplace` |
| Multi-platform chat bots | `chat-sdk` |
| Deploy, promote, roll back, `--prebuilt`, CI files | `deployments-cicd` |
| `.env` files, `vercel env`, OIDC tokens | `env-vars` |
| CLI commands generally | `vercel-cli` |

**Standing caveat:** each of those assumes CLI or dashboard. When one of them says "check the
dashboard" for live state, check whether an MCP tool answers instead.

## Corrections to the official pack

Cited by file and quoted string rather than line number, because line numbers drift between releases.
Re-verify these on each major plugin upgrade — if upstream fixes one, delete it from here.

| Where | It says | Reality |
|---|---|---|
| `commands/status.md` | `Web Analytics data \| **Dashboard only**` | `get_web_analytics` returns visitors, pageviews and aggregates over MCP. |
| `.mcp.json` | *"Read-only in initial release: search docs, list projects/deployments, inspect logs"* | The live server also deploys, buys domains and posts Toolbar replies. |
| pack-wide | (silence on Toolbar threads) | Six MCP tools exist for reading and answering preview comments. |

## Deployment conventions

These are operating rules, not Vercel documentation.

- **Never run a local production build to "check" a deploy.** Build on the platform; read
  `get_deployment_build_logs` or `get_runtime_errors` for what actually happened. A local build
  exercises a different environment and proves nothing about the artifact that shipped.
- **Non-interactive deploys need an explicit scope.** Without `--scope`, a deploy run outside an
  interactive shell can resolve to the wrong team or fail outright.
- **`vercel env add` marks the variable sensitive.** A later `vercel env pull` then returns
  `[SENSITIVE]` instead of the value. This is not corruption and not a lost secret, and it is not a
  reason to regenerate the variable — rotating a working secret to resolve a display quirk risks
  breaking production for nothing.

  **If the variable is `NEXT_PUBLIC_*`:** the value is inlined at build time and present in the
  served HTML. Get the production URL from `get_deployment` / `list_deployments`, then fetch that
  page and grep for the value. Use `curl` via Bash, or `WebFetch` — *not*
  `web_fetch_vercel_url`, which is scoped to Vercel's own documentation pages and is filed under
  the Docs group for that reason.

  **If the variable is server-only:** that verification path does not exist — the value never
  reaches the client, by design. No MCP tool reads environment variable values either; env is
  entirely CLI-side. Recover it from wherever it was issued (the provider's own dashboard, a
  password manager) before considering rotation, and say plainly that the value cannot be read back
  rather than implying it is gone.
- **A branch-scoped review domain is REST-only.** Neither the CLI nor the MCP creates a domain bound
  to a git branch; it requires a REST call carrying the branch name. Related traps: SSO protection
  settings that exempt "custom domains" do **not** exempt a branch domain, and environment variables
  scoped to Production only will degrade the preview **silently** rather than erroring.
- **When an external CDN serves the media, `public/` belongs in `.gitignore`.** Committing local
  copies produces double hosting: the platform ships duplicates while the CDN serves the real assets.
- **Do not deploy to a client's real domain before the engagement is paid.** A demo deployment on a
  platform-generated URL is the selling instrument; the production domain is the deliverable.
