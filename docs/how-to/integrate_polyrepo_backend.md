---
title: "Integrate Polyrepo Backend Container Image"
tags: ["how-to", "polyrepo", "backend", "docker", "docker-rom-stack"]
created_at: "2026-11-24"
---

# Integrate Polyrepo Backend Container Image

This runbook documents how to connect an independently developed backend service repository (e.g. authored via `python-hexagonal-template`) into `docker-rom-stack`.

## 1. Upstream Backend Release Contract

The upstream backend repository must:
1. Compile and publish an OCI container image to GitHub Container Registry:
   `ghcr.io/mnaatjes/docker-rom-stack-api:vX.Y.Z`
2. Expose internal port `8000` (or configured API port).
3. Provide a `/health` endpoint validating database and application readiness.

## 2. Compose Wiring in docker-rom-stack

In `compose.yml`:
```yaml
services:
  backend:
    image: ${BACKEND_IMAGE:-ghcr.io/mnaatjes/docker-rom-stack-api:latest}
    restart: unless-stopped
    environment:
      - APP_ENV=${APP_ENV:-dev}
      - PORT=8000
    volumes:
      - "${DATA_DIR:-./data}:/app/data"
    networks:
      - docker_rom_stack_net
```

## 3. Reverse Proxy Proxying

Frontend Nginx receives external client requests and forwards `/api/` calls across the internal Docker bridge network:
```nginx
location /api/ {
    proxy_pass http://backend:8000;
}
```
