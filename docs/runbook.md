# Self-Hosted Homelab Runbook

## Architecture

```text
Clients
  |
  +--> DNS (AdGuard Home / Cloudflare)
  |
  +--> HTTPS
          |
          v
       Caddy
          |
          +--> bentopdf network
          |       |
          |     BentoPDF
          |
          +--> silverbullet network
          |       |
          |     SilverBullet
          |
          +--> adguard network
          |       |
          |     AdGuard Home
          |
          +--> karakeep network
                  |
                Karakeep Web
                  |
                  +--> karakeep-internal
                          |
                          +--> Chrome
                          +--> Meilisearch
```

### Network isolation

Each application normally has its own dedicated Docker network.

Caddy joins every application network so it can reverse proxy to the application.

Applications on different Docker networks cannot directly communicate with each other.

Internal application dependencies can use a separate private network.

```text
Caddy
  |
  +--> bentopdf -------> BentoPDF
  |
  +--> silverbullet ---> SilverBullet
  |
  +--> adguard --------> AdGuard Home
  |
  +--> karakeep -------> Karakeep Web
                            |
                            +--> karakeep-internal --> Chrome
                            |
                            +--> karakeep-internal --> Meilisearch
```

The important rule is:

```text
Application networks are isolated from each other.

Caddy is intentionally attached to each application network
because it is the reverse-proxy ingress.
```

### Conventions

- Docker Compose manages applications.
- Each web application normally has its own dedicated Docker network.
- Caddy is the single HTTP/HTTPS ingress.
- Caddy joins each application's network.
- Applications normally do not use host `ports`.
- `expose` is optional and is not required for Docker-to-Docker connectivity.
- Caddy reaches applications by Docker container/service name.
- Persistent application state lives in volumes.
- Cloudflare provides authoritative DNS.
- Caddy obtains wildcard certificates using Cloudflare DNS-01.
- AdGuard Home provides LAN DNS, filtering, and optional DNS rewrites.
- Tailscale is preferred for private remote access.
- Watchtower updates only explicitly opted-in containers.
- Internal application dependencies may use a separate `internal` Docker network when they should not have external connectivity.

## The three states

Always distinguish:

1. **Configuration state** — files on the host.
2. **Container state** — what the running container actually sees.
3. **Application/network state** — what the process is actually doing.

Example for Caddy:

```bash
cat Caddyfile
docker exec caddy cat /etc/caddy/Caddyfile
docker exec caddy caddy adapt --config /etc/caddy/Caddyfile --pretty
```

A changed host file does not automatically mean the process has loaded it.

---

## Adding a new application

### 1. Compose

Use:

```yaml
services:
  app:
    image: <image>
    container_name: <app-container>
    restart: unless-stopped

    environment:
      # APP_SETTING=${APP_SETTING}

    networks:
      - app

    volumes:
      - <persistent-volume>:/path/in/container

networks:
  app:
    name: app
```

The application Compose project creates its dedicated network.

The name can be changed to match the application:

```yaml
networks:
  app:
    name: myapp
```

Caddy can then join that network as an external network.

`expose` is optional:

```yaml
expose:
  - "3000"
```

It is useful as documentation, but it is **not required** for another container on the same Docker network to connect to port `3000`.

Prefer no host `ports`:

```yaml
# No ports section
```

over:

```yaml
ports:
  - "3000:3000"
```

unless direct LAN/host access is actually required.

### 2. Start and inspect

```bash
docker compose up -d
docker ps
docker logs <app-container> --tail 100
```

Verify networking:

```bash
docker network inspect <app-network>
docker inspect <app-container> \
  --format '{{json .NetworkSettings.Networks}}'
```

Verify that Caddy is attached to the same network:

```bash
docker inspect caddy \
  --format '{{json .NetworkSettings.Networks}}'
```

### 3. Test the app before Caddy

From Caddy:

```bash
docker exec caddy getent hosts <app-container>
docker exec caddy wget -qO- http://<app-container>:<port> | head
```

If this fails, do not debug DNS or TLS yet. Fix Docker networking, the port, or the application first.

The important requirement is:

```text
Caddy
  |
  +--> <app-network>
          |
          +--> <app-container>
```

The application does not need to share a network with other applications.

### 4. Add Caddy

Add the application network to the Caddy Compose file:

```yaml
services:
  caddy:
    networks:
      - <app-network>

networks:
  <app-network>:
    external: true
```

For example:

```yaml
services:
  caddy:
    networks:
      - bentopdf
      - silverbullet
      - adguard
      - karakeep

networks:
  bentopdf:
    external: true
  silverbullet:
    external: true
  adguard:
    external: true
  karakeep:
    external: true
```

Then add the route:

```caddyfile
<app-subdomain>.<domain> {
    reverse_proxy <app-container>:<port>
}
```

Validate:

```bash
docker exec caddy caddy validate \
  --config /etc/caddy/Caddyfile
```

Inspect adapted config:

```bash
docker exec caddy caddy adapt \
  --config /etc/caddy/Caddyfile \
  --pretty
```

Reload:

```bash
docker exec caddy caddy reload \
  --config /etc/caddy/Caddyfile
```

