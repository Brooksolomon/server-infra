# server-infra — shared reverse proxy

This is **server-level infrastructure**, separate from any single project. It
runs **one** Traefik reverse proxy that owns ports 80/443 for the whole
machine and routes incoming requests to each project's container **by domain
name**, with automatic HTTPS for every domain — including brand-new
subdomains, with zero manual config on this side.

Keep this folder on the server (or a private repo). It is **not** part of any
project's open-source repo.

```
                          ┌─► tgsearch.example.com   → telegram-search-engine-web
Internet ─► shared Traefik ─┼─► project2.example.com   → project2-web
            (owns 80/443) └─► project3.example.com   → project3-web
```

## Why this exists

Only one process can bind ports 80/443 on a server. If every project ran its
own proxy, the second one would fail with `port is already allocated`. So
instead one shared Traefik fronts everything and forwards by hostname. Unlike
a static Caddyfile, Traefik watches the Docker socket and picks up routing
straight from **labels on each project's own container** — adding a project
never means editing a file in this repo.

## What's in here

| File                 | Purpose |
| -------------------- | ------- |
| `docker-compose.yml` | Runs the shared Traefik container (ports 80/443, on the `proxy` network). |
| `.env.example`       | Template for Cloudflare ACME credentials. Copy to `.env` (gitignored). |

---

## One-time server setup

```bash
# 1. Create the shared network every project will join.
docker network create proxy

# 2. Configure ACME credentials.
cd server-infra
cp .env.example .env
# fill in ACME_EMAIL, CF_API_EMAIL, CF_DNS_API_TOKEN (Cloudflare API token,
# Zone.DNS edit permission scoped to your domain)

# 3. Start the shared proxy.
docker compose up -d
```

> **DNS-01, not HTTP-01:** certs are issued via Cloudflare DNS challenge, not
> by answering HTTP requests on port 80. This is required for wildcard
> subdomains and means a cert can be issued for a domain even before it's
> pointed anywhere — but it does mean `CF_DNS_API_TOKEN` must be scoped
> correctly (Zone:DNS:Edit) or issuance fails silently-ish (check `docker
> compose logs -f traefik`).

> **DNS note:** the compose file also pins Traefik's DNS to `1.1.1.1` /
> `8.8.8.8`. Without this, Traefik can't resolve Let's Encrypt from inside the
> container on some hosts (the host's `127.0.0.53` stub resolver is
> unreachable from containers). Leave it in.

---

## Add a project

Each project must (a) join the external `proxy` network and (b) carry Traefik
labels on its web container declaring its domain.

### 1. Point the domain at the server

In your DNS provider, add an **A record**: `app.example.com → <server IP>`
(and optionally an `AAAA` record for IPv6). Wait until `dig +short app.example.com`
returns the server IP.

### 2. Give the project a `docker-compose.shared.yml`

In the project repo, add an override that drops its bundled proxy (if any),
joins the proxy network, and declares Traefik labels. Example (mirrors what
`telegram-search-engine` ships):

```yaml
services:
  web:
    container_name: myproject-web        # unique across the server
    networks: [default, proxy]
    labels:
      - traefik.enable=true
      - traefik.docker.network=proxy
      - traefik.http.routers.myproject.rule=Host(`app.example.com`)
      - traefik.http.routers.myproject.entrypoints=websecure
      - traefik.http.routers.myproject.tls.certresolver=cloudflare
      - traefik.http.services.myproject.loadbalancer.server.port=3000
networks:
  proxy:
    external: true
```

Use the **exact container name** and the port the app listens on. The router
name (`myproject` above) must be unique across the server — reuse it
consistently across all the labels for one service.

### 3. Start the project on the shared network

```bash
cd /path/to/project
docker compose -f docker-compose.yml -f docker-compose.shared.yml \
    --env-file .env.prod up -d --build --scale caddy=0
```

`--scale caddy=0` keeps the project's own bundled proxy (if it has one) from
starting — the shared Traefik handles HTTPS and routing.

That's it — **no edit to this repo, no reload command.** Traefik sees the new
container's labels the moment it starts and fetches an HTTPS cert on first
request (~30s).

---

## The currently-deployed project

`telegram-search-engine` runs behind this proxy. Its web container is named
`telegram-search-engine-web`, routed via the labels in its own
`docker-compose.shared.yml` to `telegramsearchengine.dev`.

Its `api`, `meili`, and `postgres` stay on the project's private network —
only the web container is exposed to the proxy network.

---

## Deploying a new version of a project

Rebuilding swaps the web container, which causes a **few seconds** of
downtime for that one site while the new container boots. For a
near-zero-downtime swap, build first, then replace only the web container:

```bash
cd /path/to/project
docker compose -f docker-compose.yml -f docker-compose.shared.yml \
    --env-file .env.prod build web
docker compose -f docker-compose.yml -f docker-compose.shared.yml \
    --env-file .env.prod up -d --no-deps web
```

The image is built before the running container is touched, so the gap
shrinks to the container's boot time. Other projects are unaffected — Traefik
just re-points the label-matched route to the new container once it's up.

---

## Operations

```bash
# see everything running on the box
docker ps --format '{{.Names}}'

# shared proxy logs (cert issuance, routing errors)
cd server-infra && docker compose logs -f traefik

# restart the shared proxy (brief downtime for ALL sites)
docker compose restart traefik
```

No reload command needed after adding/removing a project — Traefik picks up
label changes live via the Docker socket.

## Gotchas

- **Container names must be unique** across the whole server — two `web`
  containers would collide. Always set `container_name: <project>-web`.
- **Router names must be unique** across the server too (the
  `traefik.http.routers.<name>.*` label suffix) — reuse `<project>` as the
  name consistently.
- If a cert won't issue: check DNS points here, ports 80/443 are open (no
  Hetzner/ufw firewall blocking them), `CF_DNS_API_TOKEN` has Zone:DNS:Edit
  on the right zone, and check `docker compose logs -f traefik`.
- Traefik's dashboard/API is **not exposed** in this config — labels only, no
  extra attack surface. Enable `--api.insecure` only for local debugging,
  never in the compose file committed here.
