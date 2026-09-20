# CutRDFlexo production deployment

This repository contains the small amount of deployment configuration required
to run the current CutRDFlexo production application.

The application image is built and published separately to the private Gitea
OCI registry:

```text
git.roselabs.mx/roselabs/cutrdflexo:<version>
```

The deployment is intentionally version-pinned. Production should use an
immutable release tag such as `v0.5.0`, never `latest`.

## Current architecture

```text
Internet
   |
   v
Nginx / TLS on host
   |
   +--> dashboard.crdflexo.mx --+
   |                            |
   +--> portal.crdflexo.mx -----+--> 127.0.0.1:5600
                                      |
                                      v
                              CutRDFlexo container
                              - Go API
                              - Staff frontend
                              - Portal frontend
                                      |
                        +-------------+-------------+
                        |                           |
                        v                           v
                  Valkey container          External services
                                           - DigitalOcean PostgreSQL
                                           - S3 / DigitalOcean Spaces
                                           - SMTP provider
```

The Go application performs host-based frontend routing. Nginx must preserve the
original `Host` header.

Valkey is deliberately private to the Compose network and is not published on a
host port.

## Repository layout

```text
.
├── compose.yaml
├── .env.example
├── .gitignore
└── README.md
```

The real `.env` is production-secret material and must never be committed.

## Prerequisites

The production host needs:

- Docker Engine;
- Docker Compose v2 (`docker compose`);
- network access to the DigitalOcean PostgreSQL cluster;
- network access to the configured S3-compatible object store;
- network access to the SMTP provider;
- access to `git.roselabs.mx`;
- Nginx (or another reverse proxy) on the host.

## 1. Configure the deployment

Clone this repository on the production host, then:

```bash
cp .env.example .env
chmod 600 .env
```

Fill every `CHANGE_ME`, database endpoint, storage endpoint, origin and release
version before starting the stack.

Generate a Valkey password with, for example:

```bash
openssl rand -hex 32
```

The production database URL should use the connection information supplied by
DigitalOcean and TLS should remain enabled.

### Important deployment variables

```dotenv
CUTRDFLEXO_VERSION=v0.5.0
APP_PORT=5600

DATABASE_URL=...
VALKEY_PASSWORD=...

STAFF_APP_ORIGIN=https://dashboard.crdflexo.mx
PORTAL_APP_ORIGIN=https://portal.crdflexo.mx
HOMEPAGE_APP_ORIGIN=https://www.crdflexo.mx
```

The Compose file owns these container-specific settings and normally they should
not be overridden:

```text
APP_ENV=production
HTTP_ADDR=:5600
VALKEY_ADDR=valkey:6379
VALKEY_TLS=false
SERVE_FRONTEND=true
STAFF_FRONTEND_DIR=/app/frontend/staff/dist
PORTAL_FRONTEND_DIR=/app/frontend/portal/dist
```

## 2. Log in to the private registry

Use a Gitea account/token that has permission to pull the private package:

```bash
docker login git.roselabs.mx
```

Do not put the token in this repository.

Verify access:

```bash
docker pull "git.roselabs.mx/roselabs/cutrdflexo:${CUTRDFLEXO_VERSION}"
```

If the shell has not loaded `.env`, substitute the version explicitly.

## 3. Apply database migrations

The database must be migrated to the schema expected by the release before
starting that release.

At the moment migrations remain owned by the main CutRDFlexo application
repository. Run the application's pinned migration command from an authorized
workstation/source checkout against the production `DATABASE_URL`:

```bash
make migrate-status
make migrate-up
```

Do not automatically run downward migrations during rollback.

A future improvement should make migrations a first-class command inside the
release image so this deployment repository can perform them without requiring
a source checkout.

## 4. First deployment

Pull the exact version configured in `.env`:

```bash
docker compose pull
```

Start the stack:

```bash
docker compose up -d
```

Inspect it:

```bash
docker compose ps
docker compose logs -f app
```

The application is intentionally bound only to loopback on the host:

```text
127.0.0.1:${APP_PORT}
```

Check readiness locally:

```bash
curl -fsS "http://127.0.0.1:${APP_PORT}/ready"
```

A successful readiness response means PostgreSQL, Valkey and the configured S3
bucket are reachable by the application.

## 5. Nginx

