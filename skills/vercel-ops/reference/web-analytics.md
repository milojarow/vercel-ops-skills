# Web Analytics — enable, read, and instrument

Vercel Web Analytics counts visitors, pageviews and custom events. Speed Insights reports Core Web
Vitals from real traffic. Both are enabled the same way and both are fully drivable from an agent
session — **no dashboard step exists in this workflow.**

## Enabling is a CLI command

```
vercel project web-analytics  [name]   Enable Web Analytics for a project
vercel project speed-insights [name]   Enable Speed Insights for a project
    -F, --format <FORMAT>              Specify the output format (json)
```

The CLI's own help labels `--format json` as *"non-interactive / agents"* — it is built to be driven
programmatically. Recent CLI versions also detect an agent and default to `--non-interactive`.

This is the single most useful fact in this file, because the official plugin's status command
asserts that Web Analytics is dashboard-only, which sends anyone following it to a browser for a step
that is one command.

## Runbook — add analytics to a project

1. **Establish identity.** Read `.vercel/project.json` for `projectId`, `orgId` and `projectName`.
   If the file is absent, run `vercel link` first — do not guess the project.
2. **Install the package.** `npm i @vercel/analytics`
3. **Render the component** in the root layout. For the Next.js App Router:

   ```jsx
   import { Analytics } from '@vercel/analytics/next';

   export default function RootLayout({ children }) {
     return (
       <html lang="es">
         <body>
           {children}
           <Analytics />
         </body>
       </html>
     );
   }
   ```

   The import subpath is framework-specific — `@vercel/analytics/next`, `/react`, `/remix`. A
   non-framework page uses the `inject()` form or the script tag. Check
   `search_vercel_documentation` rather than assuming for anything that is not Next.js.
4. **Enable the product.** `vercel project web-analytics <projectName> --format json`
5. **Ship it.** Commit and push; the platform builds. Do not build locally to verify.
6. **Hand back the link** (see below). State plainly that data appears only after real production
   traffic.

For Speed Insights the shape is identical: `npm i @vercel/speed-insights`, render `<SpeedInsights />`,
then `vercel project speed-insights <projectName> --format json`.

## Link assembly

Single source of truth: **"Identity is derived, never stored" in `../SKILL.md`**. It holds the link
template, where each value comes from, the `orgId` = `teamId` equivalence, and what to do when no
directory was named. Do not restate it here — one copy, so the two cannot drift apart.

## Reading without a browser

`get_web_analytics` has two modes.

**`count`** returns one total:

```
projectId, teamId, mode: "count", since, until
→ { data: { visitors, pageviews } }
```

`since` and `until` accept a date string, an ISO 8601 timestamp, or a Unix timestamp in
milliseconds — `"2026-07-01"` and `"2026-07-01T00:00:00.000Z"` both work. They are supplied together
or not at all. The REST equivalents document ISO 8601 specifically, so prefer it when in doubt.

> **The window cannot predate enablement.** Counts run "since Web Analytics was enabled". Query a
> range that starts before that moment and the missing days read as zeros, not as an error. A
> month-over-month comparison across the enablement date is meaningless — say so rather than
> reporting a fabricated decline.

**`aggregate`** groups rows. It requires `since`, `until` **and** `by`, and accepts one or two
dimensions:

| Dimension family | Values |
|---|---|
| Time buckets | `hour`, `day`, `week`, `month`, `year` |
| Page | `route`, `requestPath` |
| Source | `referrerHostname`, UTM fields |
| Audience | `country`, `deviceType`, `osName`, `browserName` |
| Events | `eventName`, `eventData/<property>`, `flags/<name>` |

`filter` accepts an OData expression — `requestPath eq '/pricing' and country eq 'MX'`,
`eventName eq 'signup'`, `eventData/plan eq 'pro'`. `limit` caps distinct rows (default 10); the
remainder collapses into an "Others" bucket, so a low limit can hide the long tail.

