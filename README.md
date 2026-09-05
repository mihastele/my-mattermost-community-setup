# Self-hosted Mattermost

### Run for `compose.bind-mounts.yml` setup
```
mkdir -p \
  data/postgres \
  data/mattermost/config \
  data/mattermost/data \
  data/mattermost/logs \
  data/mattermost/plugins \
  data/mattermost/client/plugins \
  data/mattermost/bleve-indexes \
  data/caddy/data \
  data/caddy/config
```

#### Basic commands

```
docker compose pull
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs -f caddy mattermost
```


A production-oriented, single-host Mattermost deployment using Docker Compose, PostgreSQL, and Caddy.

This stack is designed for a private workspace or small trusted community. It provides:

- Mattermost Team Edition as the Slack-like collaboration server
- PostgreSQL with a persistent Docker volume
- Caddy as the only public-facing service, with automatic HTTPS
- Explicit HTTP and HTTPS port configuration
- 1 GiB Mattermost attachment limit
- SMTP configuration for invitations, verification-related mail, password resets, and notifications
- Private internal networking for PostgreSQL
- Container restart policies, health checks, resource limits, and Docker log rotation

> This is a reliable **single-server** deployment, not an HA cluster. Keep tested backups and a documented restore procedure.

## Architecture

```text
Internet
   |
   | TCP 80: HTTP / ACME validation / redirect
   | TCP 443: HTTPS (or a custom configured HTTPS port)
   v
Caddy reverse proxy
   |
   | private Docker network
   v
Mattermost :8065
   |
   | private Docker network; never exposed on the host
   v
PostgreSQL :5432
```

Only Caddy publishes host ports. Mattermost and PostgreSQL must not be exposed directly to the Internet.

## Prerequisites

- A Linux server with Docker Engine and Docker Compose v2
- A public DNS name, for example `chat.example.com`
- An A record pointing `chat.example.com` to this server's public IPv4 address
- Optional, correctly routed IPv6 and an AAAA record
- Firewall/router access to TCP ports 80 and 443
- An SMTP provider and a verified sender domain
- At least 4 GB RAM for a small instance; 8 GB is more comfortable
- Sufficient persistent storage for PostgreSQL data, attachments, logs, and backups

## Files

Keep these files together in one directory:

```text
mattermost/
├── compose.yml
├── .env
└── Caddyfile
```

The generated Caddy configuration may have been downloaded as `Caddyfile.conf`. Rename it before starting:

```bash
mv Caddyfile.conf Caddyfile
```

## Initial setup

### 1. Create the deployment directory

```bash
mkdir -p ~/services/mattermost
cd ~/services/mattermost
```

Place `compose.yml`, `.env.example`, and `Caddyfile` in this directory.

### 2. Create the real environment file

```bash
cp .env.example .env
chmod 600 .env
```

Edit it:

```bash
nano .env
```

At a minimum, replace all `REPLACE_*` values and set:

```dotenv
DOMAIN=chat.example.com
ACME_EMAIL=admin@example.com
HTTP_PORT=80
HTTPS_PORT=443
```

### 3. Generate the PostgreSQL password

Use a long, alphanumeric value. The deployment inserts this password in a PostgreSQL connection URL, so avoid characters that require URL escaping.

```bash
openssl rand -base64 64 | tr -dc 'A-Za-z0-9' | head -c 56; echo
```

Set the generated value as `POSTGRES_PASSWORD` in `.env`.

### 4. Pin Mattermost

Set `MATTERMOST_IMAGE_TAG` to one specific Mattermost Team Edition release that you have reviewed and tested. Do not use `latest` in a production deployment.

```dotenv
MATTERMOST_IMAGE_TAG=REPLACE_WITH_TESTED_MATTERMOST_VERSION
```

### 5. Configure SMTP

Use a transactional SMTP provider or an existing provider that permits application SMTP. Populate these fields:

```dotenv
SMTP_FROM_NAME=My Workspace
SMTP_FROM_ADDRESS=no-reply@example.com
SMTP_HOST=smtp.example-provider.com
SMTP_PORT=587
SMTP_USERNAME=your-smtp-username
SMTP_PASSWORD=your-smtp-password
SMTP_CONNECTION_SECURITY=STARTTLS
```

Port 587 with `STARTTLS` is typical. If the provider explicitly requires implicit TLS on port 465, use:

```dotenv
SMTP_PORT=465
SMTP_CONNECTION_SECURITY=TLS
```

Verify the sending domain with the SMTP provider and publish the SPF, DKIM, and DMARC DNS records it supplies. Without this, mail may be rejected or placed in spam.

### 6. Start the stack

Validate the resolved Compose configuration first:

```bash
docker compose config --quiet
```

Then pull and start the services:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Tail startup logs:

```bash
docker compose logs -f caddy mattermost
```

Open `https://chat.example.com` and create the first account. This account becomes the Mattermost system administrator.

## Ports and TLS

### Standard deployment

The recommended configuration is:

```dotenv
HTTP_PORT=80
HTTPS_PORT=443
```

Open TCP 80 and 443 in UFW, cloud firewall rules, and any home-router port-forwarding configuration.

Example UFW rules:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Caddy uses port 80 for normal ACME HTTP validation and redirects visitors to HTTPS. Certificate and renewal state are stored in the persistent `caddy_data` volume.

### Custom HTTPS port

