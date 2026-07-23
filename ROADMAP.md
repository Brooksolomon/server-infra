# EthioHost — roadmap

Turning this Hetzner box from "SSH in and manually docker-compose" into a
self-serve, GitHub-connected deploy platform (Vercel/Railway-style). Long-term
goal: sell access to it as a hosting service.

This file tracks status across phases so work can resume from any point
without re-deriving context. Update it as phases complete or scope changes.

## Locked decisions

- **Name / domain**: single-domain model (Railway-style, not Vercel's split).
  `ethiodeploy.com` — being purchased on Porkbun — used for **both** the
  marketing/dashboard site AND project subdomains: `<project-slug>.ethiodeploy.com`.
  Naming decided, no further action needed here.
- **Reverse proxy**: Traefik (Docker-label-based, auto TLS via ACME DNS-01),
  replacing the current static-Caddyfile setup. This is Phase 0.
- **DNS-01 provider**: Cloudflare (free API, needed for wildcard cert issuance
  — wildcard certs can't use HTTP-01). Traefik runs **two** cert resolvers:
  `cloudflare` (DNS-01, for `*.ethiodeploy.com` wildcard only — requires the
  domain's zone to live in this Cloudflare account) and `letsencrypt`
  (HTTP-01, for any other single domain — legacy projects like
  `telegramsearchengine.dev`/`solocodes.dev`, and future custom domains that
  aren't on this Cloudflare account). Don't put legacy/custom-domain projects
  on the `cloudflare` resolver unless their zone is actually added here first.
- **Build system**: Nixpacks (Railway's open-source builder). Auto-detects
  Next.js/Node/Python/etc from repo — **no Dockerfile required in user repos**.
  Falls back to repo's own Dockerfile if present.
- **Control-plane DB**: self-hosted Postgres on the same Hetzner box, internal
  network only (not exposed to the `proxy` network), regular `pg_dump` backups
  to Hetzner object storage.
- **Job queue**: Redis + BullMQ (builds are async, need status/retries).
- **GitHub integration**: GitHub App (not plain OAuth) — installed per-repo,
  gives push webhooks + scoped clone tokens.
- **Control plane app**: Next.js, GitHub OAuth login for dashboard.

## Architecture (target end state)

```
GitHub push ─► webhook ─► Control Plane (enqueues job)
                              │
                              ▼
                        Build Worker (runner daemon, docker socket access)
                        git clone → nixpacks build → tag :sha
                              │
                              ▼
                        Deploy Worker
                        docker run w/ Traefik labels (slug.<domain>)
                        health check → swap old container → remove old
                              │
                              ▼
                        Traefik (dynamic, label-based routing + wildcard TLS)
```

Key shift from today: **no manual SSH, no manual Caddyfile edits, no manual
`docker compose` per deploy.** Runner daemon on the box does that work,
driven by jobs from the control plane.

## Phases

### Phase 0 — proxy swap (DONE)
Replaced static Caddy/Caddyfile with Traefik (Docker provider, auto-discovers
containers via labels, auto wildcard TLS). Both live sites confirmed working
on Traefik with valid Let's Encrypt certs; old Caddy fully retired.

- [x] Traefik service in `docker-compose.yml`, Docker socket mount, dual cert
      resolvers (see Locked decisions)
- [x] `telegram-search-engine-web` migrated + verified live (`telegramsearchengine.dev`)
- [x] `portfolio-web` migrated + verified live (`solocodes.dev`)
- [x] Caddy service + Caddyfile removed
- [x] README.md describes label-based routing
- [x] Cloudflare zone + scoped API token set up for `ethiodeploy.com`, `.env` filled
- [x] Applied on the actual Hetzner server, both sites verified

**Gotchas hit during rollout** (relevant to Phase 1 — the runner needs to
handle these automatically for every future deploy, not just these two):
- **Docker Engine 29.x API mismatch**: Traefik v3.1/v3.3 hardcode an old
  default Docker API version internally and ignore `DOCKER_API_VERSION` env
  entirely — not a negotiation issue, just don't read it. Fixed by bumping to
  `traefik:3.7.8` (confirmed working with Docker Engine 29.6.0 / API 1.55,
  daemon minimum 1.40). If Docker gets upgraded again, may need a newer
  Traefik tag.
- **DNS-01 only works for zones actually in the Cloudflare account.** Trying
  to issue a cert for a domain not on Cloudflare fails with "zone could not
  be found" — silently falls back to Traefik's self-signed cert (browser
  shows `ERR_CERT_AUTHORITY_INVALID`), no obvious error unless you grep logs.
  Hence the two-resolver split (see Locked decisions).
- **Next.js standalone + Docker HOSTNAME collision**: Docker auto-injects
  `HOSTNAME=<container-id>` into every container. Next.js's standalone
  server reads `process.env.HOSTNAME` as its bind address instead of
  defaulting to `0.0.0.0`, so it silently listens on loopback only —
  process looks healthy (logs "Ready"), DNS resolves fine, but any other
  container gets connection-refused (502 from Traefik). Fix: explicitly set
  `environment: [HOSTNAME=0.0.0.0]` on the web service. **This will hit
  every Next.js project deployed through the platform** — the Phase 1
  runner must inject this automatically (or nixpacks/build step must set
  it), not rely on each project's compose file doing it manually.
- **Traefik static config (cert resolvers, providers) only reloads on full
  recreate** (`docker compose down && up -d`), not `restart` — `restart`
  reuses the same container/args, so config changes referenced by CLI flags
  won't take effect from a plain restart. This tripped up debugging (kept
  seeing a stale cached log line and assumed nothing had changed).

### Phase 1 — functional MVP (single user: you)
Control plane lives in its own repo: `ethiodeploy` (sibling to `server-infra`,
not folded into it). Scaffold committed locally there (not yet pushed — no
remote created yet, no `gh` CLI in this environment).

- [x] Postgres schema (Prisma): `User`, `Project`, `Deployment`, `EnvVar`
- [x] Redis + BullMQ job queue (`lib/queue.ts`)
- [x] Runner daemon skeleton (`runner/`, docker socket access) — job →
      git clone via GitHub App installation token → `nixpacks build` →
      `docker run` w/ Traefik labels → old container removed. Sets
      `HOSTNAME=0.0.0.0` on every deployed container automatically (see
      Phase 0 gotchas) — solved once, applies to every future deploy.
- [x] Control-plane webapp skeleton: GitHub OAuth login (via the GitHub App's
      own OAuth), project list, new-project form, deployment history page
- [x] Push webhook route (`app/api/webhooks/github/route.ts`) — verifies
      signature, enqueues deploy job on push to the project's default branch
- [x] `next build` passes clean, typecheck clean
- [x] Local dev verified working: Postgres+Redis via `docker-compose.dev.yml`,
      migration applied, `npm run dev` and `npm run runner:start` both boot
      clean. Gotchas hit (see commit `692c27e` in `ethiodeploy`):
  - This machine already runs two other local Postgres instances (on 5432
    and 5433) — dev Postgres moved to host port 5434. If `prisma migrate`
    ever throws a confusing `P1010`/`(not available)` auth error again on a
    dev machine, check for a port collision with an unrelated Postgres
    before assuming it's a real Prisma bug — that's what it was here.
  - `tsx` (runner's TS runtime) has a resolver bug (`resolveTsPaths`) that
    crashes on any dependency chain touching an ESM-only package with no
    `require` export condition (hit via `execa` → `unicorn-magic`). Fixed by
    dropping `execa` for a tiny `node:child_process` wrapper
    (`runner/src/exec.ts`) and giving the runner its own `tsconfig.json`
    without the `@/*` path alias. **Lesson for future runner deps**: avoid
    packages pulling in strict-ESM-only transitive deps, or test them
    against `tsx` before committing to them.
- [ ] Actually create the GitHub App on github.com (manual step, see
      `ethiodeploy/README.md`) and fill real `.env` values — nothing above
      has been exercised end-to-end against a live GitHub App yet
- [x] GitHub remote created (`gh` CLI installed + authenticated), pushed to
      `github.com/Brooksolomon/ethiodeploy` (private)
- [ ] Deploy `ethiodeploy` itself to the Hetzner server per its README
      (joins `proxy` network, `ethiodeploy.com` via the `cloudflare` DNS-01
      resolver since that zone's already in the Cloudflare account)
- [ ] "New Project" form currently takes installation ID as manual text
      input — fine for MVP (you're the only user), revisit before opening
      this up (should list installed repos via the GitHub App API instead)

### Phase 2 — polish
- [ ] Env var / secrets UI (encrypted at rest)
- [ ] Custom domain support per project (CNAME + Traefik cert)
- [ ] Preview deployments per PR/branch
- [ ] Deployment history + one-click rollback to prior sha
- [ ] **Atomic/immutable deployments** (how Vercel and Railway actually do
      rollback): currently the runner mutates in place — stops the old
      container, starts the new one, with a brief gap. Vercel/Railway keep
      each deployment's image/container addressable and "promote to
      production" is just re-pointing the router at a specific past
      deployment — that's what makes their rollback instant. Worth doing
      once there's more than one project depending on zero-downtime deploys.
- [ ] **Health-check-gated zero-downtime swap**: build the new container,
      wait for it to pass a health check *before* killing the old one, then
      swap — removes the current stop-then-start gap entirely. Render and
      Railway both do this. Currently just README-documented as a manual
      pattern (`server-infra/README.md`); the runner should do it
      automatically for every deploy.

### Phase 3 — multi-tenant safety
Before letting anyone else run code on the box:
- [ ] Per-container resource limits (cpu/mem caps), no `--privileged`
- [ ] Isolate builds from prod runtime (separate build box, or rootless
      docker / gVisor at minimum)
- [ ] Network egress restrictions for tenant containers
- [ ] Per-tenant container namespacing (no name/port collisions)
- [ ] **Stronger runtime isolation — Firecracker microVMs** (what Railway
      actually runs workloads in, same tech AWS Lambda uses under the hood):
      real security boundary between different customers' code on shared
      hardware, well beyond what plain `docker run` gives you. Bigger lift
      than gVisor/rootless above — evaluate once there's real multi-tenant
      demand, not needed for single-user MVP.

### Phase 3.5 — DDoS protection
DNS-only (grey cloud) is deliberate for now — see Phase 0 notes, keeps Traefik
as sole TLS terminator, avoids paid-tier wildcard proxy issues, simpler to
debug during buildout. Revisit once stable + before opening to real users:
- [ ] Flip Cloudflare proxy to ON (orange cloud) for `@` and `*` A records —
      needs Advanced Certificate Manager or Cloudflare for SaaS (paid) for
      wildcard proxying to work correctly
- [ ] Configure Cloudflare WAF rules / rate limiting
- [ ] Verify Traefik still gets real client IPs (via `X-Forwarded-For` /
      Cloudflare's IP ranges trusted in Traefik config) once proxied — a
      naive flip breaks IP-based logging/rate-limiting on the origin side
- [ ] Consider Cloudflare "Under Attack Mode" runbook for active incidents

### Phase 4 — billing
- [ ] Stripe integration (subscription or usage-based)
- [ ] Plan tiers → resource quota mapping
- [ ] Metering (cpu-seconds, bandwidth)
- [ ] Auto-suspend on nonpayment / quota breach

### Phase 5 — scale past one box
- [ ] Multi-server orchestration (Nomad or k8s)
- [ ] Shared image registry
- [ ] Load balancer across boxes
- [ ] Shared build cache
- [ ] Multi-region (Vercel/Railway both run multiple regions — pick per
      project or auto-nearest). Only matters once boxes exist in more than
      one place; not a concern while everything's on one Hetzner server.

## Open items / needs decision
- Naming/domain done (`ethiodeploy.com`, single-domain model).
- Control-plane repo location done: `ethiodeploy`, sibling repo. See Phase 1
  for what's left before it's actually running.
