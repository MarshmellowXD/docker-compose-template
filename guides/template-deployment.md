# Deployment Guide

This guide walks you through deploying the stack on your VPS.

By the end you'll have:

- Traefik reverse proxy with automatic SSL
- Tinyauth + Pocket ID login for your apps
- Streaming, media, monitoring, and productivity apps running

If you haven't set up your VPS yet, follow the [VPS Setup Guide](./vps-setup.md) first.

---

## Before you start

Make sure you have:

- A VPS with Ubuntu 24.04 and Docker installed
- A domain name you own
- DNS records pointing to your VPS IP
- Ports 80 and 443 open on your VPS firewall
- About 20-30 minutes

> **Warning:** This stack binds ports 80 and 443. Don't run it on a server that already uses those ports for another reverse proxy.

---

## Step 1: Connect to your VPS

Run this on your local computer:

```bash
ssh -i /path/to/your-key ubuntu@YOUR_SERVER_IP
```

All remaining commands run on the VPS.

---

## Step 2: Create the folder and clone the template

Create `/opt/docker`:

```bash
sudo mkdir -p /opt
```

Move into `/opt`:

```bash
cd /opt
```

Clone the template:

```bash
sudo git clone https://github.com/MarshmellowXD/docker-compose-template.git docker
```

Make yourself the owner:

```bash
sudo chown -R $(id -u):$(id -g) /opt/docker
```

Move into the project folder:

```bash
cd /opt/docker
```

---

## Step 3: Fill in `.env`

Open the main config file:

```bash
nano .env
```

### Basic settings

Set these near the top:

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

> **Cloudflare token:** Log into Cloudflare, go to **My Profile → API Tokens → Create Token**, and use the **Edit zone DNS** template.
>
> Not using Cloudflare? Leave the token blank and create DNS A records manually.

### Generate secrets

Run this twice:

```bash
openssl rand -base64 32
```

Copy each output into `.env`:

```env
POCKET_ID_ENCRYPTION_KEY=<first output>
```

```env
TINYAUTH_SECRET=<second output>
```

### Create your first user

Generate a password hash:

```bash
htpasswd -nbB yourusername yourpassword
```

Output looks like:

```
yourusername:$2y$05$abcdefghijklmnopqrstuvwxyz123456789
```

Paste only the part after the colon, with `$$` instead of `$`:

```env
TINYAUTH_AUTH_USERS=yourusername:$$2y$$05$$abcdefghijklmnopqrstuvwxyz123456789
```

> **Why `$$`?** Docker Compose treats `$` specially, so you must double it.

### Choose which apps to run

Find this line:

```env
COMPOSE_PROFILES="required"
```

Pick one:

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

For your first deploy, use `required` only.

Save and exit nano:

```text
Ctrl+X, Y, Enter
```

---

## Step 4: Configure Traefik

Open Traefik's environment file:

```bash
nano apps/traefik/.env
```

For testing, use Let's Encrypt staging to avoid rate limits:

```env
LE_CA_SERVER=https://acme-staging-v02.api.letsencrypt.org/directory
```

For production, use the real server:

```env
LE_CA_SERVER=https://acme-v02.api.letsencrypt.org/directory
```

Save and exit:

```text
Ctrl+X, Y, Enter
```

---

## Step 5: Start the core services

Pull and start the required services:

```bash
docker compose --profile required up -d
```

Wait 30 seconds:

```bash
sleep 30
```

Check that everything is up:

```bash
docker compose ps
```

You should see these services as `Up` or `Up (healthy)`:

- `traefik`
- `tinyauth`
- `pocket-id`
- `socket-proxy`
- `cloudflare-ddns`
- `warp`

> **Docker Hub rate limits:** If pulls fail with `toomanyrequests`, wait 1-2 minutes and run the `up -d` command again.

---

## Step 6: Check SSL

Open this in your browser:

```text
https://traefik.YOURDOMAIN.com
```

You should see a Tinyauth login page.

> **Staging certificates show a browser warning.** That's expected. Click past it or switch to production LE_CA_SERVER once everything works.

If you get a connection error:

1. Make sure DNS points to your VPS IP.
2. Check Traefik logs:

