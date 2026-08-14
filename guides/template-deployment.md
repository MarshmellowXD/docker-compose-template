# Deployment Guide

This guide will help you deploy the Docker Compose stack on your VPS. By the end, you'll have:

- A reverse proxy with automatic SSL (Traefik)
- Single sign-on for your apps (Tinyauth + Pocket ID)
- Media streaming apps (AIOStreams, StremThru, etc.)
- Monitoring dashboards (Grafana, Homepage, etc.)

If anything goes wrong, check the [Troubleshooting](#troubleshooting) section at the bottom.

---

## What you need

Before you start:

- A VPS running Ubuntu 24.04 (the [VPS setup guide](./vps-setup.md) shows how)
- A domain name pointing to your VPS
- Ports 80 and 443 open in your firewall
- About 20-30 minutes

---

## Step 1: Connect to your VPS

Open a terminal on your computer and SSH into your server:

```bash
ssh -i /path/to/your-key ubuntu@YOUR_SERVER_IP
```

Once you're logged in, run all the commands below while connected to the VPS.

---

## Step 2: Clone the repo

Create the folder and download the template:

```bash
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/MarshmellowXD/docker-compose-template.git docker
sudo chown -R $(id -u):$(id -g) /opt/docker
cd /opt/docker
```

> The GitHub repo is currently private. If you are using the Forgejo copy instead, replace the GitHub URL with your Forgejo URL.

---

## Step 3: Fill in your settings

All the main settings live in one file: `.env`.

Open it:

```bash
nano .env
```

Fill in these required values:

| Setting | What to enter | Example |
|---------|--------------|---------|
| `TZ` | Your timezone | `America/New_York` |
| `DOMAIN` | Your domain | `example.com` |
| `LETSENCRYPT_EMAIL` | Your email for SSL cert alerts | `you@example.com` |
| `CLOUDFLARE_API_TOKEN` | Your Cloudflare API token | see below |

### Get a Cloudflare API token

1. Log into [Cloudflare](https://dash.cloudflare.com).
2. Click your profile picture → **My Profile** → **API Tokens**.
3. Click **Create Token**.
4. Use the **Edit zone DNS** template.
5. Select your domain and create the token.
6. Copy the token and paste it into `.env`.

> If you don't use Cloudflare, leave the token blank and create your DNS records manually.

### Generate secrets

Run these commands to create random secrets:

```bash
openssl rand -base64 32
openssl rand -base64 32
```

Paste the outputs into:

```env
POCKET_ID_ENCRYPTION_KEY=<first output>
TINYAUTH_SECRET=<second output>
```

### Create your first login user

Run this command, replacing `yourusername` and `yourpassword`:

```bash
htpasswd -nbB yourusername yourpassword
```

You will see something like:

```
yourusername:$2y$05$abcdefghijklmnopqrstuvwxyz123456789
```

Copy the part **after** the colon (starting with `$2y$05$`) and paste it into `.env` like this:

```env
TINYAUTH_AUTH_USERS=yourusername:$$2y$$05$$abcdefghijklmnopqrstuvwxyz123456789
```

> The `$$` is required because Docker Compose uses `$` for variables.

### Pick which services to run

Find this line in `.env`:

```env
COMPOSE_PROFILES=required
```

You can leave it as `required` for now and add more later, or change it to `all` to start everything:

```env
COMPOSE_PROFILES=all
```

Save the file and exit nano: press `Ctrl+X`, then `Y`, then `Enter`.

---

## Step 4: Start the core services

Start Traefik, Tinyauth, Pocket ID, and Cloudflare DDNS:

```bash
docker compose --profile required up -d
```

Wait about 30 seconds, then check that the containers are running:

```bash
docker compose ps
```

You should see `traefik`, `tinyauth`, `pocket-id`, and `cloudflare-ddns` all showing `Up`.

### Check SSL is working

Open this in your browser:

```
https://traefik.YOURDOMAIN.com
```

You should see a login page. If you get a certificate error, wait a minute and refresh.

---

## Step 5: Set up login

You need to connect Tinyauth to Pocket ID so you can log in with one click.

### Create a Pocket ID admin account

1. Open `https://id.YOURDOMAIN.com` in your browser.
2. Click **Create Account**.
3. Enter a username and email.
4. Set up a passkey (Face ID, Windows Hello, or a security key).
5. You are now logged into the Pocket ID dashboard.

### Create an OIDC client for Tinyauth

1. In Pocket ID, click **Authorized clients** in the sidebar.
2. Click **Create authorized client**.
3. Fill in:
   - **Name:** `Tinyauth`
   - **Callback URLs:** `https://login.YOURDOMAIN.com/api/oauth/callback/pocketid`
4. Click **Save**.

You will see a **Client ID** and **Client Secret**. Keep this page open.

### Add the credentials to `.env`

Open `.env` again:

```bash
nano .env
```

Paste the values:

```env
POCKET_ID_TINYAUTH_CLIENTID=<your client id>
POCKET_ID_TINYAUTH_CLIENTSECRET=<your client secret>
```

Save and restart Tinyauth:

```bash
docker compose restart tinyauth
```

### Test login

1. Open `https://login.YOURDOMAIN.com`.
2. Click **Log in with Pocket ID**.
3. Verify with your passkey.
4. You should be logged in.

---

## Step 6: Start the rest of your services

If you set `COMPOSE_PROFILES=all` earlier, start everything:

```bash
docker compose up -d
```

If you left it as `required`, edit `.env` first and change it to the profiles you want:

```env
COMPOSE_PROFILES=required,streaming,monitoring
```

Then run:

```bash
docker compose up -d
```

Check the status:

```bash
docker compose ps
```

All services should show `Up` or `Up (healthy)`.

---

## Step 7: Fill in per-app settings

Some apps need their own `.env` files. Edit only the ones you are using.

### Gluetun (VPN)

Open `apps/gluetun/.env`:

```bash
nano apps/gluetun/.env
```

If you use ProtonVPN:

1. Log into [ProtonVPN](https://account.proton.me/u/0/vpn/WireGuard).
2. Copy your WireGuard private key.
3. Paste it into the file:

```env
WIREGUARD_PRIVATE_KEY=<your key>
```

Then restart Gluetun:

```bash
docker compose restart gluetun
```

### StremThru

Open `apps/stremthru/.env` and generate a secret:

```bash
openssl rand -hex 32
```

Paste it:

```env
STREMTHRU_VAULT_SECRET=<your secret>
```

### AIOStreams

Open `apps/aiostreams/.env` and generate a secret:

```bash
openssl rand -hex 32
```

Paste it:

```env
SECRET_KEY=<your secret>
```

Restart the apps after editing:

```bash
docker compose up -d
```

---

## Step 8: Configure streaming

### AIOStreams

1. Open `https://aiostreams.YOURDOMAIN.com`.
2. Log in with Tinyauth → Pocket ID.
3. Add your debrid service API key under **Debrid Services**.
4. Add a TMDB API key under **Poster Configuration**.
   - Get one at [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api).
5. Save the config.
6. Copy the manifest URL it gives you.

### Stremio

1. Open Stremio on your computer or phone.
2. Go to **Addons** (puzzle piece icon).
3. Click **Install from URL**.
4. Paste the AIOStreams manifest URL.
5. Click **Install**.

---

## Step 9: Monitoring (optional)

If you enabled the `monitoring` profile, these are available:

- Grafana: `https://grafana.YOURDOMAIN.com` (default login: `admin` / `admin`)
- Homepage: `https://home.YOURDOMAIN.com`
- Uptime Kuma: `https://status.YOURDOMAIN.com`

---

## Troubleshooting

### I get a certificate error

Wait 2-3 minutes for Let's Encrypt. Then check Traefik logs:

```bash
docker compose logs traefik | grep -i "error\|cert"
```

Make sure your DNS records point to your VPS.

### A container keeps restarting

Check its logs:

```bash
docker compose logs <service-name>
```

For example:

```bash
docker compose logs tinyauth
```

### Tinyauth login doesn't work

Double-check these in `.env`:

- `TINYAUTH_SECRET` is filled in
- `TINYAUTH_AUTH_USERS` has the right format with `$$`
- `POCKET_ID_TINYAUTH_CLIENTID` and `POCKET_ID_TINYAUTH_CLIENTSECRET` are correct

Then restart Tinyauth:

```bash
docker compose restart tinyauth
```

### I forgot my Pocket ID passkey

Pocket ID stores its data in `data/pocket-id/pocket-id.db`. You can reset it by stopping Pocket ID, deleting that file, and starting over:

```bash
docker compose stop pocket-id
rm data/pocket-id/pocket-id.db
docker compose up -d pocket-id
```

Then recreate your admin account.

### Still stuck?

Check the live container logs:

```bash
docker compose logs -f <service-name>
```

Or open Dozzle at `https://dozzle.YOURDOMAIN.com` to browse logs in your browser.
