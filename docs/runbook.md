# Self-Hosted Homelab Runbook

## Architecture

Refer [README.md](/README.md#architecture)

### Conventions

- Docker Compose manages applications.
- Web applications join the external Docker network `homelab`.
- Caddy is the single HTTP/HTTPS ingress.
- Applications normally use `expose`, not host `ports`.
- Caddy reaches applications by Docker container/service name.
- Persistent application state lives in volumes.
- Cloudflare provides authoritative DNS.
- Caddy obtains wildcard certificates using Cloudflare DNS-01.
- AdGuard Home provides LAN DNS, filtering, and optional DNS rewrites.
- Tailscale is preferred for private remote access.
- Watchtower updates only explicitly opted-in containers.

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

    expose:
      - "<internal-port>"

    networks:
      - homelab

    volumes:
      - <persistent-volume>:/path/in/container

networks:
  homelab:
    external: true
```

Prefer:

```yaml
expose:
  - "3000"
```

over:

```yaml
ports:
  - "3000:3000"
```

unless direct LAN access is actually required.

### 2. Start and inspect

```bash
docker compose up -d
docker ps
docker logs <app-container> --tail 100
```

Verify networking:

```bash
docker network inspect homelab
docker inspect <app-container>   --format '{{json .NetworkSettings.Networks}}'
```

### 3. Test the app before Caddy

From Caddy:

```bash
docker exec caddy getent hosts <app-container>
docker exec caddy wget -qO- http://<app-container>:<port> | head
```

If this fails, do not debug DNS or TLS yet. Fix Docker networking, the port, or the application first.

### 4. Add Caddy

```caddyfile
<app-subdomain>.<domain> {
    reverse_proxy <app-container>:<port>
}
```

Validate:

```bash
docker exec caddy caddy validate   --config /etc/caddy/Caddyfile
```

Inspect adapted config:

```bash
docker exec caddy caddy adapt   --config /etc/caddy/Caddyfile   --pretty
```

Reload:

```bash
docker exec caddy caddy reload   --config /etc/caddy/Caddyfile
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
curl -s   -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"   -H "Content-Type: application/json"   "https://api.cloudflare.com/client/v4/zones?name=<domain>"
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
docker inspect caddy   --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}'
```

Check adapted config:

```bash
docker exec caddy caddy adapt   --config /etc/caddy/Caddyfile   --pretty
```

For Caddyfile-only changes, reload:

```bash
docker exec caddy caddy reload   --config /etc/caddy/Caddyfile
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
Browser --HTTPS--> Caddy --Docker network--> AdGuard :3000
```

The DNS service needs host port 53. The web UI can be reached by Caddy over `homelab`.

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
      - "3000:3000/tcp"

    volumes:
      - <work-volume>:/opt/adguardhome/work
      - <config-volume>:/opt/adguardhome/conf

    networks:
      - homelab

networks:
  homelab:
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

---

## Watchtower

Watchtower does not need to be on `homelab`.

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
Caddy <---- homelab ----> Applications
Watchtower ---- Docker socket ----> Docker daemon
```

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
- check both containers are on `homelab`

If upstream works, inspect Caddy:

```bash
docker exec caddy caddy adapt   --config /etc/caddy/Caddyfile   --pretty
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

docker network inspect homelab

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

# New-app checklist

```text
[ ] Stable container name
[ ] Internal port identified
[ ] Persistent volumes identified
[ ] Service attached to external `homelab`
[ ] No unnecessary host port
[ ] Container starts successfully
[ ] Logs checked
[ ] Caddy can reach upstream directly
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

# Debugging order

```text
host configuration
      ↓
container configuration
      ↓
process/application
      ↓
Docker networking
      ↓
DNS
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
