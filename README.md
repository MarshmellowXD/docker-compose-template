# Docker Compose Template

A Docker Compose stack for self-hosting **40+ services** behind **Traefik**, with single sign-on via **Tinyauth + Pocket ID**.

## What you get

- **Reverse proxy & SSL**: Traefik with automatic Let's Encrypt certificates
- **Authentication**: Tinyauth (forward-auth) + Pocket ID (OIDC provider)
- **Streaming**: AIOStreams, AIOMetadata, AIOManager, SeedStream, StremThru, Comet
- **Media backend**: qBittorrent, Transmission, Prowlarr, Radarr, Sonarr, Bazarr, Jellyfin, Overseerr, and more
- **Monitoring**: Grafana, Prometheus, Alertmanager, Uptime Kuma, Dozzle, Homepage, Beszel
- **Productivity**: Forgejo, Outline
- **Utilities**: ntfy, Warp, Technitium DNS

## Quick start

```bash
# 1. Clone the repo
git clone https://github.com/MarshmellowXD/docker-compose-template.git /opt/docker
cd /opt/docker

# 2. Copy and fill in environment files
cp .env.example .env
for f in apps/*/.env.example; do cp "$f" "${f%.example}"; done
nano .env

# 3. Deploy the core services first
docker compose --profile required up -d

# 4. Add more services as needed, e.g.
docker compose --profile streaming --profile arr up -d
```

See the [Template Deployment Guide](guides/template-deployment.md) for the full walkthrough.

## Profiles

Services are grouped by profiles so you can enable only what you need:

| Profile | Services |
|---------|----------|
| `required` | Traefik, Tinyauth, Pocket ID, Cloudflare DDNS, Watchtower, Socket Proxy |
| `streaming` | AIOStreams, AIOMetadata, AIOManager, SeedStream, StremThru, Comet |
| `arr` | *arr stack, download clients, Jellyfin, Overseerr |
| `media` | Immich |
| `security` | Vaultwarden |
| `monitoring` | Grafana, Prometheus, Alertmanager, Uptime Kuma, Dozzle, Homepage, Beszel |
| `productivity` | Forgejo, Forgejo Runner, Outline |
| `utils` | ntfy, Warp, Technitium |
| `all` | Everything |

Set `COMPOSE_PROFILES` in `.env` or pass `--profile <name>` to `docker compose` commands.

## Security notes

- `.env` files are gitignored. Never commit them.
- Set strong passwords and API tokens before starting services.
- Review `apps/technitium/compose.yaml` before exposing DNS ports.
- The default network is a single flat `aio_network`. Segment further if your threat model requires it.

## Credits

The stack and guides started as templates from [Viren's Docker Compose template](https://github.com/Viren070/docker-compose-template).

## License

MIT
