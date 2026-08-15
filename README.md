# Docker Compose Template

A Docker Compose stack for self-hosting various applications on a VPS, using Traefik as a reverse proxy and Tinyauth + Pocket ID for authentication.

This is a template, so feel free to edit it by removing or adding services as you see fit.

For setting up the VPS itself, see the [VPS Setup Guide](guides/vps-setup.md).

This is the general deployment guide:

## Prerequisites

- A VPS with Docker installed.
- Port 80 and 443 open.
- A domain with DNS records pointing to the VPS IP for each subdomain you want to use.
- Moving your nameservers to Cloudflare is preferred as it enables Cloudflare DDNS, included in this template, to automatically create and maintain your DNS records.

## Quick start

1. Ensure Docker is installed per [https://get.docker.com](https://get.docker.com).
2. Clone this repository and cd into it:

   ```bash
   cd /opt
   git clone https://github.com/MarshmellowXD/docker-compose-template.git docker
   cd docker
   ```

3. Use a text editor to open the `.env` file. If you are using nano, run this command:

   ```bash
   nano .env
   ```

4. After you fill in the initial values in the root `.env`, it is recommended to use the `required` profile only and run the following command to start Traefik, Tinyauth, and Pocket ID:

   ```bash
   docker compose --profile required up -d
   ```

5. Now you can either start adding other profiles to the `COMPOSE_PROFILES` environment variable or use the `all` profile and fill in any other `.env` files for the services you want to use.

6. Finally, run this command to start all the services according to your profiles in `.env`:

   ```bash
   docker compose up -d
   ```

   - Ensure you defined the `COMPOSE_PROFILES` environment variable in the `.env` file, otherwise none of the services will start.
   - You can also use the `--profile` flag, e.g. `docker compose --profile streaming --profile arr up -d`.
   - Ensure port 443 and port 80 are open.
   - If you have not set up Cloudflare DDNS in the `.env` by providing your token and using the `cloudflare-ddns` profile, you will need to manually create A records for each service you want to use using the subdomain in the `.env` file.

See the [Template Deployment Guide](guides/template-deployment.md) for the full walkthrough.

## About

A flexible Docker Compose template to host various apps using Traefik and Tinyauth + Pocket ID, focusing on media streaming.

## Credits

The stack and guides started as templates from [Viren's Docker Compose template](https://github.com/Viren070/docker-compose-template).

## License

MIT
