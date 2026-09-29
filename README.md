# CutRDFlexo deployment

Docker Compose deployment for one dedicated installation: application and private
Valkey, external PostgreSQL and private S3-compatible storage, with host Nginx/TLS.
The optional staging override captures email in Mailpit. It does not provision
PostgreSQL, buckets, DNS, certificates or backups. The homepage remains separately
hosted; the application image includes the staff and portal frontends.

## Current images and compatibility

The application release configuration currently targets **Linux amd64 only**:

| Tags | `CAPABILITY_PRICING_SUGGESTIONS` |
| --- | --- |
| `vX.Y.Z-ps`, `latest-ps` | `true` |
| `vX.Y.Z-nops`, `latest-nops` | `false` |

Repositories: `git.roselabs.mx/roselabs/cutrdflexo` and
`lax.vultrcr.com/roselabsmx/cutrdflexo`. Confirm the chosen version was actually
published. A failed multi-registry release can leave partial publication. Pulling
from Gitea does not require the Vultr copy to have succeeded.

`CUTRDFLEXO_IMAGE` now contains the **complete image reference**, preferably
`repository@sha256:VERIFIED_DIGEST`. The old separate `CUTRDFLEXO_VERSION` variable
is no longer used. Version tags can be overwritten; only digest pinning identifies
immutable content. The former unsuffixed and `-standard` examples are obsolete.
The runtime capability must match the compiled frontend or startup fails.

Build and publish from the application repository, not this deployment checkout:

```bash
make docker-builder-setup
make release-snapshot
make release BUMP=patch
```

The publisher needs Docker authentication to both configured registries and Gitea
API authentication. Do not create a new version merely to retry an interrupted
publication; recover the existing tag with the application release tooling.

## 1. Prepare an installation

Use a separate checkout/directory for each deployment. For staging:

```bash
cp .env.staging.example .env
cp installation.example.json installation.json
chmod 600 .env
chmod 644 installation.json
```

For production, copy `.env.example` instead. Fill every placeholder. Generate a
Valkey password with `openssl rand -hex 32` and a TOTP key with
`openssl rand -base64 32`. Keep the TOTP key stable and backed up separately.

Set a stable, unique `INSTALLATION_ID`; a visible display name; separate database,
cache, bucket, credentials, hostnames, cookie names and host ports. The env display
name/capability override the JSON profile. Review currency and order policy before
importing. The supplied profile disables Telegram; do not substitute the bundled
CutRD profile, which enables Telegram. The staging override also clears its token.

Compose injects `.env` into the application. Keep database owner credentials and
registry tokens out of it. Profile paths are deployment-specific:
`./installation.json` is mounted read-only at `/app/config/installation.json`.
A missing host file fails mounting rather than silently creating a directory.

Use TLS with certificate verification for remote dependencies and a bucket-scoped
S3 credential. If PostgreSQL needs a private CA, install it in an ignored `certs/`
directory, add a read-only bind mount in a local Compose override, and append
`sslrootcert=/app/certs/postgres-ca.crt` to the database URL. The administration
connection needs the appropriate CA path on its own machine. URL-encode passwords.
Do not disable certificate verification to bypass missing CA configuration.

Use these shell helpers for the rest of this guide (choose exactly one):

```bash
# STAGING: use this for every command, including run/exec/pull/up/down.
dc() { docker compose -f compose.yaml -f compose.staging.yaml "$@"; }

# PRODUCTION alternative, once real SMTP is available and configured:
# dc() { docker compose -f compose.yaml "$@"; }
```

The examples assume you run commands from this checkout. Validate without printing
secrets, then authenticate using the same OS user that runs Compose:

```bash
dc config --quiet
docker login git.roselabs.mx
dc pull
```

Set `MAILPIT_IMAGE` to a verified pinned image before using staging. The Mailpit
SMTP port is private; only its UI is bound to loopback. No forwarding/relay is
configured. Its captured mail is ephemeral across container replacement.

## 2. Apply schema migrations and bind the database

Provision PostgreSQL 16 or a compatible supported server and the private bucket
first. Database setup is explicit: neither the image nor startup runs migrations.
Use separate owner and runtime LOGIN roles; the runtime role must not inherit the
owner or have superuser/role/database creation privileges.

