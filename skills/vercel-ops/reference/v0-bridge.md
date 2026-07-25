# v0 bridge — deciding, and landing the output

v0 is Vercel's generative UI product. This file is deliberately **not** a v0 manual: its feature set
changes monthly, and any walkthrough written today is stale by the next release. What is durable is
the decision of when to reach for it, and what has to happen to its output before it can live in an
established codebase.

For anything about current v0 features, query `search_vercel_documentation` rather than relying on
this file or on training data.

## When v0 wins, and when it loses

| Situation | Verdict |
|---|---|
| A client-facing demo, where the artifact's job is to sell rather than to ship | **v0**. Speed dominates; conventions barely matter because the code is disposable. |
| Exploring several visual directions before committing | **v0**. Cheap parallel exploration is exactly what it is good at. |
| A greenfield project with no established conventions yet | **v0**, then adopt its output as the baseline. |
| A feature inside an existing repo with settled conventions | **Not v0.** The conversion tax exceeds the generation savings. |
| Anything touching auth, payments, or data access | **Not v0.** Generated code that looks right in those areas is a liability. |

The honest framing for anyone who already ships quickly: **v0 does not make you a faster builder —
it makes demos cheap enough to personalize per prospect.** If building was never the bottleneck, that
is the only axis where it pays.

## Landing rules

v0 emits **shadcn/ui** components — stated in the official Vercel plugin's own platform map. If the
target project uses a different component library, every generation arrives in the wrong dialect.
Four things have to happen before the code is committed.

Before any of them, **determine what the target project actually uses** — do not assume. Read
`package.json` for the UI dependency (`daisyui`, `@radix-ui/*` alongside a `components/ui/`
directory for shadcn, `@mui/*`, and so on), and skim one existing component for the house style.
Landing generated code in the wrong dialect is the failure this whole section exists to prevent, so
spending one file read to confirm the target is never wasted.

### 1. Translate the component library

shadcn/ui composes primitives with utility classes; a class-based library like DaisyUI expresses the
same component as a semantic class. The conversion is mechanical but must actually be done — a
half-converted file inherits both systems and looks broken in a way that is hard to attribute.

```jsx
// v0 output — shadcn/ui
<Button variant="outline" size="sm">Enviar</Button>

// landed — DaisyUI
<button className="btn btn-outline btn-sm">Enviar</button>
```

Do the whole file at once. A mixed file is worse than either pure form.

### 2. Purge unnecessary client directives

v0 generates client components by default because its preview is interactive. Most of what it
produces has no interactivity at all. Every `'use client'` that survives without a reason moves work
to the browser permanently.

Keep the directive only where the component genuinely needs browser APIs, event handlers, or state.
Push it to the smallest leaf that needs it rather than leaving it at the top of a subtree.

### 3. Place files by the project's convention

Generated code arrives in v0's own layout. Move it into the project's structure and rename to the
project's convention before committing, not after. Renaming later means the review reads as a
refactor rather than an addition.

### 4. Write real characters in JSX text

Generated markup sometimes carries escape sequences in text nodes. Inside JSX text — between tags —
a `á` renders as seven literal characters, not as `á`. Accented characters and inverted
punctuation must be written directly. String literals inside attributes and expressions do interpret
escapes, so the bug appears only in visible copy, which is exactly where it is most embarrassing.

### 5. Re-check the exclusion against what the file actually contains

The decision table above is normally consulted *before* generating. Consult it again now, because
only at this point do you know what the generated code really does. A section described as "pricing"
routinely arrives carrying checkout wiring; a "contact form" arrives posting somewhere. Generated
code touching auth, payments, or data access does not land — it gets rewritten by hand, regardless of
how good it looks.

## Removing the conversion tax

Design Systems 2.0 imports an existing design system — from a repo, a package registry, a component
explorer, or a design tool — so generations use the project's real components instead of generic
shadcn. Setting this up once turns rules 1 and 3 above from per-generation work into a one-time
configuration.

This is the difference between v0 as a novelty and v0 as infrastructure. If v0 is going to be used
more than a handful of times, do this first.

## Cost

Pricing as published **2026-07-02**; re-check before quoting it, this is the perishable part of this
file.

| Plan | Price | Included | Notes |
|---|---|---|---|
| Free | $0 | $5 credits/month | 7 messages/day; includes Design Mode, deploy, repo sync |
| Plus | $30/user/month | $30/month + $2 daily on login | All models |
| Business | $100/user/month | $30/month + $2 daily on login | Training opt-out by default |
| Enterprise | custom | — | SSO, RBAC, no training on your data |

Per-model token pricing (input / output per 1M): Mini $1 / $5 · Pro $3 / $15 · Max $5 / $25 ·
Max Fast $10 / $50.

**Evaluate on the free tier first.** Seven messages a day is enough to produce one real demo and
learn whether the output survives the landing rules above. Paying before that answer is known buys
nothing.