`dataset` selects `visits` (default, automatic pageviews) or `events` (custom events).

The same data is available over REST at `/v1/query/web-analytics/visits/count`,
`/visits/aggregate`, `/events/count` and `/events/aggregate` — useful in a script, unnecessary in a
session where the MCP is connected.

## Custom events

Pageviews say a page was seen. Custom events say a **business action happened**, which is the number
worth reporting to whoever paid for the site.

```jsx
'use client';
import { track } from '@vercel/analytics';

export function OrderButton({ href }) {
  return (
    <a href={href} onClick={() => track('order_click', { channel: 'whatsapp' })}>
      Pedir por WhatsApp
    </a>
  );
}
```

Then query it:

```
dataset: "events", mode: "count", filter: "eventName eq 'order_click'"
dataset: "events", mode: "aggregate", by: ["day"], filter: "eventName eq 'order_click'"
```

Guidelines that keep the data usable:

- **Name events after the business action**, not the widget — `order_click`, not `button_3`.
- **Keep property cardinality low.** Properties are dimensions; one property per unbounded value
  (an order ID, a timestamp) makes aggregation useless.
- **`track()` is client-side.** The component that calls it needs the client directive. Keep that
  component small so the directive does not spread up the tree.
- Custom events are recorded in production only, exactly like pageviews.

## When it reads zero

The most common support question after running the runbook, and usually not a bug. Every cause below
fails **silently** — nothing errors, nothing warns. Work the list in order and stop at the first hit.

1. **Was the traffic production traffic?** Local and preview visits never count. The user refreshing
   their own `localhost` or a preview URL produces exactly this symptom.
2. **Did a production deploy ship after the component was added?** Check with `list_deployments` /
   `get_deployment`. The code has to be live, not merely committed.
3. **Are both halves wired?** The package plus `<Analytics />` without
   `vercel project web-analytics` collects nothing. Enabling the product without the component in
   the layout also collects nothing. Neither half errors when the other is missing.
4. **Is `<Analytics />` in the *root* layout?** In a nested layout it silently reports only the
   routes beneath it — which looks like a traffic problem rather than an instrumentation gap.
5. **Does a `beforeSend` hook drop the event?** `<Analytics beforeSend={…} />` (and the
   `webAnalyticsBeforeSend` global in non-React setups) can return `null` to redact an event. An
   over-broad URL match there discards traffic with no trace. Grep for it before concluding anything.
6. **Does the query window predate enablement?** See the note above — pre-enablement days read as
   zeros.
7. **Only then** look at the deployment itself: `get_deployment_build_logs` and `get_runtime_errors`.

If 1–6 are clean and the site is genuinely new, the honest answer is *"it's wired correctly and
waiting for visitors"* — not a debugging session. Say that plainly.

## Walls

> **The local directory name is not the Vercel project name.** A repo checked out as `foo` is
> commonly registered as `website-foo`, `foo-prod`, or a name chosen when the project was created.
> Assembling the dashboard link from the directory name produces a 404, which reads to the user as
> "you set it up wrong". Read `projectName` from `.vercel/project.json`.

> **Data is collected in production only.** This is the root of most zero-data reports; it is the
> first item in the diagnosis order above for that reason.

## What this file does not know

Stated so nobody fills these in from intuition. If one of them decides an answer, query
`search_vercel_documentation` and then record what came back here.

- **Ingestion latency.** How long between a real visit and it appearing in the dashboard or API is
  not documented here. Do not quote a number. If a user asks "how long until I see it", say it is
  not instantaneous and check the docs rather than inventing a figure.
- **Whether `vercel project web-analytics` is idempotent**, and whether any command reports the
  current enabled/disabled state. Re-running it on an already-enabled project has not been verified
  as safe here.
- **Content blockers.** Whether and how browser extensions suppress the script is not established.
  The `scriptSrc` prop and the self-hosted script path exist as knobs, but do not present blocking as
  the diagnosis without evidence.
