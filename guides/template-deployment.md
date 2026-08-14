# Deployment Guide

This guide walks you through deploying the stack on your VPS.

By the end you'll have:
- Traefik reverse proxy with automatic SSL
- Tinyauth + Pocket ID login for your apps
- Streaming, media, monitoring, and productivity apps running

If you haven't set up your VPS yet, follow the [VPS Setup Guide](./vps-setup.md) first.

---

## Requirements

- VPS with Ubuntu 24.04 and Docker installed
- Domain name pointing to your VPS IP
- Ports 80 and 443 open
- About 20-30 minutes

---

## Step 1: Connect to your VPS

```bash
ssh -i /path/to/your-key ubuntu@YOUR_SERVER_IP
```

All commands below run on the VPS.

---

## Step 2: Clone the template

```bash
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/MarshmellowXD/docker-compose-template.git docker
sudo chown -R $(id -u):$(id -g) /opt/docker
cd /opt/docker
```

> The GitHub repo is currently private. If you're using the Forgejo copy, use that URL instead.

---

## Step 3: Fill in `.env`

Open the main config file:

```bash
nano .env
```

Set these first:

```env
TZ=America/New_York
DOMAIN=yourdomain.com
LETSENCRYPT_EMAIL=you@yourdomain.com
CLOUDFLARE_API_TOKEN=your_cloudflare_token
```

Get a Cloudflare token:
1. Log into [Cloudflare](https://dash.cloudflare.com).
2. Go to **My Profile → API Tokens → Create Token**.
3. Use the **Edit zone DNS** template and select your domain.

> Not using Cloudflare? Leave the token blank and create DNS records manually.

### Secrets

Generate two random secrets:

```bash
openssl rand -base64 32
openssl rand -base64 32
```

Paste them into `.env`:

```env
POCKET_ID_ENCRYPTION_KEY=<first output>
TINYAUTH_SECRET=<second output>
```

### First user

Create a password hash:

```bash
htpasswd -nbB yourusername yourpassword
```

Output looks like:

```
yourusername:$2y$05$abcdefghijklmnopqrstuvwxyz123456789
```

Copy only the part after the colon and paste it like this:

```env
TINYAUTH_AUTH_USERS=yourusername:$$2y$$05$$abcdefghijklmnopqrstuvwxyz123456789
```

> Use `$$` instead of `$` because Docker Compose needs it.

### Choose services

Find this line:

```env
COMPOSE_PROFILES=required
```

Options:
- `required` — only Traefik, Tinyauth, Pocket ID, DDNS
- `required,streaming` — adds streaming apps
- `required,streaming,arr` — adds *arr stack and Jellyfin
- `all` — everything

Save and exit nano: `Ctrl+X`, `Y`, `Enter`.

---

## Step 4: Start core services

```bash
docker compose --profile required up -d
```

Wait 30 seconds, then check:

```bash
docker compose ps
```

You should see `traefik`, `tinyauth`, `pocket-id`, and `cloudflare-ddns` as `Up`.

### Check SSL

Open `https://traefik.YOURDOMAIN.com`. You should see a Tinyauth login page.

If you get a certificate error, wait 1-2 minutes and refresh.

---

## Step 5: Set up login

### Create a Pocket ID admin

1. Open `https://id.YOURDOMAIN.com`.
2. Click **Create Account**.
3. Pick a username and email.
4. Set up a passkey (Face ID, Windows Hello, or security key).

### Connect Tinyauth to Pocket ID

1. In Pocket ID, go to **Authorized clients → Create authorized client**.
2. Fill in:
   - **Name:** `Tinyauth`
   - **Callback URLs:** `https://login.YOURDOMAIN.com/api/oauth/callback/pocketid`
3. Click **Save**.
4. Copy the **Client ID** and **Client Secret**.

Open `.env` again:

```bash
nano .env
```

Paste:

```env
POCKET_ID_TINYAUTH_CLIENTID=<client id>
POCKET_ID_TINYAUTH_CLIENTSECRET=<client secret>
```

Restart Tinyauth:

```bash
docker compose restart tinyauth
```

### Test it

Open `https://login.YOURDOMAIN.com`, click **Log in with Pocket ID**, and verify.

---

## Step 6: Start the rest

If you picked `all` or other profiles, start them:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Everything should be `Up` or `Up (healthy)`.

---

## Step 7: Fill in app secrets

Only edit the `.env` files for apps you actually use.

### Gluetun (VPN)

```bash
nano apps/gluetun/.env
```

For ProtonVPN:

```env
WIREGUARD_PRIVATE_KEY=<your wireguard key>
```

Get your key from [ProtonVPN](https://account.proton.me/u/0/vpn/WireGuard).

Restart:

```bash
docker compose restart gluetun
```

### StremThru

```bash
nano apps/stremthru/.env
```

Generate and paste:

```bash
openssl rand -hex 32
```

```env
STREMTHRU_VAULT_SECRET=<output>
```

### AIOStreams

```bash
nano apps/aiostreams/.env
```

Generate and paste:

```bash
openssl rand -hex 32
```

```env
SECRET_KEY=<output>
```

Restart all apps after editing:

```bash
docker compose up -d
```

---

## Step 8: Set up streaming

### Configure AIOStreams

1. Open `https://aiostreams.YOURDOMAIN.com`.
2. Log in.
3. Add your debrid API key.
4. Add a TMDB API key from [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api).
5. Save and copy the manifest URL.

### Add to Stremio

1. Open Stremio.
2. Go to **Addons → Install from URL**.
3. Paste the AIOStreams manifest URL.
4. Click **Install**.

---

## Step 9: Monitoring (optional)

If you enabled `monitoring`:

- Grafana: `https://grafana.YOURDOMAIN.com` (default `admin` / `admin`)
- Homepage: `https://home.YOURDOMAIN.com`
- Uptime Kuma: `https://status.YOURDOMAIN.com`

---

## Troubleshooting

### Certificate error

Wait 2-3 minutes. Then check:

```bash
docker compose logs traefik | grep -i "error\|cert"
```

Make sure DNS points to your VPS.

### Container keeps restarting

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
- `TINYAUTH_AUTH_USERS` uses `$$`
- Pocket ID client ID and secret are correct

Then restart Tinyauth:

```bash
docker compose restart tinyauth
```

### Need to browse logs

Open Dozzle: `https://dozzle.YOURDOMAIN.com`

Or run:

```bash
docker compose logs -f <service-name>
```