```bash
docker compose logs traefik | grep -i "error\|cert"
```

---

## Step 7: Set up Pocket ID

### Create a Pocket ID admin

Open:

```text
https://id.YOURDOMAIN.com
```

Click **Create Account**.

Pick a username and email.

Set up a passkey (Face ID, Windows Hello, or security key).

### Connect Tinyauth to Pocket ID

In Pocket ID, go to **Authorized clients → Create authorized client**.

Fill in:

- **Name:** `Tinyauth`
- **Callback URLs:** `https://login.YOURDOMAIN.com/api/oauth/callback/pocketid`

Click **Save**.

Copy the **Client ID** and **Client Secret**.

Open `.env` again:

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

Save and exit:

```text
Ctrl+X, Y, Enter
```

Restart Tinyauth:

```bash
docker compose restart tinyauth
```

Wait 10 seconds:

```bash
sleep 10
```

### Test login

Open:

```text
https://login.YOURDOMAIN.com
```

Click **Log in with Pocket ID** and verify it works.

---

## Step 8: Start the rest of your apps

If you picked `required,streaming`, `required,arr`, or `all`, start them now:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Everything should be `Up` or `Up (healthy)`.

If something is restarting, check its logs:

```bash
docker compose logs <service-name>
```

Example:

```bash
docker compose logs aiostreams
```

---

## Step 9: Fill in app secrets

Only edit the `.env` files for apps you actually use.

### Gluetun (VPN)

Open:

```bash
nano apps/gluetun/.env
```

Set your WireGuard private key:

```env
WIREGUARD_PRIVATE_KEY=<your wireguard key>
```

Get your key from [ProtonVPN](https://account.proton.me/u/0/vpn/WireGuard).

Restart Gluetun:

```bash
docker compose restart gluetun
```

### StremThru

Open:

```bash
nano apps/stremthru/.env
```

Generate a secret:

```bash
openssl rand -hex 32
```

Paste:

```env
STREMTHRU_VAULT_SECRET=<output>
```

Restart StremThru:

```bash
docker compose restart stremthru
```

### AIOStreams

Open:

```bash
nano apps/aiostreams/.env
```

Generate a secret:

```bash
openssl rand -hex 32
```

Paste:

```env
SECRET_KEY=<output>
```

Restart AIOStreams:

```bash
docker compose restart aiostreams
```

---

## Step 10: Set up streaming

Stremio addons expose public `manifest.json` and `/stream/` endpoints so the Stremio client can talk to them. Tinyauth is configured to allow these paths automatically, so you don't need to disable auth for your addons.

### Configure AIOStreams

Open:

```text
https://aiostreams.YOURDOMAIN.com
```

Log in with Pocket ID.

Add your debrid API key.

Add a TMDB API key from [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api).

Save and copy the manifest URL.

### Add to Stremio

Open Stremio.

Go to **Addons → Install from URL**.

Paste the AIOStreams manifest URL.

Click **Install**.

---

## Step 11: Monitoring (optional)

If you enabled the `monitoring` profile:

- Grafana: `https://grafana.YOURDOMAIN.com` (default `admin` / `admin`)
- Homepage: `https://home.YOURDOMAIN.com`
- Uptime Kuma: `https://status.YOURDOMAIN.com`

---

## Troubleshooting

### Certificate error

Wait 2-3 minutes.

Check Traefik logs:

```bash
docker compose logs traefik | grep -i "error\|cert"
```

Make sure DNS points to your VPS.

### Container keeps restarting

Check logs:

```bash
docker compose logs <service-name>
```

Example:

```bash
docker compose logs tinyauth
```

### Can't log in

Check in `.env`:

- `TINYAUTH_SECRET` is set
- `TINYAUTH_AUTH_USERS` uses `$$
- Pocket ID client ID and secret are correct

Then restart Tinyauth:

```bash
docker compose restart tinyauth
```

### Need to browse logs

Open Dozzle:

```text
https://dozzle.YOURDOMAIN.com
```

Or follow logs in the terminal:

```bash
docker compose logs -f <service-name>
```

### Docker Hub rate limit

If image pulls fail with `toomanyrequests`, log into Docker Hub or wait a few minutes and retry:

```bash
docker compose up -d
```