Example staging SQL, run as a database administrator (adapt names for production):

```sql
CREATE ROLE cutrd_staging_owner LOGIN;
CREATE ROLE cutrd_staging_runtime LOGIN;
CREATE DATABASE cutrd_staging OWNER cutrd_staging_owner;
REVOKE ALL ON DATABASE cutrd_staging FROM PUBLIC;
GRANT CONNECT ON DATABASE cutrd_staging TO cutrd_staging_runtime;
```

Set generated role passwords securely through your administration tooling. Audit
access to other databases on shared servers as well.

Use migrations from the **same application release** (source checkout or binary
release archive), with an owner connection supplied securely as
`MIGRATION_DATABASE_URL`. From that migration directory's parent:

```bash
go run github.com/pressly/goose/v3/cmd/goose@v3.27.3 \
  -dir migrations postgres "$MIGRATION_DATABASE_URL" status
go run github.com/pressly/goose/v3/cmd/goose@v3.27.3 \
  -dir migrations postgres "$MIGRATION_DATABASE_URL" up
```

Current application migrations run through `00011_legacy_geometry_overrides.sql`.
Do not use `make migrate-up` from a checkout containing a development `.env`:
it sources that file. The consolidated `00001_baseline.sql` is for **empty
databases**, not the old migration history. Never edit Goose rows to force an
upgrade or drop valuable data to make the baseline fit.

Connect to the new staging database as its owner. Inspect any existing identity;
stop on a mismatch. For a fresh, unbound database, run once:

```sql
BEGIN;
INSERT INTO installation_identity (installation_id) VALUES ('cutrd-staging');
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO cutrd_staging_runtime;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public
  TO cutrd_staging_runtime;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public
  TO cutrd_staging_runtime;
-- Legacy quote import calls setval to preserve historical quote numbers.
GRANT UPDATE ON SEQUENCE quotes_id_seq TO cutrd_staging_runtime;
REVOKE ALL ON installation_identity, goose_db_version FROM cutrd_staging_runtime;
GRANT SELECT ON installation_identity TO cutrd_staging_runtime;
COMMIT;
```

Repeat applicable grants for new objects after future migrations and reapply the
identity/history restrictions. `DATABASE_URL` in the deployment uses the runtime
role, never `MIGRATION_DATABASE_URL`.

## 3. Provision, start and configure HTTPS

Stop existing app writers before provisioning. In a fresh staging deployment:

```bash
dc up -d valkey mailpit
# Production uses: dc up -d valkey
dc run --rm app provision-installation
dc up -d app
dc ps
dc logs --tail 100 app
```

Provisioning verifies the identity and initializes installation/order-policy
settings before ordinary writes. Existing populated databases cannot silently
switch order-numbering policy.

The app listens only on `127.0.0.1:APP_PORT` on the host (5601 in the staging
sample). Use distinct staff/portal hostnames and valid certificates; redirect
HTTP to HTTPS and reject unknown virtual hosts. Example Nginx location in **both**
staging TLS server blocks:

```nginx
location / {
    proxy_pass http://127.0.0.1:5601;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    client_max_body_size 6m;
    proxy_read_timeout 60s;
}
```

Use 5600 for the production sample. Preserve external Host and Origin: they
select the frontend and enforce unsafe-request checks. Configure
`HTTP_TRUSTED_PROXY_CIDRS` with the immediate proxy IP/CIDR **as seen by the
container**. Do not assume this is loopback across Docker networking, and never
trust all addresses. Restrict staging access to the trial team. Do not cache
private APIs, documents or responses that set cookies.

The homepage is not included in the image. Its separately deployed origin must
match `HOMEPAGE_APP_ORIGIN` if its public inquiry form calls this application.

```bash
curl -fsS https://staff.staging.example.com/health
curl -fsS https://staff.staging.example.com/ready
curl -fsS https://staff.staging.example.com/api/branding
```

Readiness checks database/cache/bucket access, not SMTP or PDF rendering. The
image includes Chromium and currently defaults its no-sandbox setting to true;
retain host/container isolation and size resources for document workloads.

## 4. Bootstrap without sending real email

For staging, keep the override active. It forces SMTP to Mailpit even if `.env`
contains a real provider. Do not configure Mailpit forwarding or relay.

