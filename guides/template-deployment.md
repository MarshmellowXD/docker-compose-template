# Template Deployment Guide

This guide walks you through deploying the Docker Compose stack on your VPS.

By the end, you'll have:
- **Reverse proxy** (Traefik) with automatic SSL certificates
- **Authentication** (Tinyauth + Pocket ID) — single sign-on for all your services
- **Media streaming** (AIOStreams, StremThru, Comet) — watch your content through Stremio
- **Monitoring** (Grafana, Prometheus, Uptime Kuma, Dozzle, Homepage) — see what your server is doing

---

## Prerequisites

- A **VPS** with Ubuntu 24.04 and Docker installed (see the [VPS Setup Guide](./vps-setup.md))
- A **domain** with DNS pointing to your VPS (A record + wildcard `*.yourdomain.com`)
- Ports **80** and **443** open in your firewall (we opened them in the VPS guide)
- About **20 minutes**

---

## Step 1: Clone the Repository

SSH into your VPS and run:

```bash
# Create the directory and clone
sudo mkdir -p /opt
cd /opt
# If you are using the public GitHub mirror, replace the URL below.
git clone https://forgejo.theallblue.net/MarshmellowXD/docker-compose-template.git docker
sudo chown -R $(id -u):$(id -g) /opt/docker
cd /opt/docker
```

---

Each app lives in its own folder under `apps/<name>/` with its own `compose.yaml` and `.env`. The root `compose.yaml` just pulls them all in with includes. Services are gated by profiles — you pick which ones to run with `COMPOSE_PROFILES`.

## Step 2: Copy and Configure Environment Files

The repo ships `.env.example` files so real secrets are never committed. Copy them to `.env` and fill in your own values:

```bash
cd /opt/docker
cp .env.example .env
for f in apps/*/.env.example; do cp "$f" "${f%.example}"; done
```

Open the root `.env`:

```bash
nano .env
```

> Prefer a GUI? Use **VS Code Remote SSH** or **XPipe** to browse and edit files on your server without the terminal.

### Required settings (set these first):

| Variable | What to put | Example |
|----------|------------|---------|
| `TZ` | Your timezone | `America/New_York` |
| `DOMAIN` | Your domain | `example.com` |
| `LETSENCRYPT_EMAIL` | Your email (for SSL cert notifications) | `you@example.com` |
| `CLOUDFLARE_API_TOKEN` | Your Cloudflare API token (see below) | `abc123...` |

**Getting a Cloudflare API token:**

