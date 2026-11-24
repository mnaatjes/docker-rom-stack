---
title: "Deploy docker-rom-stack Locally"
tags: ["how-to", "docker", "local", "docker-rom-stack"]
created_at: "2026-11-24"
---

# Deploy docker-rom-stack Locally

Practical runbook for running `docker-rom-stack` on developer workstations.

## Prerequisites

* Docker Engine 24+ & Docker Compose v2.
* Port `8082` available on host.

## Commands

```bash
cp .env.example .env
docker compose up -d
docker compose ps
```
