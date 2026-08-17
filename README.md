# Entrack Plataforma deployment

This template runs the dashboard, business backend, and custom receiver service
behind one public web endpoint. It pulls the application images from GitHub
Container Registry, so customers do not need the source repositories to launch
it. The database is provided and managed by the customer; it is not part of
this Compose project.

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

The dashboard uses the existing same-origin routes: `/api/*` and `/api/socket`
go to the receiver, while `/functions/v1/*` goes to the business backend. This
preserves the current `/functions/v1/reseller-api` contract. Set
`DASHBOARD_VERSION`, `BACKEND_VERSION`, and `RECEIVER_VERSION` to published
release tags. `latest` is convenient for evaluation, but immutable version tags
are safer in production.

Receiver integrations are disabled by default. Configure the relevant SMTP,
Firebase, SMS, geocoder, geolocation, OpenID, or forwarding values in `.env`
before adding their notification types or changing their `*_ENABLE` flags.

If either GitHub Container Registry package is private, authenticate once using
a GitHub personal access token with `read:packages` before starting:

```sh
printf '%s' "$GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USER --password-stdin
```

## Device ports

The template publishes TCP and UDP ports 5000-5250 so the protocols enabled in
the current `traccar.xml` work without further setup. For an internet-facing
host, reduce this to only the ports used by your devices. For example, GT06 is
5023 and Teltonika is 5027 in the supplied configuration.

## Operations

```sh
docker compose logs -f receiver
docker compose pull
docker compose up -d
docker compose down
```

`docker compose down` keeps media and logs. Adding `--volumes` deletes those
local volumes, but never deletes or manages the customer's external database.
