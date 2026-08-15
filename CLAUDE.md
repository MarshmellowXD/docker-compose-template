# CLAUDE.md

This file provides guidance to Claude Code and other AI assistants when working with this repository.

## What this repo is

A public Docker Compose template for self-hosting ~40 services on a single VPS, fronted by **Traefik** (reverse proxy + Let's Encrypt) and protected by **Tinyauth** (forward-auth) + **Pocket ID** (OIDC provider).

End users clone this onto a VPS at `/opt/docker`, fill in `.env`, and run `docker compose up -d`. There is no build step for the template itself — the repo *is* the artifact. Some apps (SeedStream, Forgejo runner) may build local images when deployed.

## Architecture

**Root `compose.yaml` is just an `include:` list** of `apps/<name>/compose.yaml` files. Each app is self-contained under `apps/<name>/`. Adding or removing a service is done by editing the `include:` list and/or toggling profiles.

**Profiles gate every service.** Every service declares `profiles: [<name>, all]`, and core gateway services (`traefik`, `tinyauth`, `pocket-id`, `cloudflare-ddns`, `socket-proxy`, `watchtower`) also declare `required`. Nothing starts unless its profile is listed in `COMPOSE_PROFILES` (set in root `.env`) or passed via `--profile`. The `all` profile turns everything on; `required` is the minimum bootstrap.

**Config flows through `.env`, not YAML.**

- Root `.env` holds shared config: `DOMAIN`, `DOCKER_DATA_DIR`, `DOCKER_DIR`, `PUID`/`PGID`, `TZ`, `COMPOSE_PROFILES`, secrets, and one `<APP>_HOSTNAME=<sub>.${DOMAIN}` line per service.
- Per-app secrets/options live in `apps/<name>/.env` and are pulled in via `env_file: - .env` inside that app's compose file.
- `.env` files are tracked in this repo and contain placeholder values. Users fill in real values after cloning.
- Hostnames use `${X_HOSTNAME?}` (the `?` makes the variable required); preserve this pattern when adding services so misconfiguration fails fast.

**Traefik wiring is by label.** Each web-exposed service declares labels such as:

```yaml
traefik.enable=true
traefik.http.routers.<svc>.rule=Host(`${<SVC>_HOSTNAME?}`)
traefik.http.routers.<svc>.entrypoints=websecure
traefik.http.routers.<svc>.tls.certresolver=letsencrypt
traefik.http.routers.<svc>.middlewares=tinyauth@docker   # if login required
traefik.http.services.<svc>.loadbalancer.server.port=<container-port>
```

Traefik reads these via the Docker provider. The `letsencrypt` resolver and `tinyauth@docker` middleware are defined in `apps/traefik/compose.yaml` and `apps/tinyauth/compose.yaml`.

**Auth is Tinyauth + Pocket ID.** Tinyauth is the forward-auth middleware. Pocket ID is the OIDC provider. Most web services use `tinyauth@docker`. Do not add it to `outline` (uses Pocket ID directly) or Forgejo public routes (`/api/`, `.git`).

**Network.** All services share one Docker bridge network named `aio_network`. Service-to-service references use container names (e.g., `http://tinyauth:3000`).

**Volume conventions.**

- App data → `${DOCKER_DATA_DIR}/<app>:/...` (absolute, from root `.env`).
- App-shipped config files → relative paths like `./config:/config`, which resolve relative to that app's compose file (e.g., `apps/traefik/config/`).

**Docker socket access.** Read-only consumers use `tcp://socket-proxy:2375`. Write consumers (Watchtower, Forgejo runner) mount `/var/run/docker.sock` directly.

## Common commands

```bash
# First boot — bring up only core services
docker compose --profile required up -d

# Use the profiles set in .env
docker compose up -d

# Operate on one service stack
docker compose --profile <name> up -d
docker compose --profile <name> restart
docker compose --profile <name> logs -f

# Validate the merged config without starting anything
docker compose config
```

Note: `--profile` must come before the subcommand. `COMPOSE_PROFILES` in `.env` is read for bare `docker compose up -d`, but `--profile` flags override it for that invocation.

## Adding a new service

1. Create `apps/<name>/compose.yaml` with `container_name`, `restart: unless-stopped`, the right `image`, `profiles: [<name>, all]`, Traefik labels (including `tinyauth@docker` unless there's a protocol reason not to), and volumes under `${DOCKER_DATA_DIR}/<name>`.
2. Add `- apps/<name>/compose.yaml` to the `include:` list in root `compose.yaml` (keep alphabetical).
3. Add `<NAME>_HOSTNAME=<sub>.${DOMAIN}` to root `.env` near the other hostname entries.
4. If the service needs its own secrets/config, create `apps/<name>/.env` and reference it with `env_file: - .env` in the compose file.
5. If the service needs read-only Docker socket access, point it at `tcp://socket-proxy:2375` instead of mounting `/var/run/docker.sock`.
6. Add a card to `apps/homepage/config/services.yaml` if you want it on the dashboard.
7. If the service needs path-based bypasses or group restrictions with Tinyauth, update `apps/tinyauth/compose.yaml` environment variables (`TINYAUTH_APPS_<name>_PATH_ALLOW`, etc.).

## Gotchas

- Relative bind-mount paths (`./config:/config`) resolve relative to the *including* compose file's directory — i.e. `apps/<name>/`, not the repo root. Absolute paths via `${DOCKER_DATA_DIR}` avoid this ambiguity and are preferred for data volumes.
- `${VAR?}` syntax aborts compose with a clear error if `VAR` is unset — preserve it when copying patterns.
- `apps/cloudflare-ddns/compose.yaml` only works for domains using Cloudflare nameservers.
- ARM64 only — verify images support `linux/arm64`.
- qBittorrent is pinned to `5.0.3`. Do not upgrade without retesting.
- SeedStream is a separate Git repo under `/opt/docker/seedstream` on the live VPS. It is gitignored in `docker-compose-vps`.

## Repo sync

The live VPS stack lives in `docker-compose-vps` on Forgejo. The public template lives in `docker-compose-template` on Forgejo and GitHub. Changes to `/opt/docker` auto-sync to Forgejo; changes to `/opt/docker-compose-template` auto-sync to Forgejo and then to GitHub every 5 minutes.
