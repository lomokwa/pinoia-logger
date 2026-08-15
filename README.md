# pinoia-logger

Shared logging stack for the homelab: **Loki** (log storage), **Promtail**
(log collector), and **Grafana** (UI). Runs independently of
`selton-mello-bot` and `mc-manager` — it discovers and tails their container
logs via the Docker socket, so it doesn't need to be redeployed when either
app is.

## How it works

- **Loki** stores logs, indexed only by *labels* (not full text), so it's
  cheap to run. Config: `loki/loki-config.yml` (filesystem storage, 30-day
  retention).
- **Promtail** auto-discovers every running container on the same Docker
  host via the Docker API and tails its stdout/stderr. It relabels using
  Docker Compose's own labels so each app gets a clean `job` label (e.g.
  `job="selton-mello-bot"`, `job="mc-manager"`) — this is what lets you pick
  "which app's logs do I want to see" in Grafana. Config:
  `promtail/promtail-config.yml`.
- **Grafana** is the UI. The Loki datasource and a starter "Service Logs"
  dashboard (with a `$job` picker) are auto-provisioned on first boot — see
  `grafana/provisioning/`.

## Running locally

```bash
cp .env.example .env   # set GRAFANA_ADMIN_PASSWORD; leave CLOUDFLARE_TUNNEL_TOKEN blank locally
docker compose up -d loki promtail grafana   # skip cloudflared locally, it needs a real tunnel token
```

Open http://localhost:3000 (`admin` / whatever you set), and check
**Dashboards → Service Logs** or **Explore** (Loki datasource).

## Deploying to the homelab

1. Clone this repo to the homelab (same pattern as the other repos, e.g.
   `/home/lomokwa/homelab/pinoia-logger`).
2. Create `.env` from `.env.example`:
   - `GRAFANA_ADMIN_PASSWORD` — a real password (Grafana still forces a reset
     on first login, but don't leave it as the literal default even briefly).
   - `GRAFANA_ROOT_URL` — the public hostname you'll configure in step 3
     (e.g. `https://logs.lomokwa.com`), so Grafana generates correct links.
   - `CLOUDFLARE_TUNNEL_TOKEN` — see step 3.
   - `DOCKER_DATA_ROOT` — only if `docker info --format '{{.DockerRootDir}}'`
     on this host isn't `/var/lib/docker` (e.g. snap-installed Docker on
     Ubuntu uses `/var/snap/docker/common/var-lib-docker`). Promtail needs
     this to find containers' log files — get it wrong and `docker compose
     up` fails with `error while creating mount source path
     '/var/lib/docker/containers': ... read-only file system`.
3. **Expose Grafana via Cloudflare Tunnel** (no inbound port opened on the
   homelab). This is a tunnel dedicated to this stack — separate from the
   `cloudflared access ssh` setup the other repos' deploy workflows use:
   - Cloudflare Zero Trust dashboard → **Networks → Tunnels → Create a
     tunnel** → choose **Docker** as the connector → copy the token from the
     `--token ...` flag shown there → put it in `.env` as
     `CLOUDFLARE_TUNNEL_TOKEN`.
   - Still in that tunnel's config, add a **Public Hostname**: hostname
     `logs.lomokwa.com` (must be a domain already added as a zone in the same
     Cloudflare account as your tunnel — Cloudflare auto-creates the DNS
     record when you add this) → Service `http://grafana:3000` (the
     `cloudflared` container reaches Grafana over the compose network by
     service name, no port publishing needed).
   - Optional but recommended: put a **Cloudflare Access** policy in front of
     that hostname (Zero Trust → Access → Applications → Add an
     application → Self-hosted), e.g. restricting it to your and your devs'
     email addresses. This gates reaching the Grafana login page at all,
     on top of Grafana's own per-user login — two layers instead of one.
4. `docker compose up -d`.
5. Log in at your chosen hostname as `admin`, set a new password when
   prompted, then create accounts for the other devs (see below).

## Adding accounts for other devs

Self-signup and org-creation are disabled
(`GF_USERS_ALLOW_SIGN_UP`/`GF_USERS_ALLOW_ORG_CREATE=false` in
`docker-compose.yml`), so only an existing admin can create accounts:

1. Log in as `admin`.
2. **Administration → Users and access → Users → New user.**
3. Create one account per dev with their own username/password (they can
   change their password after first login, same as the admin account).
4. Set their **Org role**:
   - **Viewer** — can browse dashboards and Explore, can't change
     datasources/dashboards/settings. Right default for "just want to read
     logs."
   - **Editor** — can also build/edit dashboards. Give this to whoever wants
     to add panels beyond the default "Service Logs" dashboard.
   - Avoid handing out **Admin** unless they need to manage other users too.

If you'd rather not share passwords out-of-band, Grafana also supports SMTP
invites (Administration → Users → Invite) — not configured here since it
needs a mail provider; ask if you want that added.