```bash
dc exec app cutrdflexo bootstrap-admin --email admin@example.test --first-name Admin
```

From your workstation, tunnel to the host:

```bash
ssh -N -L 8025:127.0.0.1:8025 YOUR_USER@YOUR_HOST
```

Open `http://localhost:8025`, follow the staff invitation and set your password.
Mailpit contains login/reset tokens: keep it private. Bootstrap refuses once any
staff user exists. Verify password resets and both SPAs through their HTTPS URLs.
Production requires a working SMTP provider and real delivery testing before
accepting traffic; use the staging workflow while real mail is unavailable.

## 5. Rerun the legacy import

The importer is maintained separately. Create a dedicated active staff account,
then issue an expiring key from its account menu → API keys with:

```json
["clients:write", "machines:write", "legacy:write", "legacy:read"]
```

Set its target to `https://staff.staging.example.com` and use the new staging key.
Keep secrets and importer state outside this repository. For a fresh destination,
create a new mapping/checkpoint store; old destination IDs and asset UUIDs do not
belong to this database. Keep prior exports/audit logs for reconciliation.

1. Take a consistent legacy snapshot and record a watermark. Establish durable
   change capture first if ongoing mirroring is required.
2. Import clients, then resolve/import machines per client and persist mappings.
3. Upload special images before their quotes using
   `POST /api/v1/legacy/quotes/{id}/special-image?source=legacy-main` (multipart
   field `file`); retain each returned asset ID. Limits: 5 MiB image, 6 MiB body.
4. Import historical quotes with `PUT /api/v1/legacy/quotes/{id}` using those IDs.
   This preserves quote numbers, dates, explicit prices and historical states,
   and creates the corresponding orders. Do not use ordinary quote POSTs.
5. Reconcile counts by status, dates, prices, client/machine ownership, order links,
   images and PDFs against the source.

Start with a representative sample, then continue the same run. Keep ordinary
quote creation off until the legacy number range is loaded and prevent local
edits to mirrored records while syncing. Begin around 5 requests/second total.
Honor `Retry-After` on 429; retry quote PUTs with identical payload/version after
uncertain failures. Client/machine POSTs are not duplicate-safe: reconcile before
retrying. Investigate 409 conflicts and retain complete 422 validation details.

The `geometryOverride.reason` option is only for approved historical geometry
exceptions. It does not bypass missing images, relationships, dates or prices.
The earlier reported 516 quarantined and 9 rate-limited records require actual
reconciliation; deploying a release does not retry them. Legacy mirror writes
suppress normal integration events, but regular workflows retain their behavior.
Keep Mailpit enabled and Telegram disabled throughout staging.

The authoritative payload contracts are `docs/legacy-migration-staging.md` and
`docs/legacy-discovery-handoff.md` in the matching application checkout.

## Updates, rollback and recovery

Record the current image digest, configuration, installation ID and schema
version. Back up the database and bucket together, plus the stable TOTP key, and
test a restore. A database dump does not contain objects or PostgreSQL roles.

Test schema changes against an isolated restore with captured email. Use a
maintenance window unless compatibility with concurrent versions is proven:
stop app writers, apply migrations as owner, reapply grants, update
`CUTRDFLEXO_IMAGE`, then run:

```bash
dc pull app
dc up -d --force-recreate app
dc logs --tail 100 app
```

A simple `restart` does not apply changed environment, image or Compose settings.
Verify readiness, login, branding/capabilities, image upload/download, quotes,
orders and both PDFs. Reverting an image only works if the schema remains
compatible. Otherwise restore a coordinated database/object backup into an
isolated replacement with the correct identity and TOTP key. Do not run automatic
Down migrations or discard populated legacy mappings.

`dc down` preserves named volumes; `dc down -v` deletes the Valkey volume and its
sessions. Compose project names isolate containers/networks/volumes, but external
databases, buckets, credentials and host ports must also be separate. Keep
installation IDs stable across upgrades and coordinated restores.

## Secret handling

Never commit `.env`, owner credentials, registry tokens, CA/private-key material,
importer state or dumps. `docker compose config` can print secrets; prefer
`dc config --quiet` for validation. The container runtime and host administrators
can inspect injected environment variables. Keep deployment access restricted.
