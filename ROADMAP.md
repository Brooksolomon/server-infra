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
  — wildcard certs can't use HTTP-01).
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

### Phase 0 — proxy swap (IN PROGRESS)
Replace static Caddy/Caddyfile with Traefik (Docker provider, auto-discovers
containers via labels, auto wildcard TLS). Foundation for everything else —
without this, every new project still needs a manual file edit + reload.

- [x] Add Traefik service to `docker-compose.yml`, Docker socket mount,
      Cloudflare DNS-01 resolver config — done locally, not yet applied on
      the actual server
- [x] Migrate `telegram-search-engine-web` from Caddyfile block to Traefik
      labels on its own compose file (`tg-discovery/docker-compose.shared.yml`)
      — file changed locally, **not yet deployed** (this is the one live prod
      site — highest-risk step, apply carefully, verify zero downtime)
- [x] Remove Caddy service + Caddyfile locally
- [x] Update README.md to describe label-based routing instead of Caddyfile
      edits
- [ ] Create Cloudflare account/zone for final domain, generate scoped API
      token (Zone:DNS:Edit), fill in real `server-infra/.env`
- [ ] Apply on the actual Hetzner server (SSH required — not done from this
      session, needs manual run or explicit access grant): bring up new
      Traefik stack, redeploy tg-discovery with new labels, confirm
      `telegramsearchengine.dev` still resolves + HTTPS valid, then remove
      old Caddy container

### Phase 1 — functional MVP (single user: you)
- [ ] GitHub App registration + install flow + webhook signature verification
- [ ] Postgres schema: `users`, `projects`, `deployments`, `env_vars`
- [ ] Redis + BullMQ job queue
- [ ] Runner daemon (docker socket access) — job = `{repoUrl, sha, slug}` →
      `nixpacks build` → tag `:sha` → `docker run` w/ Traefik labels →
      health check → swap old container
- [ ] Control-plane webapp: GitHub OAuth login, connect repo, create project
      (repo + branch → auto slug), deployment list + status + streamed logs
- [ ] Push webhook → build job → live status in dashboard

### Phase 2 — polish
- [ ] Env var / secrets UI (encrypted at rest)
- [ ] Custom domain support per project (CNAME + Traefik cert)
- [ ] Preview deployments per PR/branch
- [ ] Deployment history + one-click rollback to prior sha

### Phase 3 — multi-tenant safety
Before letting anyone else run code on the box:
- [ ] Per-container resource limits (cpu/mem caps), no `--privileged`
- [ ] Isolate builds from prod runtime (separate build box, or rootless
      docker / gVisor at minimum)
- [ ] Network egress restrictions for tenant containers
- [ ] Per-tenant container namespacing (no name/port collisions)

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

## Open items / needs decision
- Naming/domain done (`ethiodeploy.com`, single-domain model).
- Which repo hosts the control-plane app + runner daemon — new repo, or
  folded into this one? Currently assumed **new repo**, this repo (`server-infra`)
  stays scoped to proxy + box-level infra.
