---
title: "Environment Variables Reference for docker-rom-stack"
tags: ["reference", "environment", "configuration", "docker-rom-stack"]
created_at: "2026-11-24"
---

# Environment Variables Reference for docker-rom-stack

Authoritative specification of configuration options for `docker-rom-stack`.

| Variable | Default | Description | Environment |
| :--- | :--- | :--- | :--- |
| `APP_ENV` | `dev` | Application lifecycle mode (`dev`, `stage`, `prod`). | All |
| `SERVICE_HTTP_PORT` | `8082` | Ingress port exposed on the host. | All |
| `DATA_DIR` | `./data` | Persistent host bind path (overridden by Ansible in prod). | All |
| `LOG_LEVEL` | `info` | Operational logging verbosity (`debug`, `info`, `warn`, `error`). | All |