### 5. DNS

For LAN-only access, optionally add an AdGuard DNS rewrite:

```text
<app-subdomain>.<domain> -> <homelab-LAN-IP>
```

Verify:

```bash
dig <app-subdomain>.<domain>
dig <app-subdomain>.<domain> @<adguard-ip>
```

### 6. HTTPS

```bash
curl -vk https://<app-subdomain>.<domain>
curl -s https://<app-subdomain>.<domain> | head
```

Do not rely only on `curl -I`; a `200` response can still have an empty or incorrect body.

---

## TLS / Cloudflare

Wildcard configuration:

```caddyfile
*.<domain> {
    tls {
        dns cloudflare {$CLOUDFLARE_API_TOKEN}
        resolvers 1.1.1.1
    }
}
```

`resolvers 1.1.1.1` is for Caddy's DNS-01 certificate workflow. It does not replace AdGuard as the LAN resolver.

Verify token presence without printing it:

```bash
docker exec caddy sh -c '
if [ -n "$CLOUDFLARE_API_TOKEN" ]; then
  echo "TOKEN_PRESENT"
else
  echo "TOKEN_MISSING"
fi
'
```

Verify the Cloudflare API:

```bash
docker exec caddy sh -c '
curl -s \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones?name=<domain>"
'
```

Check ACME:

```bash
docker logs caddy --tail 100 | grep -i -E 'dns-01|authorization|certificate'
```

`_acme-challenge` is temporary. An NXDOMAIN after successful certificate issuance is not automatically a problem.

### Token rotation

If the token is supplied through the container environment, changing `.env` is not enough. Recreate Caddy:

```bash
docker compose up -d --force-recreate caddy
```

Then verify:

```bash
docker exec caddy sh -c '
if [ -n "$CLOUDFLARE_API_TOKEN" ]; then
  echo "TOKEN_PRESENT"
else
  echo "TOKEN_MISSING"
fi
'
```

Never print or commit the token.

---

## Caddy configuration vs container state

Check host config:

```bash
cat Caddyfile
```

Check mounted config:

```bash
docker exec caddy cat /etc/caddy/Caddyfile
```

Check mounts:

```bash
docker inspect caddy \
  --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}'
```

Check adapted config:

```bash
docker exec caddy caddy adapt \
  --config /etc/caddy/Caddyfile \
  --pretty
```

For Caddyfile-only changes, reload:

```bash
docker exec caddy caddy reload \
  --config /etc/caddy/Caddyfile
```

For image/environment/network/Compose changes, recreate:

```bash
docker compose up -d --force-recreate caddy
```

This distinction caused one of the main debugging issues: the host Caddyfile, container-mounted Caddyfile, and active Caddy configuration are separate states.

---

## AdGuard Home

AdGuard has two roles:

```text
LAN clients --DNS :53--> AdGuard

Browser --HTTPS--> Caddy --adguard network--> AdGuard :3000
```

The DNS service needs host port 53.

The web UI does **not** need a host port because Caddy reaches it over the dedicated `adguard` Docker network.

Example:

```yaml
services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard
    restart: unless-stopped

    ports:
      - "<LAN-IP>:53:53/tcp"
      - "<LAN-IP>:53:53/udp"

    volumes:
      - <work-volume>:/opt/adguardhome/work
      - <config-volume>:/opt/adguardhome/conf

    networks:
      - adguard

networks:
  adguard:
    name: adguard
```

Do **not** publish:

```yaml
- "3000:3000/tcp"
```

Caddy joins the network:

```yaml
services:
  caddy:
    networks:
      - adguard

networks:
  adguard:
    external: true
```

Caddy:

```caddyfile
adguard.<domain> {
    reverse_proxy adguard:3000
}
```

Verify:

```bash
docker exec caddy wget -qO- http://adguard:3000 | head
```

Verify the DNS port remains published:

```bash
docker port adguard
```

Expected:

```text
53/tcp -> <LAN-IP>:53
53/udp -> <LAN-IP>:53
```

The important distinction is:

```text
DNS:
LAN client -> host:53 -> AdGuard

Web UI:
Browser -> Caddy:443 -> adguard network -> AdGuard:3000
```

---

## Application networks and internal dependencies

A simple application normally needs only one dedicated network:

```yaml
services:
  app:
    networks:
      - app

networks:
  app:
    name: app
```

If an application has internal dependencies, it can use two networks.

Example:

```text
Caddy
  |
  +--> karakeep
          |
          +--> karakeep-internal
                  |
                  +--> Chrome
                  +--> Meilisearch
```

Example:

```yaml
services:
  web:
    image: ghcr.io/karakeep-app/karakeep:${KARAKEEP_VERSION:-release}
    container_name: karakeep-web
    restart: unless-stopped

    volumes:
      - $HOME/.karakeep/data:/data

    env_file:
      - .env

    environment:
      MEILI_ADDR: http://meilisearch:7700
      BROWSER_WEB_URL: http://chrome:9222
      DATA_DIR: /data

    networks:
      - karakeep
      - karakeep-internal

  chrome:
    image: ghcr.io/karakeep-app/karakeep-chrome:release
    restart: unless-stopped
    init: true

    networks:
      - karakeep-internal

  meilisearch:
    image: getmeili/meilisearch:v1.41.0
    restart: unless-stopped

    volumes:
      - meilisearch:/meili_data

    networks:
      - karakeep-internal

volumes:
  meilisearch:

networks:
  karakeep:
    name: karakeep

  karakeep-internal:
    internal: true
```