Both the staff and portal hostnames may proxy to the same Go process. The
application chooses the correct frontend from the HTTP `Host` header.

The important proxy settings are:

```nginx
location / {
    proxy_pass http://127.0.0.1:5600;

    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Use this behind the appropriate TLS-enabled server blocks for:

```text
dashboard.crdflexo.mx
portal.crdflexo.mx
```

Do not replace `Host $host` with the upstream hostname: host-based frontend
routing depends on the original browser hostname.

The public institutional homepage may remain separately hosted. Its origin is
still configured through `HOMEPAGE_APP_ORIGIN` for the public API/CORS policy.

## Updating CutRDFlexo

First publish and validate the new release from the main application repository.

On production, update only the version in `.env`:

```dotenv
CUTRDFLEXO_VERSION=v0.6.0
```

Apply any migrations required by that release, then:

```bash
docker compose pull app
docker compose up -d app
```

Watch startup:

```bash
docker compose logs -f app
```

Verify readiness:

```bash
curl -fsS "http://127.0.0.1:${APP_PORT}/ready"
```

Then verify both public hostnames through Nginx.

Because the image tag is explicit, `docker compose up -d` is deterministic and
does not silently move production to a newer application version.

## Rollback

If the previous application version remains compatible with the current
database schema, set the previous image tag:

```dotenv
CUTRDFLEXO_VERSION=v0.5.0
```

Then:

```bash
docker compose pull app
docker compose up -d app
```

Do not automatically migrate the database downward. Database rollback needs to
be considered separately for each migration.

## Useful commands

Status:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f app
docker compose logs -f valkey
```

Restart only the application:

```bash
docker compose restart app
```

Pull configured images:

```bash
docker compose pull
```

Stop containers while preserving Valkey data:

```bash
docker compose down
```

Start again:

```bash
docker compose up -d
```

Do **not** casually use:

```bash
docker compose down -v
```

because `-v` deletes the persistent Valkey volume.

## Secrets

The `.env` file contains production credentials.

Recommended permissions:

```bash
chmod 600 .env
```

Never commit it.

Also be aware that:

```bash
docker compose config
```

may render interpolated configuration containing secrets. Do not paste its
output into public tickets, logs or chat without reviewing/redacting it.

## Valkey durability

Valkey currently stores sessions and related transient application state.

It uses AOF persistence and a named Docker volume:

```text
valkey_data
```

The database and S3 storage remain the authoritative durable business stores.

## Preparing for per-tenant deployments

This Compose layout intentionally avoids fixed `container_name` values and fixed
network/volume names. Docker Compose therefore namespaces resources using the
Compose project name.

A future per-tenant deployment can reuse this same repository/configuration:

```text
Tenant A
COMPOSE_PROJECT_NAME=cutrdflexo-tenant-a
APP_PORT=5610
DATABASE_URL=postgres://...tenant_a...
S3_BUCKET=cutrdflexo-tenant-a
STAFF_APP_ORIGIN=https://dashboard.tenant-a.example
PORTAL_APP_ORIGIN=https://portal.tenant-a.example

Tenant B
COMPOSE_PROJECT_NAME=cutrdflexo-tenant-b
APP_PORT=5620
DATABASE_URL=postgres://...tenant_b...
S3_BUCKET=cutrdflexo-tenant-b
STAFF_APP_ORIGIN=https://dashboard.tenant-b.example
PORTAL_APP_ORIGIN=https://portal.tenant-b.example
```

Each project automatically receives its own:

- application container;
- Valkey container;
- private Compose network;
- Valkey volume.

Nginx can route each tenant's hostnames to its corresponding loopback port.

This does not itself make the application multi-tenant; it simply keeps the
deployment topology compatible with a later isolated-per-tenant model.

## Recommended next improvements

Once this first production deployment is stable, the next useful pieces are:

1. add a migration subcommand to the CutRDFlexo release image;
2. add an in-image HTTP healthcheck dependency so Compose can mark `app` healthy;
3. add a deployment script that performs pull -> migrate -> restart -> `/ready`
   verification;
4. optionally automate deployment from the release process;
5. add backup/restore documentation for PostgreSQL and object storage;
6. decide whether Valkey should remain per-host or move to managed infrastructure
   when tenant isolation/scale requires it.