1. Log into [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Go to **My Profile** → **API Tokens**
3. Click **Create Token**
4. Use the **Edit zone DNS** template
5. Select your domain
6. Create the token and copy it

> **Not using Cloudflare?** Leave `CLOUDFLARE_API_TOKEN` as-is and create DNS records manually. The DDNS service won't work, but everything else will.

### Generate secrets:

These must be **random and unique**. Run these commands in your terminal (not on the VPS — or in the VPS):

```bash
# Pocket ID encryption key
openssl rand -base64 32

# Tinyauth session secret
openssl rand -base64 32
```

Copy the output of each command and paste them into the corresponding lines in `.env`:

```env
POCKET_ID_ENCRYPTION_KEY=<paste-base64-output-here>
TINYAUTH_SECRET=<paste-base64-output-here>
```

### Create a Tinyauth user:

You need at least one user to log in with:

```bash
# Generate a bcrypt hash for your password
# Replace 'yourpassword' with your actual password
htpasswd -nbB yourusername yourpassword
```

You'll get output like:
```
yourusername:$2y$05$abcdefghijklmnopqrstuvwxyz123456789
```

Copy the **entire second part** (after the username and colon) — including the `$2y$05$...` — and set:

```bash
# In .env, escape the $ signs
TINYAUTH_AUTH_USERS=yourusername:$$2y$$05$$abcdefghijklmnopqrstuvwxyz123456789
```

> Note: the `$$` is because Docker Compose interprets `$` as variable expansion. You're replacing `$` with `$$`.

### Leave the rest as-is for now:

- `COMPOSE_PROFILES` — set to `required` for now (we'll add more later)
- Hostname variables (e.g. `TRAEFIK_HOSTNAME`) — already use `${DOMAIN}` which resolves to your domain
- Per-app hostnames — you can customize subdomains later

Save and close the file (`Ctrl+X`, `Y`, `Enter` in nano).

---

## Step 3: Start Core Services

Deploy the required profile — Traefik, Tinyauth, Pocket ID, socket-proxy, and Cloudflare DDNS:

```bash
docker compose --profile required up -d
```

This will pull images and start containers. Wait about 30 seconds, then check:

```bash
docker compose ps
```

You should see these containers all showing `Up`:

| Name | Purpose |
|------|---------|
| `traefik` | Reverse proxy, SSL certificates |
| `tinyauth` | Authentication middleware |
| `pocket-id` | OIDC provider (SSO) |
| `socket-proxy` | Secure Docker socket proxy |
| `cloudflare-ddns` | Auto-updates DNS records |

### Verify SSL certificates

Visit your Traefik dashboard:
```
https://traefik.YOURDOMAIN
```

If you see a Tinyauth login page, SSL is working. If you get a certificate error, wait a minute (Let's Encrypt needs time) and refresh.

> **Troubleshooting:** If certs don't appear after 5 minutes, check:
> ```bash
> docker compose logs traefik | grep "error\|cert"
> ```

---

## Step 4: Set Up Pocket ID + Tinyauth

This is the most important setup step — it connects Pocket ID (SSO) with Tinyauth (auth middleware).

### 4a. Create a Pocket ID admin account

1. Open `https://id.YOURDOMAIN` (replace `YOURDOMAIN` with your domain)
2. Click **Create Account**
3. Enter:
   - **Username:** `admin` (or whatever you like)
   - **Email:** your email
   - Click **Submit**
4. Follow the prompts to set up passkey authentication (Face ID, Windows Hello, or a security key)
5. Once logged in, you'll see the Pocket ID dashboard

### 4b. Create an OIDC client for Tinyauth

1. In Pocket ID, go to **Authorized clients** (sidebar)
2. Click **Create authorized client**
3. Fill in:
   - **Name:** `Tinyauth`
   - **Callback URLs:** `https://login.YOURDOMAIN/api/oauth/callback/pocketid`
   - Leave the rest as defaults
4. Click **Save**

You'll see a **Client ID** and **Client Secret** — **keep this page open**.

### 4c. Add the client credentials to .env

Back in your terminal:

```bash
nano .env
```

Find these lines and paste the values from Pocket ID:

```env
POCKET_ID_TINYAUTH_CLIENTID=<paste-the-client-id>
POCKET_ID_TINYAUTH_CLIENTSECRET=<paste-the-client-secret>
```

Save and restart Tinyauth:

```bash
docker compose restart tinyauth
```

### 4d. Verify authentication works

1. Open `https://login.YOURDOMAIN`
2. You should see a Tinyauth page with a **Log in with Pocket ID** button
3. Click it — it should redirect to Pocket ID
4. Verify with your passkey
5. You're logged in!

> **💡 First-time Tinyauth user:** The fallback user you created in `.env` also works — log in with `yourusername` and the password you set at the Tinyauth login page.

---

## Step 5: Add Services

Now that auth is working, let's add the streaming services. Services are grouped into profiles — `required` for core, `streaming` for streaming media apps, `monitoring` for dashboards, and `all` for everything.

Edit `.env` and change `COMPOSE_PROFILES` from `required` to whatever you want:

```env
COMPOSE_PROFILES=required,streaming,arr,monitoring
```

Or just run everything:

```env
COMPOSE_PROFILES=all
```

You can also launch specific services on the fly without editing `.env`:

```bash
docker compose --profile streaming --profile arr up -d
```

### Configure per-app secrets

Each app has its own `.env` file in `apps/<name>/.env`. You need to configure the ones you enabled.

**Gluetun (VPN):** `apps/gluetun/.env`

If you use ProtonVPN:
1. Log into [ProtonVPN](https://account.proton.me/u/0/vpn/WireGuard)
2. Copy your **WireGuard Private Key**
3. Paste it into `apps/gluetun/.env`:

```env
WIREGUARD_PRIVATE_KEY=<your-protonvpn-wireguard-key>
```

> Not using ProtonVPN? See [Gluetun's provider docs](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) for other providers.
>
> **Alternative:** Prefer Cloudflare WARP instead of a VPN? Swap gluetun for warp — lighter, faster, no VPN config. See `apps/warp/compose.yaml`. Point services at `socks5://warp:1080` instead of `socks5://gluetun:1080`.

**StremThru:** `apps/stremthru/.env`

Generate a vault secret:

```bash
openssl rand -hex 32
```

Paste it:

```env
STREMTHRU_VAULT_SECRET=<paste-hex-output>
```

**AIOStreams:** `apps/aiostreams/.env`

Generate a secret key:

```bash
openssl rand -hex 32
```

Paste it:

```env
SECRET_KEY=<paste-hex-output>
```

### 5c. Deploy everything

```bash
docker compose up -d
```

Wait 1-2 minutes for all containers to start. Check status:

```bash
docker compose ps
```

All services should show `Up (healthy)`.

### 5d. Verify the services

Visit these URLs — they should all redirect to the Tinyauth login, then let you in after authentication:

| Service | URL |
|---------|-----|
| AIOStreams | `https://aiostreams.YOURDOMAIN` |
| StremThru | `https://stremthru.YOURDOMAIN` |
| AIOManager | `https://aiomanager.YOURDOMAIN` |
| AIOMetadata | `https://aiometadata.YOURDOMAIN` |
| Comet | `https://comet.YOURDOMAIN` |
| ntfy | `https://ntfy.YOURDOMAIN` |
| Dozzle | `https://dozzle.YOURDOMAIN` |
| Homepage | `https://home.YOURDOMAIN` |

---

## Step 6: Configure Services for Streaming

### AIOStreams

1. Open `https://aiostreams.YOURDOMAIN`
2. Log in (Tinyauth → Pocket ID)
3. You'll see the AIOStreams configuration page
4. Under **Debrid Services**, add your API keys:
   - **TorBox:**
5. Under **Poster Configuration**, add a TMDB API key:
   - Get one at [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api)
6. Configure your stream filters (quality, language, etc.)
7. Click **Save**

After saving, you'll get a **manifest URL** like:
```
https://aiostreams.YOURDOMAIN/stremio/<your-uuid>/manifest.json
```

Copy this — you'll paste it into Stremio.

### StremThru

1. Open `https://stremthru.YOURDOMAIN`
2. You'll see the StremThru dashboard
3. Configure your debrid service under **Stores**
4. The tunnel through Gluetun (VPN) is already configured — traffic routes through the VPN automatically

### Install in Stremio

1. Open Stremio on your computer
2. Go to **Addons** (puzzle piece icon)
3. Click **Install from URL**
4. Paste the manifest URL from AIOStreams
5. Click **Install**

---

## Step 7: Set Up Monitoring (Optional)

The monitoring stack (Grafana, Prometheus, Uptime Kuma, Homepage) requires minimal config:

1. Enable `monitoring` (and any other groups you want) in `COMPOSE_PROFILES` or set it to `all`
2. Deploy: `docker compose up -d`
3. Open `https://grafana.YOURDOMAIN` — default login is `admin` / `admin`
4. Open `https://home.YOURDOMAIN` — your dashboard
5. Open `https://status.YOURDOMAIN` — Uptime Kuma for service monitoring