To serve Mattermost on a non-standard port such as 8443:

```dotenv
HTTP_PORT=80
HTTPS_PORT=8443
```

Open TCP 80 and TCP 8443, and use:

```text
https://chat.example.com:8443
```

A standard port 443 is preferable for browser compatibility, mobile clients, corporate firewalls, and clean shared links.

## Registration and email

The stack is intentionally configured as invite-only:

```dotenv
MM_EMAILSETTINGS_ENABLESIGNUPWITHEMAIL=false
MM_TEAMSETTINGS_ENABLEOPENSERVER=false
```

This prevents strangers from creating accounts if they discover the URL. Invite trusted users from Mattermost after SMTP is configured.

Email remains required for invitations, verification-related mail, password resets, and notification delivery. Test SMTP from Mattermost System Console after signing in as administrator.

## File uploads

Mattermost allows attachments up to 1 GiB:

```dotenv
MAX_FILE_SIZE_BYTES=1073741824
```

The value is applied through `MM_FILESETTINGS_MAXFILESIZE`. Caddy has no stricter request-body limit in this configuration, and Mattermost's read/write timeouts are extended to 900 seconds for slower uploads.

A 1 GiB limit does not mean unlimited storage. Plan for attachment growth, backup duration, bandwidth, and restoration time. Lower the value if that is inappropriate for the server.

## Operations

### Container status

```bash
docker compose ps
docker compose logs --tail=200 mattermost
docker compose logs --tail=200 caddy
docker compose logs --tail=200 postgres
```

### Restart services

```bash
docker compose restart
docker compose restart mattermost
```

### Validate a configuration change

```bash
docker compose config --quiet
docker compose up -d
```

### Upgrade safely

1. Create and verify backups.
2. Read the Mattermost release notes and upgrade notes.
3. Change `MATTERMOST_IMAGE_TAG` in `.env` to one tested version.
4. Pull the image and recreate only the application service.
5. Verify logs, login, uploads, search, and SMTP delivery.

```bash
docker compose pull mattermost
docker compose up -d mattermost
docker compose logs -f mattermost
```

Do not upgrade PostgreSQL major versions by merely changing `postgres:16-alpine` to a new major tag. A PostgreSQL major upgrade requires a planned migration process.

## Backup and restore

You need both a PostgreSQL backup and the Mattermost persistent volumes. A database-only backup does not include uploaded attachments or all application configuration.

### PostgreSQL logical backup

Create a local backup directory:

```bash
mkdir -p backups
chmod 700 backups
```

Create a compressed database backup:

```bash
docker compose exec -T postgres pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" | gzip > "backups/mattermost-db-$(date +%F-%H%M%S).sql.gz"
```

The command needs variables available in your current shell. Load the safe values from `.env` without echoing secrets in shell history, or replace the two variable references with the database/user names.

### Docker volume backup

Back up the Mattermost and Caddy volumes as well. The relevant volumes are:

- `mattermost_postgres_data`
- `mattermost_mattermost_config`
- `mattermost_mattermost_data`
- `mattermost_mattermost_logs`
- `mattermost_mattermost_plugins`
- `mattermost_mattermost_client_plugins`
- `mattermost_mattermost_bleve`
- `mattermost_caddy_data`
- `mattermost_caddy_config`

Exact Docker volume names may differ if you change the Compose project name. List them with:

```bash
docker volume ls | grep mattermost
```

Use an encrypted backup target and test restoration into an isolated server or an alternate Compose project. A backup is only useful once restore has been tested.

> Never run `docker compose down -v` on this project. The `-v` option destroys named volumes, including database data, attachments, configuration, and TLS certificate state.

## Troubleshooting

### Caddy cannot obtain a certificate

Check all of the following:

- `DOMAIN` resolves publicly to this server.
- TCP 80 is reachable from the Internet.
- No other reverse proxy or web server uses ports 80 or 443.
- The DNS record is not proxied in a way incompatible with certificate validation.
- The Caddy logs identify the hostname you expect.

```bash
docker compose logs --tail=200 caddy
```

### Mattermost shows 502 Bad Gateway

Mattermost may still be starting, especially after its first database migration. Check:

```bash
docker compose ps
docker compose logs --tail=200 postgres
docker compose logs --tail=200 mattermost
```

Do not expose Mattermost port 8065 directly as a workaround; fix the application/database startup issue instead.

### Mail does not arrive

Check SMTP settings in `.env`, then inspect Mattermost logs:

```bash
docker compose logs --tail=300 mattermost
```

Confirm the SMTP provider accepts the sender address, the password is correct, and SPF/DKIM/DMARC records have been published. Also check recipient spam/junk folders during initial testing.

### An upload fails

Verify the configured attachment cap:

```bash
docker compose exec mattermost env | grep MM_FILESETTINGS_MAXFILESIZE
```

Then verify available Docker-host disk capacity:

```bash
df -h
docker system df
```

Large uploads can fail because of inadequate disk space, reverse-proxy timeouts, a provider/proxy body-size limit in front of Caddy, or insufficient upstream bandwidth.

## Scaling note

This deployment stores files locally in the Mattermost data volume and is correct for one application node. Before deploying multiple Mattermost replicas, move attachment storage to S3-compatible object storage and use a PostgreSQL deployment appropriate for HA. Do not share a local Docker volume between multiple hosts as an improvised HA solution.
