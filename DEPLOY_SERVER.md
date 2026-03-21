# Deploy Fynivo Landing on `fynivo.in` (Oracle / docker-compose + Caddy)

The landing app is a **Vite + React** static site. This repo includes a **Dockerfile** that builds with `npm run build` and serves files with **nginx**.

## 1. Put the code on the server

You need this folder on the VM (same machine where `/home/opc/fynivo/docker-compose.yml` lives).

**Option A — inside the fynivo stack folder (recommended):**

```bash
cd /home/opc/fynivo
git clone <your-FynivoLanding-repo-url> FynivoLanding
```

**Option B — sibling folder:**

```bash
cd /home/opc
git clone <your-FynivoLanding-repo-url> FynivoLanding
```

If you use Option B, set `build: /home/opc/FynivoLanding` in `docker-compose.yml` below.

## 2. Add service to `docker-compose.yml`

In `/home/opc/fynivo/docker-compose.yml`, add:

```yaml
  landing:
    build: ./FynivoLanding
    container_name: fynivo-landing
    restart: unless-stopped
    networks:
      - fynivo_net
```

- If the repo path is not `./FynivoLanding`, use an absolute path, e.g. `build: /home/opc/FynivoLanding`.

Add **`landing`** to the **`proxy`** service `depends_on` list so Caddy starts after the container exists:

```yaml
    depends_on:
      - ui
      - landing
      - login
      # ... rest
```

## 3. Caddy — point `fynivo.in` at the landing container

Edit `/home/opc/fynivo/proxy/Caddyfile` and add (keep `app.fynivo.in` as your main app UI):

```caddy
fynivo.in, www.fynivo.in {
    reverse_proxy landing:80
}
```

- **`landing`** is the Docker Compose **service name** (resolves on `fynivo_net`).
- Caddy will obtain TLS certificates automatically once DNS points to this server.

## 4. DNS

At your DNS provider:

- **A** record: `fynivo.in` → server public IP  
- **A** (or CNAME) for `www.fynivo.in` → same (or CNAME to `fynivo.in`)

## 5. Deploy

```bash
cd /home/opc/fynivo
docker compose up -d --build landing proxy
docker compose ps
```

Or extend your `deploy.sh` with a `landing` target (see below).

## 6. Optional: `deploy.sh` snippet

```bash
  landing)
    pull_repo /home/opc/fynivo/FynivoLanding
    deploy_one landing
    ;;
```

And in the menu script, add an option that runs `./deploy.sh landing`.

## Troubleshooting

| Issue | Check |
|--------|--------|
| 502 from Caddy | `docker compose ps` — is `fynivo-landing` up? `docker compose logs landing` |
| Wrong site / old build | Rebuild: `docker compose build --no-cache landing` then `up -d` |
| Certificate errors | DNS must resolve to this host before Caddy can issue certs |
