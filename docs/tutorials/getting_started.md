---
title: "Getting Started with docker-rom-stack"
tags: ["tutorial", "onboarding", "docker-rom-stack"]
created_at: "2026-11-24"
---

# Getting Started with docker-rom-stack

This tutorial walks through spinning up `docker-rom-stack` in local development.

## Steps

1. Copy `.env.example` to `.env`.
2. Inspect `SERVICE_HTTP_PORT` (defaults to `8082`).
3. Launch with `docker compose up -d`.
4. Validate service health at `http://localhost:8082`.