Only `web` joins the `karakeep` network, so Caddy can reach it.

Chrome and Meilisearch remain isolated on `karakeep-internal`.

The `karakeep-internal` network is application-owned and does not need to be declared as external.

---

## Watchtower

Watchtower does not need to be on any application network.

It communicates with Docker through:

```text
/var/run/docker.sock
```

Typical opt-in label:

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

Mental model:

```text
Caddy
  |
  +---- application networks ----> Applications

Watchtower ---- Docker socket ----> Docker daemon
```

Watchtower therefore remains independent from application network isolation.

---

# Troubleshooting decision tree

## 1. Hostname does not resolve

```bash
dig <hostname>
dig <hostname> @<adguard-ip>
```

Check:

- DNS record/rewrite exists.
- Correct DNS server is being queried.
- AdGuard is running.
- Hostname is spelled correctly.

Do not debug Caddy until DNS works.

## 2. Hostname resolves but HTTPS fails

```bash
curl -vk https://<hostname>
docker logs caddy --tail 100
```

Look for certificate, ACME, TLS, or hostname errors.

## 3. HTTPS works but returns 404/502

Test from Caddy:

```bash
docker exec caddy getent hosts <app-container>
docker exec caddy wget -qO- http://<app-container>:<port> | head
```

If upstream fails:

- check container is running
- check internal port
- check application bind address
- check Caddy and the application are on the same application network
- check the application network exists
- check Caddy is attached to that network

Inspect the network:

```bash
docker network inspect <app-network>
```

If upstream works, inspect Caddy:

```bash
docker exec caddy caddy adapt \
  --config /etc/caddy/Caddyfile \
  --pretty
```

## 4. HTTPS returns 200 but UI is blank

Check the actual body:

```bash
curl -s https://<hostname> | head -c 1000
```

Check assets:

```bash
curl -I https://<hostname>/<asset>
```

Compare directly with the upstream:

```bash
docker exec caddy sh -c '
curl -sS -D- "http://<app-container>:<port>/" -o /tmp/body
wc -c /tmp/body
'
```

Then inspect browser Network/Console logs.

A `200` with `Content-Length: 0` is a strong signal that the proxy/application response needs investigation.

---

# Useful commands

```bash
docker ps
docker ps -a
docker inspect <container>
docker logs <container> --tail 100

docker network ls
docker network inspect <app-network>

docker inspect <container> \
  --format '{{json .NetworkSettings.Networks}}'

docker inspect caddy \
  --format '{{json .NetworkSettings.Networks}}'

docker exec caddy getent hosts <container>
docker exec caddy wget -qO- http://<container>:<port>

dig <hostname>
dig <hostname> @<dns-server>

curl -I https://<hostname>
curl -vk https://<hostname>
curl -s https://<hostname> | head

docker exec caddy cat /etc/caddy/Caddyfile
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
docker exec caddy caddy adapt --config /etc/caddy/Caddyfile --pretty
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

For a specific application:

```bash
docker network inspect <app-network>
```

For AdGuard:

```bash
docker port adguard
```

---

# New-app checklist

```text
[ ] Stable container name
[ ] Internal port identified
[ ] Persistent volumes identified
[ ] Dedicated application Docker network
[ ] Caddy attached to the application network
[ ] No unnecessary host port
[ ] `expose` used only if desired for documentation
[ ] Container starts successfully
[ ] Logs checked
[ ] Caddy can resolve the application
[ ] Caddy can reach the application directly
[ ] Caddy route added
[ ] Caddy config validated
[ ] Caddy config adapted/inspected
[ ] Caddy reloaded
[ ] DNS verified
[ ] HTTPS verified
[ ] Actual GET body verified
[ ] Browser UI verified
[ ] Persistence verified after restart
[ ] Watchtower opt-in considered
```

For applications with dependencies:

```text
[ ] Internal dependency network identified
[ ] Internal network is separate from the Caddy-facing network
[ ] `internal: true` used when external connectivity is intentionally blocked
[ ] Only the application component that Caddy needs is attached to the Caddy-facing network
```

For AdGuard:

```text
[ ] DNS port 53 published to the LAN IP
[ ] DNS TCP and UDP both published
[ ] Web UI port 3000 not published to the host
[ ] Caddy attached to the `adguard` network
```

---

# Debugging order

```text
host configuration
      ↓
container configuration
      ↓
process/application
      ↓
Docker network membership
      ↓
Docker internal DNS
      ↓
upstream connectivity
      ↓
LAN DNS
      ↓
TLS
      ↓
Caddy routing
      ↓
HTTP response body
      ↓
browser assets/UI
```

Do not skip layers just because the browser is the thing that appears broken.
