# caddy

Caddy reverse proxy config for local self-hosted Docker apps. Routes clean
LAN hostnames to services running in other Compose stacks via a shared external
Docker network.

## Architecture

```
Phone / TV / browser
       │  (hostname DNS resolved by UniFi router → 192.168.1.42)
       ▼
  caddy :80
  ├── http://tensara   → frontend:3000  (tensara's nginx frontend)
  └── http://pickarr  → pickarr:8787   (pickarr service)
```

All proxied services join the shared external Docker network `proxy` so Caddy
can reach them by service name. The existing host-port bindings on each stack
(`:3000`, `:3001`, `:8000`) are kept as a direct-access fallback.

v1 is HTTP-only. Future TLS options:
- `tls internal` (self-signed, requires installing Caddy CA root on every device)
- DNS-01 ACME with a real domain (best for iOS PWA / trusted certs)

## One-time setup

### 1. Create the shared Docker network

```bash
docker network create proxy
```

This only needs to be done once per host. The network persists across restarts.

### 2. Configure DNS in your UniFi router

Add host/A records mapping each app hostname to your host machine's LAN IP
(e.g. `192.168.1.42`):

| Hostname | Address      |
|----------|-------------|
| tensara  | 192.168.1.42 |
| pickarr  | 192.168.1.42 |

### 3. Start Caddy

```bash
docker compose up -d
```

Check for Caddyfile errors:

```bash
docker logs caddy
```

### 4. Join each app to the proxy network

Each app's `docker-compose.yml` must declare the `proxy` external network and attach
its frontend service to it. See the individual app repos for the exact changes.

## Adding a new app

1. In the app's `docker-compose.yml`, add:
   ```yaml
   services:
     your-service:
       networks: [default, proxy]
   networks:
     proxy:
       external: true
   ```

2. Add a site block to `Caddyfile`:
   ```
   http://your-app {
       reverse_proxy your-service:port
   }
   ```

3. Add a DNS record in UniFi: `your-app` → your LAN IP.

4. Reload Caddy (no restart needed):
   ```bash
   docker exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```
