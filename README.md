# Entrack Plataforma deployment

This template runs the dashboard, business backend, and custom receiver behind
an Nginx gateway on one public web endpoint. It pulls the application images
from GitHub Container Registry.

## Start

From this deployment folder:

```sh
cp .env.example .env
# Edit .env and set the external database connection and release versions.
docker compose pull
docker compose up -d
docker compose ps
```

Open <http://localhost>. The receiver connects to the external PostgreSQL database
configured through `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD`.
Use `DB_SSL_MODE=require` for a TLS-enabled external database, or the mode
specified by the customer's provider. On a new empty database, the receiver
applies its Liquibase schema automatically. Uploaded media and logs live in
named Docker volumes and survive container replacement.

The gateway serves HTTP on host port 80. You are responsible for DNS, HTTPS
termination, and firewall rules.

GPS devices connect directly to the receiver ports (`5000-5300` TCP/UDP). If a
hostname is used for device connections, its Cloudflare record must be DNS-only;
Cloudflare's standard HTTP proxy does not forward tracker TCP/UDP traffic.

The gateway sends tracking requests to the receiver, business and chat requests
to the backend, and everything else to the dashboard. 

Use `DASHBOARD_VERSION`, `BACKEND_VERSION`, and `RECEIVER_VERSION` to choose the
application versions. For production, use fixed release tags instead of
`latest`.

Configure the relevant SMTP, Firebase, SMS, geocoder, geolocation, OpenID `.env`

## Camera streaming

Set `MEDIAMTX_WEBRTC_URL`, `MEDIAMTX_RTMP_URL`, and optionally
`MEDIAMTX_RTSP_URL` to the external service endpoints. The business backend and
receiver receive these values through the common environment.

```sh
printf '%s' "$GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USER --password-stdin
```

## Device ports

The receiver publishes TCP and UDP ports `5000-5300`. You can reduce
this range to only the protocols their devices use. For example, GT06 uses
5023 and Teltonika uses 5027 in the supplied receiver configuration.

## Operations

```sh
docker compose logs -f receiver
docker compose pull
docker compose up -d
docker compose down
```

`docker compose down` keeps media and logs. Adding `--volumes` deletes those
local volumes, but never deletes or manages the customer's external database.
