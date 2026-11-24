---
title: "Architecture Overview for docker-rom-stack"
tags: ["explanation", "architecture", "hsl", "docker-rom-stack"]
created_at: "2026-11-24"
---

# Architecture Overview for docker-rom-stack

This document explains the runtime container topology, volume persistence model, and network boundaries governing `docker-rom-stack`.

## 1. Homelab Systems Lifecycle (HSL) Boundary

`docker-rom-stack` operates under strict HSL separation:
* **Declarative Configuration:** Maintained in Git and synchronized to `/opt/homelab/prod/compose/docker-rom-stack/`.
* **State Persistence:** Preserved under `/opt/homelab/prod/data/docker-rom-stack/` with host-managed permissions.
* **Pre-Built OCI Images:** Worker nodes pull pre-compiled container packages from registries without local compilers.

## 2. Ingress & Reverse Proxying

Client traffic enters via the host network on `${SERVICE_HTTP_PORT:-8082}`. All internal container communication is strictly encapsulated within the Docker bridge network.
