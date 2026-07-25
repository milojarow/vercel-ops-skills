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

```
https://vercel.com/<teamSlug>/<projectName>/analytics
https://vercel.com/<teamSlug>/<projectName>/speed-insights
```

`teamSlug` comes from `list_teams`; `projectName` comes from `.vercel/project.json`. Never hardcode
either, and never substitute the local directory name for `projectName` — see the walls below.

## Reading without a browser

`get_web_analytics` has two modes.

**`count`** returns one total:

```
projectId, teamId, mode: "count", since, until
→ { data: { visitors, pageviews } }
```

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

## Walls

> **The local directory name is not the Vercel project name.** A repo checked out as `foo` is
> commonly registered as `website-foo`, `foo-prod`, or a name chosen when the project was created.
> Assembling the dashboard link from the directory name produces a 404, which reads to the user as
> "you set it up wrong". Read `projectName` from `.vercel/project.json`.

> **Data is collected in production only.** Immediately after enabling, both the dashboard and the
> API report zero. Preview and local traffic never counts. This is expected behavior — say so
> explicitly instead of diagnosing a problem that does not exist. If the site genuinely has no
> traffic yet, the honest answer is "it's wired correctly and waiting for visitors", not a debugging
> session.

> **Enabling and instrumenting are two different steps.** The package plus `<Analytics />` without
> `vercel project web-analytics` collects nothing; enabling the product without the component in the
> layout also collects nothing. Both are required, and neither errors when the other is missing.

> **A partial rollout is invisible.** `<Analytics />` belongs in the **root** layout. Placed in a
> nested layout, it silently reports only the routes beneath it, and the numbers look like a traffic
> problem rather than an instrumentation gap.
