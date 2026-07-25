# Ship 2026 products — the silent failures

**Dated 2026-07-25.** Four products announced at Vercel Ship 2026 have **zero coverage** in the
official Vercel plugin (verified at v0.45.1, which is also the current upstream release): Dockerfile
support, the Vercel Container Registry, Vercel Services, and Vercel Passport.

This file is deliberately **not** a tour of those products — upstream will document them, and a
product tour written today becomes a stale contradiction the moment it does. What is recorded here
is only the set of behaviors that **fail silently**, because those stay true and are what actually
costs an afternoon.

**Delete this file when the official plugin covers these products.** Check on each plugin upgrade.
For anything beyond the traps below, query `search_vercel_documentation` — it is current where this
file is frozen.

## Vercel Services

Multiple deployable units — a frontend and a backend, several backends — declared in one
`vercel.json` and deployed together.

```json
{
  "services": {
    "my_frontend": { "root": "frontend/" },
    "my_backend":  { "root": "backend/", "entrypoint": "main:app" }
  },
  "rewrites": [
    { "source": "/api/(.*)", "destination": { "service": "my_backend" } },
    { "source": "/(.*)",     "destination": { "service": "my_frontend" } }
  ]
}
```

> **Services are internal by default.** A service with no `rewrites` entry pointing at it is
> unreachable from the internet. It deploys, it reports success, and it answers nothing. There is no
> error to find — the symptom is a request that never arrives. Any service meant to take public
> traffic needs an explicit rewrite whose `destination` is `{ "service": "<name>" }`.

> **Two configuration shapes exist.** The stable form is `services` with `root` / `entrypoint`. An
> experimental form uses `experimentalServices` with `entrypoint` / `routePrefix`, and requires the
> project's framework setting to be **Services** before it takes effect at all. Writing the
> experimental keys without that setting, or mixing the two shapes, does not error — it is ignored.
> Confirm which form the docs currently describe before writing either.

**Bindings** declare a dependency between services and inject the target's URL as an environment
variable, which removes hardcoded inter-service URLs:

```json
{
  "services": {
    "orders": {
      "root": "services/orders/",
      "framework": "express",
      "bindings": [
        { "type": "service", "service": "inventory", "format": "url", "env": "INVENTORY_URL" }
      ]
    },
    "inventory": { "root": "services/inventory/", "framework": "fastapi", "entrypoint": "main:app" }
  }
}
```

Local: `vercel dev`, or `vercel dev -L` to run without Vercel Cloud authentication.

## Dockerfile and the Container Registry

A `Dockerfile.vercel` in the project is built, stored, deployed and autoscaled on Fluid compute. A
service can also opt in with `"runtime": "container"`.

Registry operations live under `vercel vcr`:

```bash
vercel vcr add <name>                      # create a repository
vercel vcr ls [--project <p>] [-F json]    # list repositories
vercel vcr image inspect <repo> <image-id> # inspect an image and its layers
vercel vcr login docker|podman|buildah     # authenticate a container tool
```

Registry host: `vcr.vercel.com/<team-slug>/<project-slug>/<repository>:<tag>`. REST equivalent:
`GET /v1/vcr/repository`.

> **`vcr login` mints a token that expires in 12 hours.** It is short-lived, project-scoped and
> OIDC-based. A push that worked this morning fails this evening with an auth error that looks like a
> permissions problem rather than an expiry. Re-run `vcr login` before diagnosing anything else.

> **`podman` and `buildah` are first-class**, not workarounds — both are named in the CLI reference
> alongside Docker. There is no need to install a Docker daemon to use the registry.

Vercel's own build recommendation is Buildx with zstd compression:

```bash
docker buildx build \
  --platform linux/amd64 \
  --output "type=image,name=vcr.vercel.com/<team>/<project>/<repo>:latest,push=true,oci-mediatypes=true,compression=zstd,compression-level=3,force-compression=true" \
  .
```

> **Cost lives in Active CPU and provisioned memory, not per-request.** A container that stays warm
> bills continuously. See [plans-and-limits.md](plans-and-limits.md) — on the free tier this is the
> binding constraint, and the plan gating of these products is not documented either way.

## Not covered here

**eve** (the agent framework) and **Vercel Connect** already have official skills — route there
instead of duplicating. **Vercel Passport**, the security dashboard, and own-tenant AWS deployment
are enterprise-tier concerns; nothing in this skill addresses them.
