# Ente Self-Hosted Stack for Synology

A Docker Compose stack for self-hosting [Ente](https://ente.io) (end-to-end encrypted photo storage) on a Synology NAS, managed via **Portainer** or **Dockhand**, served behind a **reverse proxy with HTTPS**.

This setup is based on Ente's official [self-hosting quickstart](https://ente.com/help/self-hosting/), adapted for Synology (bind mounts under `/volume1`, `restart: unless-stopped`, Docker Engine compatibility) and for running entirely behind subdomains of your own domain.

## Contents

- `docker-compose.yml` — the stack (museum, web, postgres, minio)
- `museum.yaml` — server configuration (mounted read-only into the `museum` container)

## Services

| Service | Image | Purpose |
|---|---|---|
| `museum` | `ghcr.io/ente-io/server` | API / backend |
| `web` | `ghcr.io/ente-io/web` | All Ente web apps (Photos, Accounts, Public Albums, Cast, Share, Embed, Paste) |
| `postgres` | `postgres:15` | Database |
| `minio` | `minio/minio` | S3-compatible object storage for photos/videos |

## Prerequisites

- Synology DSM with **Container Manager** (Docker) installed
- **Portainer** or **Dockhand** for stack management
- A domain you control with DNS access, for the reverse proxy subdomains
- A reverse proxy capable of TLS termination — DSM's built-in reverse proxy (Control Panel → Login Portal → Advanced → Reverse Proxy) is sufficient

## 1. Directory setup

On the Synology, create:

```sh
mkdir -p /volume1/docker/ente/data /volume1/docker/ente/postgres-data /volume1/docker/ente/minio-data
```

Place `museum.yaml` at `/volume1/docker/ente/museum.yaml`.

## 2. Replace placeholders

Both files contain `<PLACEHOLDER>` values that must be filled in **identically across both files**:

| Placeholder | Description |
|---|---|
| `myente.tld` | Your own domain (see port/domain table below) |
| `<POSTGRES_PASSWORD>` | Any strong password for the Postgres `ente` user |
| `<MINIO_ROOT_USER>` | A MinIO username, e.g. `ente-minio` |
| `<MINIO_ROOT_PASSWORD>` | Any strong password for MinIO |
| `<MUSEUM_ENCRYPTION_KEY>` | 32-byte base64 key: `head -c 32 /dev/urandom \| base64` |
| `<MUSEUM_HASH_KEY>` | 64-byte base64 key: `head -c 64 /dev/urandom \| base64` |
| `<MUSEUM_JWT_SECRET>` | 32-byte URL-safe base64 secret: `head -c 32 /dev/urandom \| base64 \| tr '+/' '-_'` |
| `<SMTP_PASSWORD>` | Password for your mail provider (for verification emails) |
| `smtp.example.com`, `mailbox@example.com` | Your mail provider's host/username/sender address |

> Note: the domain substitution produces `ente.myente.tld` for most subdomains (e.g. `api.ente.myente.tld`) but `photos.myente.tld` for the Photos app (no `ente.` prefix). This mirrors the original setup exactly — adjust freely if you prefer a flatter scheme.

## 3. Required ports & reverse proxy mapping

The `web` container serves several apps on different internal ports. Each one needs its own HTTPS subdomain pointing at the matching local port — Ente's apps expect to be served from a domain root (`/`), so a single shared domain with path-based routing is **not** supported.

| Local port (NAS) | App | Reverse proxy subdomain (example) |
|---|---|---|
| 8080 | Museum (API/backend) | `api.ente.myente.tld` |
| 3010 *(mapped from container port 3000)* | Photos web app | `photos.myente.tld` |
| 3001 | Accounts (passkeys) | `accounts.ente.myente.tld` |
| 3002 | Public Albums | `albums.ente.myente.tld` |
| 3003 | Auth *(not enabled by default)* | — |
| 3004 | Cast | `cast.ente.myente.tld` |
| 3005 | Share / Public Locker | `share.ente.myente.tld` |
| 3006 | Embed | `embed.ente.myente.tld` |
| 3008 | Paste | `paste.ente.myente.tld` |
| 3200 | MinIO (S3 API — used directly by the browser for uploads) | `storage.ente.myente.tld` |
| 3201 | MinIO web console *(optional, not exposed by default)* | — |
| 5432 | Postgres *(internal only, not exposed)* | — |

For each subdomain in DSM: **Control Panel → Login Portal → Advanced → Reverse Proxy**
- Source: subdomain, port 443, HTTPS
- Destination: `localhost:<port>`, HTTP
- A matching DNS A record and a Let's Encrypt certificate are required for each subdomain.

If your reverse proxy enforces a request body size limit (e.g. nginx `client_max_body_size`), raise it (or set to unlimited) for `storage.*` — otherwise large photo/video uploads will fail.

## 4. Deploy the stack

In Portainer or Dockhand: **Stacks → Add stack**, paste the contents of `docker-compose.yml`, deploy.

## 5. First-run configuration

### Create your account
Open `https://photos.myente.tld`, sign up. The verification code appears in the `museum` container logs unless SMTP is configured (see below).

### Configure SMTP (email verification codes)
Fill in the `smtp:` section in `museum.yaml` with your mail provider's details, then restart the `museum` container.

### Lock down registration
Once your own account exists, uncomment in `museum.yaml`:
```yaml
internal:
      disable-registration: true
```
Restart `museum`. New sign-ups are now blocked; existing accounts are unaffected.

### Make yourself an admin / lift the storage quota
By default, new accounts get a 10 GB trial quota — this is unrelated to actual disk space and must be raised manually:

1. Find your user ID:
   ```sh
   docker exec -it ente-postgres sh
   psql -U ente -d ente_db
   \d users
   SELECT * FROM users;
   ```
2. Add yourself as admin in `museum.yaml`:
   ```yaml
   internal:
         admins:
               - <YOUR_USER_ID>
   ```
3. Redeploy the stack, then use the [Ente CLI](https://ente.com/help/self-hosting/administration/cli) to lift your quota:
   ```sh
   ente admin update-subscription -a <admin-email> -u <your-email> --no-limit
   ```

### MinIO CORS
Browser uploads go directly to MinIO and require CORS to be open:
```sh
mc alias set ente http://<NAS-IP>:3200 <MINIO_ROOT_USER> <MINIO_ROOT_PASSWORD>
mc admin config set ente api cors_allow_origin="*"
mc admin service restart ente
```
(Run via `docker exec -it ente-minio sh` if `mc` isn't available on the host.)

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `can't set healthcheck.start_interval` | Synology's Docker Engine is older than v25; this field is already removed from `docker-compose.yml`. |
| `port is already allocated` | Another package/container is using that host port. Change the **left** (host) side of the `ports:` mapping, keep the right side unchanged, and update the matching `ENTE_*_ORIGIN`/`apps:` value to match. |
| `Die folgenden Dateien wurden nicht hochgeladen` / uploads fail | MinIO's `endpoint` in `museum.yaml` must be reachable **from the browser**, not just from inside the Docker network — never use `localhost`. Also check the MinIO CORS step above. |
| `You trusted all proxies, this is NOT safe` | Harmless Gin framework startup warning, not Ente-specific. Only relevant once you're behind a reverse proxy that could have its `X-Forwarded-For` header spoofed; otherwise safe to ignore. |
| `unable to create new parent process: namespace path: ... no such file or directory` | Generic Docker/containerd race condition, typically from a container using `network_mode: service:X` against a container that was just recreated. Remove the stack fully (not just stop) and redeploy. |

## Security notes

- Treat `museum.yaml` and the Postgres/MinIO credentials as secrets — don't commit filled-in values to a public repository.
- TLS should always terminate at the reverse proxy; `museum` itself only needs to listen on plain HTTP internally.
- Disable registration (see above) once your account(s) are created, especially if any subdomain is reachable from the public internet.
