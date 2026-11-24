---
title: "Configure Production Environment for docker-rom-stack"
tags: ["how-to", "production", "ansible", "hsl", "docker-rom-stack"]
created_at: "2026-11-24"
---

# Configure Production Environment for docker-rom-stack

Runbook for onboarding `docker-rom-stack` into `homelab-ops` Ansible automation.

## 1. Register Stack in Workload Inventory

In `homelab-ops/group_vars/all/workloads/compose_stacks.yml`:

```yaml
compose_stacks:
  docker-rom-stack:
    repo_url: "https://github.com/mnaatjes/docker-rom-stack.git"
    version: "v1.0.0"
    target_hosts: ["prd-worker-01.lan"]
    environment_variables:
      SERVICE_HTTP_PORT: 8082
      DATA_DIR: "/opt/homelab/prod/data/docker-rom-stack"
```

## 2. Converge Fleet Node

```bash
ansible-playbook -i inventory.ini playbooks/deploy_stack.yml -e "stack=docker-rom-stack"
```
