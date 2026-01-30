# infra-edge

Centralized **edge / ingress layer** for multiple Docker-based projects on a single VPS.

This repository contains **only infrastructure**:
- Nginx (80 / 443)
- TLS certificates (Let’s Encrypt, mounted from host)
- Domain routing to internal Docker services via a shared network

Application projects (API, Frontend, etc.) **do not embed nginx** and are deployed independently.

---

## Architecture overview

```
Internet
   |
   | 80 / 443
   v
+----------------+
|   edge-nginx   |   (infra-edge repo)
+----------------+
        |
        | Docker network: edge
        |
+----------------------+----------------------+
| api-backend:4000    | todo-dashboard-app:3000 |
| (API project)       | (Next.js project)       |
+----------------------+----------------------+
```

### Domains

| Domain                  | Target service              |
|-------------------------|-----------------------------|
| dev.api.makestack.eu    | api-backend:4000            |
| dev.todo.makestack.eu   | todo-dashboard-app:3000     |

---

## Repository structure

```
infra-edge/
  docker-compose.yml
  nginx/
    nginx.conf
  .github/
    workflows/
      deploy.yml
  README.md
```

---

## Prerequisites (VPS)

- Linux VPS (Ubuntu recommended)
- Docker ≥ 24
- Docker Compose plugin
- DNS records pointing to the VPS
- SSH access (used by GitHub Actions)

---

## Docker network

All projects communicate through a **shared Docker network** named `edge`.

This network is created by `infra-edge` and reused by other projects.

```bash
docker network ls | grep edge
```

If manual creation is ever required:

```bash
docker network create edge
```

---

## TLS certificates (Let’s Encrypt)

Certificates are **not managed by containers**.
They are issued once on the VPS and mounted read-only into nginx.

### Initial certificate issuance

Stop any service currently binding port 80:

```bash
sudo systemctl stop nginx || true
docker stop edge-nginx || true
```

Issue certificates:

```bash
sudo certbot certonly --standalone   -d dev.api.makestack.eu   -d dev.todo.makestack.eu
```

Certificates will be stored in:

```
/etc/letsencrypt/live/
```

Restart infra-edge afterwards.

---

## Nginx configuration

Located at:

```
nginx/nginx.conf
```

Responsibilities:
- HTTP → HTTPS redirects
- TLS termination
- Reverse proxy to Docker services
- WebSocket support
- Zero-downtime reloads

---

## Local commands (on VPS)

### First-time setup

```bash
git clone git@github.com:<your-org>/infra-edge.git
cd infra-edge
docker compose up -d
```

### Validate nginx config

```bash
docker compose exec nginx nginx -t
```

### Reload nginx

```bash
docker compose exec nginx nginx -s reload
```

---

## GitHub Actions deployment

Workflow: `.github/workflows/deploy.yml`

Triggered on:
- push to `main`

What it does:
1. SSH into the VPS
2. Pull latest infra-edge changes
3. Run `docker compose up -d`
4. Validate nginx config
5. Reload nginx

No images are built.
No application containers are touched.

---

## Required GitHub Secrets

| Secret name     | Description          |
|-----------------|----------------------|
| VPS_HOST        | VPS IP / hostname    |
| VPS_USER        | SSH user             |
| VPS_SSH_KEY     | Private SSH key      |
| VPS_SSH_PORT    | SSH port             |

---

## How application projects integrate

Application projects must:

1. Not expose ports 80 / 443
2. Join the `edge` Docker network
3. Provide a stable container name

Example:

```yaml
services:
  app:
    container_name: todo-dashboard-app
    networks:
      - default
      - edge

networks:
  edge:
    external: true
    name: edge
```

---

## Deployment order

1. Deploy infra-edge
2. Deploy API project
3. Deploy Todo dashboard

infra-edge never depends on application projects.

---

## Adding a new domain

1. Add DNS record
2. Issue TLS certificate
3. Add server block in `nginx.conf`
4. Connect service to `edge`
5. Deploy infra-edge

---

## Troubleshooting

Check running containers:

```bash
docker ps
```

Inspect edge network:

```bash
docker network inspect edge
```

Test routing:

```bash
docker compose exec nginx sh
apk add curl
curl http://todo-dashboard-app:3000
```

View logs:

```bash
docker compose logs nginx
```

---

## Scope

This repository contains **only ingress infrastructure**.

No application code, databases, or secrets (except TLS certs).

---

## License

Private infrastructure repository.
