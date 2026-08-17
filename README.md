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

The template intentionally serves plain HTTP on host port 80. DNS, a Cloudflare
proxy, TLS mode, certificates, and the server firewall remain the customer's
responsibility. If Cloudflare proxies the web hostname, configure the origin to
accept Cloudflare traffic on port 80. GPS device ports must remain directly
reachable and should use a DNS-only hostname because a standard Cloudflare DNS
proxy does not carry arbitrary tracker TCP/UDP protocols.

The gateway preserves the existing same-origin routes: `/api/*` and
`/api/socket` go to the receiver, while `/functions/v1/*` and `/chatkit` go to
the business backend. All other requests go to the dashboard. This preserves
the current `/functions/v1/reseller-api` contract without requiring Cloudflare
Pages Functions. Application logic previously implemented by those functions
must be provided by the receiver or business backend. Set
`DASHBOARD_VERSION`, `BACKEND_VERSION`, and `RECEIVER_VERSION` to published
release tags. `latest` is convenient for evaluation, but immutable version tags
are safer in production.

Receiver integrations are disabled by default. Configure the relevant SMTP,
Firebase, SMS, geocoder, geolocation, OpenID, or forwarding values in `.env`
before adding their notification types or changing their `*_ENABLE` flags.

## Camera streaming

MediaMTX is customer-managed and is not hosted by this Compose project. Set
`MEDIAMTX_WEBRTC_URL`, `MEDIAMTX_RTMP_URL`, and optionally
`MEDIAMTX_RTSP_URL` to the external service endpoints. The business backend and
receiver receive these values through the common environment.

The customer must configure MediaMTX networking, TLS, authentication, storage,
and firewall access. A standard Cloudflare proxy does not carry RTMP, RTSP, or
WebRTC UDP media traffic, so its media hostname normally needs to be DNS-only.
Enter the same WebRTC and RTMP endpoints in the dashboard Camera Settings until
the business backend exposes centralized runtime settings to the frontend.

If either GitHub Container Registry package is private, authenticate once using
a GitHub personal access token with `read:packages` before starting:

```sh
printf '%s' "$GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USER --password-stdin
```

## Device ports

Following the official Traccar Docker configuration, the receiver publishes
TCP and UDP ports `5000-5300`. For an internet-facing host, customers can reduce
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
