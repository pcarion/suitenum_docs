# Deploy Docs on Hetzner Cloud with your own domain

This guide explains how to run Docs on a single Hetzner Cloud VM using Docker Compose, with HTTPS and your own domain names.

> [!IMPORTANT]
> This follows the project's Compose deployment path, which maintainers describe as experimental compared with Kubernetes.

## 1) What you will deploy

You will run:
- Docs (frontend + backend + y-provider)
- PostgreSQL
- Redis
- Optionally Keycloak (OIDC identity provider)
- Optionally MinIO (S3-compatible object storage)
- `nginx-proxy` + `acme-companion` for TLS certificates

Recommended hostnames:
- `docs.example.com` → Docs
- `id.example.com` → OIDC provider (Keycloak)
- `storage.example.com` → object storage (MinIO)

## 2) Create your Hetzner Cloud server

1. Create a project and an SSH key in Hetzner Cloud.
2. Create one Ubuntu VM (for example CPX31+ for small production usage).
3. Attach a volume if you want persistent data outside the root disk.
4. In Hetzner Cloud firewall, open:
   - TCP `22` (SSH)
   - TCP `80` (Let's Encrypt ACME + HTTP)
   - TCP `443` (HTTPS)

## 3) Point your domain to the server

At your DNS provider, add `A` records to the VM public IPv4:
- `docs.example.com`
- `id.example.com`
- `storage.example.com`

Wait for DNS propagation before requesting certificates.

## 4) Install Docker and Compose on the VM

SSH to the server, then install a current Docker Engine with Compose plugin (see Docker official instructions for your distro).

Quick validation:

```bash
docker --version
docker compose version
```

## 5) Prepare directories and fetch official examples

```bash
mkdir -p ~/lasuite/{docs,keycloak,minio,nginx-proxy}

# Docs stack files
cd ~/lasuite/docs
mkdir -p env.d
curl -o compose.yaml https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/docs/examples/compose/compose.yaml
curl -o env.d/common https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/env.d/production.dist/common
curl -o env.d/backend https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/env.d/production.dist/backend
curl -o env.d/yprovider https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/env.d/production.dist/yprovider
curl -o env.d/postgresql https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/env.d/production.dist/postgresql
curl -o default.conf.template https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/docker/files/production/etc/nginx/conf.d/default.conf.template

# nginx proxy
cd ~/lasuite/nginx-proxy
curl -o compose.yaml https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/docs/examples/compose/nginx-proxy/compose.yaml

# Keycloak (optional if you already have OIDC)
cd ~/lasuite/keycloak
mkdir -p env.d
curl -o compose.yaml https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/docs/examples/compose/keycloak/compose.yaml
curl -o env.d/kc_postgresql https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/env.d/production.dist/kc_postgresql
curl -o env.d/keycloak https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/env.d/production.dist/keycloak

# MinIO (optional if you already have S3 storage)
cd ~/lasuite/minio
curl -o compose.yaml https://raw.githubusercontent.com/suitenumerique/docs/refs/heads/main/docs/examples/compose/minio/compose.yaml
```

## 6) Create shared Docker network for proxying

```bash
docker network create proxy-tier
```

Then follow each compose example to uncomment/add `proxy-tier` network and set `VIRTUAL_HOST`, `VIRTUAL_PORT`, and `LETSENCRYPT_HOST` for services that must be exposed.

## 7) Configure Docs for your domain

Edit `~/lasuite/docs/env.d/common`:

```env
DOCS_HOST=docs.example.com
KEYCLOAK_HOST=id.example.com
S3_HOST=storage.example.com
BUCKET_NAME=docs-media-storage
REALM_NAME=docs
```

Edit `~/lasuite/docs/env.d/backend`:
- Generate strong secrets:
  - `DJANGO_SECRET_KEY`
  - `OIDC_RP_CLIENT_SECRET`
  - `AWS_S3_ACCESS_KEY_ID` / `AWS_S3_SECRET_ACCESS_KEY`
- Keep OIDC endpoints aligned to your IdP host and realm.
- Keep `OIDC_REDIRECT_ALLOWED_HOSTS` including `https://docs.example.com`.
- Configure SMTP values (`DJANGO_EMAIL_*`) for invitations.

## 8) Configure Keycloak (if self-hosted)

In `~/lasuite/keycloak/env.d/keycloak` set at least:

```env
KC_HOSTNAME=https://id.example.com
KC_BOOTSTRAP_ADMIN_PASSWORD=<strong-password>
```

In Keycloak admin UI:
1. Create realm `docs`.
2. Create client (example `docs`) with client authentication enabled.
3. Set redirect URI to `https://docs.example.com/*`.
4. Set web origin to `https://docs.example.com`.
5. Copy client ID/secret into Docs `env.d/backend`.

## 9) Bring services up

Start in this order:

```bash
cd ~/lasuite/nginx-proxy && docker compose up -d
cd ~/lasuite/keycloak && docker compose up -d     # if used
cd ~/lasuite/minio && docker compose up -d        # if used
cd ~/lasuite/docs && docker compose up -d
```

Initialize database and admin user:

```bash
cd ~/lasuite/docs
docker compose run --rm backend python manage.py migrate
docker compose run --rm backend python manage.py createsuperuser --email admin@example.com --password '<strong-password>'
```

## 10) How authentication works in Docs

Docs uses OpenID Connect (OIDC) for login.

High-level flow:
1. User opens `https://docs.example.com`.
2. Docs redirects to your OIDC provider authorization endpoint.
3. After login at the IdP, user returns to Docs callback URL.
4. Docs backend exchanges authorization code for tokens at token endpoint.
5. Docs fetches user claims from userinfo endpoint and maps them to local user fields.
6. A Django session is created and reused for authenticated API access.

Important behavior in this repository:
- OIDC endpoint URLs and client credentials are configured through env vars in `env.d/backend`.
- The custom backend computes `full_name` and `short_name` from userinfo claims.
- Existing users are matched by OIDC `sub` (and optionally email fallback, depending on settings).
- Media requests (`/media/*`) are authorized by nginx `auth_request` against `/api/v1.0/documents/media-auth/` before proxying to S3/MinIO.
- Collaboration WebSocket traffic is proxied via `/collaboration/ws/` to y-provider.

## 11) Operational notes for Hetzner

- Use Hetzner backups/snapshots and external backup for PostgreSQL + object storage.
- Pin image tags instead of `latest` before production upgrades.
- Run upgrades with: pull images, restart containers, run migrations.
- Start with generous RAM sizing if self-hosting both Keycloak and Docs.

## 12) If you already have external OIDC and S3

You can skip local Keycloak/MinIO entirely:
- Keep only Docs + PostgreSQL + Redis + reverse proxy.
- Set OIDC variables in Docs to your external IdP endpoints/client.
- Set S3 variables in Docs to your external object store.
