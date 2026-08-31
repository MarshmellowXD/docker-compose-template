# Deployment Guide

This guide walks you through deploying the stack on your VPS.

By the end you'll have Traefik reverse proxy with automatic SSL, Tinyauth + Pocket ID login, and your chosen apps running.

If you haven't set up your VPS yet, follow the [VPS Setup Guide](./vps-setup.md) first.

---

## Requirements

- VPS with Ubuntu 24.04 and Docker installed
- Domain name pointing to your VPS IP
- Ports 80 and 443 open
- About 20-30 minutes

> This stack binds ports 80 and 443. Don't run it on a server that already uses those ports.

---

## Step 1: Connect to your VPS

```bash
ssh -i /path/to/your-key ubuntu@YOUR_SERVER_IP
```

All remaining commands run on the VPS.

---

## Step 2: Clone the template

```bash
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/MarshmellowXD/docker-compose-template.git docker
sudo chown -R $(id -u):$(id -g) /opt/docker
cd /opt/docker
```

---

## Step 3: Fill in `.env`

```bash
nano .env
```

Set the basics:

```env
TZ=America/New_York
```

```env
DOMAIN=yourdomain.com
```

```env
LETSENCRYPT_EMAIL=you@yourdomain.com
```

```env
CLOUDFLARE_API_TOKEN=your_cloudflare_token
```

Get a Cloudflare token at **My Profile → API Tokens → Create Token** using the **Edit zone DNS** template. Not using Cloudflare? Leave the token blank and create DNS A records manually.

### Secrets

Generate two secrets:

```bash
openssl rand -base64 32
```

```bash
openssl rand -base64 32
```

Paste them into `.env`:

```env
POCKET_ID_ENCRYPTION_KEY=<first output>
```

```env
TINYAUTH_SECRET=<second output>
```

### First user

Create a local user with the Tinyauth CLI:

```bash
docker run -i -t --rm ghcr.io/tinyauthapp/tinyauth:v5 user create --interactive
```

Enter your username and password. When asked for the format, pick **format for docker**.

The output looks like:

```text
user:$$2a$$10$UdLYoJ5lgPsC0RKqYH/jMua7zIn0g9kPqWmhYayJYLaZQ/FTmH2/u
```

Paste it into `.env`:

```env
TINYAUTH_AUTH_USERS=user:$$2a$$10$UdLYoJ5lgPsC0RKqYH/jMua7zIn0g9kPqWmhYayJYLaZQ/FTmH2/u
```

### Choose services

Set one of these in `.env`:

```env
COMPOSE_PROFILES="required"
```

```env
COMPOSE_PROFILES="required,streaming"
```

```env
COMPOSE_PROFILES="required,streaming,arr"
```

```env
COMPOSE_PROFILES="all"
```

- `required` — Traefik, Tinyauth, Pocket ID, Cloudflare DDNS, socket-proxy, WARP
- `streaming` — AIOStreams, AIOMetadata, AIOManager, StremThru
- `arr` — *arr stack, Jellyfin, download clients
- `all` — everything

For your first deploy, use `required`.

Save and exit nano with `Ctrl+X`, `Y`, `Enter`.

---

## Step 4: Configure Traefik

```bash
nano apps/traefik/.env
```

For testing, use staging to avoid Let's Encrypt rate limits:

```env
LE_CA_SERVER=https://acme-staging-v02.api.letsencrypt.org/directory
```

For production:

```env
LE_CA_SERVER=https://acme-v02.api.letsencrypt.org/directory
```

Save and exit.

---

## Step 5: Start core services

```bash
docker compose --profile required up -d
```

Wait 30 seconds, then check status:

```bash
sleep 30
docker compose ps
```

Core services should show `Up` or `Up (healthy)`:

- `traefik`
- `tinyauth`
- `pocket-id`
- `socket-proxy`
- `cloudflare-ddns`
- `warp`

If pulls fail with `toomanyrequests`, wait 1-2 minutes and run `docker compose --profile required up -d` again.

---

## Step 6: Check SSL

Open `https://traefik.YOURDOMAIN.com` in your browser. You should see a Tinyauth login page.

Staging certificates will show a browser warning. That's expected. Switch to production `LE_CA_SERVER` once everything works.

If you get a connection error, check DNS and Traefik logs:

```bash
docker compose logs traefik | grep -i "error\|cert"
```

---

## Step 7: Set up Pocket ID

Open `https://id.YOURDOMAIN.com`:

1. Click **Create Account**
2. Pick a username and email
3. Set up a passkey

Then connect Tinyauth:

1. In Pocket ID, go to **Authorized clients → Create authorized client**
2. Set **Name** to `Tinyauth`
3. Set **Callback URLs** to `https://login.YOURDOMAIN.com/api/oauth/callback/pocketid`
4. Save and copy the **Client ID** and **Client Secret**

Open `.env`:

```bash
nano .env
```

Paste:

```env
POCKET_ID_TINYAUTH_CLIENTID=<client id>
```

```env
POCKET_ID_TINYAUTH_CLIENTSECRET=<client secret>
```

Save and exit, then restart Tinyauth:

```bash
docker compose restart tinyauth
```

Test by opening `https://login.YOURDOMAIN.com` and clicking **Log in with Pocket ID**.

---

## Step 8: Start the rest

If you picked `required,streaming`, `required,arr`, or `all`:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

If something is restarting, check its logs:

```bash
docker compose logs <service-name>
```

---

## Step 9: Fill in app secrets

Only edit the `.env` files for apps you use.

### Gluetun

```bash
nano apps/gluetun/.env
```

```env
WIREGUARD_PRIVATE_KEY=<your wireguard key>
```

Get your key from [ProtonVPN](https://account.proton.me/u/0/vpn/WireGuard).

```bash
docker compose restart gluetun
```

### StremThru

```bash
nano apps/stremthru/.env
```

```bash
openssl rand -hex 32
```

```env
STREMTHRU_VAULT_SECRET=<output>
```

```bash
docker compose restart stremthru
```

### AIOStreams

```bash
nano apps/aiostreams/.env
```

```bash
openssl rand -hex 32
```

```env
SECRET_KEY=<output>
```

```bash
docker compose restart aiostreams
```

---

## Step 10: Set up streaming

Open `https://aiostreams.YOURDOMAIN.com`, log in with Pocket ID, add your debrid API key, and add a TMDB API key from [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api). Save and copy the manifest URL.

In Stremio, go to **Addons → Install from URL**, paste the manifest URL, and click **Install**.

---

## Step 11: Monitoring (optional)

If you enabled `monitoring`:

- Grafana: `https://grafana.YOURDOMAIN.com` (default `admin` / `admin`)
- Homepage: `https://home.YOURDOMAIN.com`
- Uptime Kuma: `https://status.YOURDOMAIN.com`

---

## Troubleshooting

### Certificate error

Wait 2-3 minutes, then check Traefik logs:

```bash
docker compose logs traefik | grep -i "error\|cert"
```

Make sure DNS points to your VPS.

### Container keeps restarting

```bash
docker compose logs <service-name>
```

### Can't log in

Check `.env`:

- `TINYAUTH_SECRET` is set
- `TINYAUTH_AUTH_USERS` uses `$$`
- Pocket ID client ID and secret are correct

Then restart Tinyauth:

```bash
docker compose restart tinyauth
```

### Browse logs

Open `https://dozzle.YOURDOMAIN.com` or run:

```bash
docker compose logs -f <service-name>
```
