# Server infrastructure — shared reverse proxy

This folder is **server-level infrastructure**, separate from any single project.
It runs ONE Caddy reverse proxy that owns ports 80/443 for the whole machine and
routes incoming requests to each project's container by domain name — with
automatic HTTPS for every domain.

```
                          ┌─► tgsearch.example.com  → tgsearch-web
Internet ─► shared Caddy ─┼─► project2.example.com  → project2-web
            (owns 80/443) └─► project3.example.com  → project3-web
```

Keep this folder on the server (or in a private repo) — it is NOT part of any
project's open-source repo.

## One-time server setup

```bash
# 1. Create the shared network all projects will join.
docker network create proxy

# 2. Start the shared proxy.
cd server-infra
docker compose up -d
```

## Add a project

Each project's compose must (a) join the external `proxy` network and (b) give
its web container a predictable name. For an example see the tg-search project's
`docker-compose.shared.yml`, which names its web container `tgsearch-web`.

**1. Point the project's domain** at this server (DNS A record → server IP).

**2. Start the project on the shared network** (its own Caddy is skipped):

```bash
cd /path/to/project
docker compose -f docker-compose.yml -f docker-compose.shared.yml \
    --env-file .env.prod up -d --build --scale caddy=0
```

**3. Add a routing block** to `server-infra/Caddyfile`:

```caddy
project2.example.com {
	encode gzip
	reverse_proxy project2-web:3000
}
```

**4. Reload the proxy** to pick up the new route (no downtime for other sites):

```bash
cd server-infra
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

That's it — Caddy fetches HTTPS for the new domain on first request.

## Migrating the tg-search demo from its bundled Caddy

It currently runs with its own Caddy (IP mode). To move it behind the shared
proxy:

```bash
# stop the standalone stack
cd /path/to/tg-discovery
docker compose -f docker-compose.yml -f docker-compose.ip.yml down

# bring it up on the shared network instead
docker compose -f docker-compose.yml -f docker-compose.shared.yml \
    --env-file .env.prod up -d --build --scale caddy=0
```

Then ensure `server-infra/Caddyfile` has the `tgsearch.example.com` block and
reload Caddy. Set the project's `.env.prod` `DOMAIN` / `CORS_ORIGINS` to the real
domain first.

## Notes

- Only the shared Caddy binds 80/443. Project Caddies are never started on this
  server (`--scale caddy=0`), avoiding port conflicts.
- Each project's api/meili stay on that project's private network — only its web
  container is exposed to the proxy network.
- Container names must be unique across the server (hence `tgsearch-web`, etc.).
