---
title: "Master Risk Register: docker-rom-stack"
tags: ["risk", "architecture", "boehm-up", "docker-rom-stack"]
created_at: "2026-11-24"
last_updated_at: "2026-11-24"
---

# Master Risk Register: docker-rom-stack

This document constitutes the authoritative Master Risk Register for `docker-rom-stack`, governed by Barry Boehm's Software Risk Management Taxonomy and the Unified Process Inception Discipline.

---

## 1. Risk Evaluation Methodology

Risk exposure is computed per the classical Barry Boehm metric:

$$RE = P \times I$$

* **Probability ($P$):** Integer scale `1` (Extremely Unlikely) to `5` (Near Certainty).
* **Impact ($I$):** Integer scale `1` (Negligible / Cosmetic) to `5` (Catastrophic / Complete Outage / Data Loss).
* **Risk Exposure ($RE$):** Product range `1` to `25`.

### Prioritization & Action Thresholds
* **Critical Exposure ($RE \ge 16$):** Architectural blocker requiring immediate mitigation or prototype prior to production operations.
* **Moderate Exposure ($9 \le RE \le 15$):** Active monitoring with explicit mitigation procedures defined during Elaboration.
* **Low Exposure ($RE < 9$):** Documented and managed via standard procedural verifications.

---

## 2. Active Risk Register

| Risk ID | Risk Description | Category | $P$ | $I$ | $RE$ | Status | Target Phase |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **RSK-01** | Public WAN IP leak during gaming tracker downloads | Security / Privacy | 3 | 5 | **15** | Mitigated | Inception / Elaboration |
| **RSK-02** | Stale NFS file handles from MergerFS inode mutation | Infrastructure / Storage | 3 | 4 | **12** | Mitigated | Elaboration |
| **RSK-03** | Port collision with existing Arr applications (`8787`) | Runtime / Orchestration | 4 | 3 | **12** | Mitigated | Inception |
| **RSK-04** | RAM exhaustion and OOM kill on `prd-seed-01.lan` | Resource / Hardware | 3 | 3 | **9** | Monitored | Construction |
| **RSK-05** | Rate-limiting and authentication failure on Twitch IGDB API | External Dependency | 3 | 2 | **6** | Monitored | Construction |

---

## 3. Risk Analysis & Mitigation Specifications

### RSK-01: Public WAN IP Leak During BitTorrent Transfers
* **Description:** Initiating peer-to-peer BitTorrent traffic directly from an unshielded container could expose the homelab's public ISP address to copyright monitoring swarms.
* **Mechanism:** If Gamarr or an embedded client attempts downloads without an enforced tunnel, packets route through the default Docker bridge directly to the host WAN interface.
* **Mitigation Strategy:**
  1. Ban all internal BitTorrent download engines within `docker-rom-stack`.
  2. Route all torrent traffic exclusively through `docker-torrent`'s Transmission daemon via RPC (`http://192.168.1.52:9091/transmission/rpc`).
  3. Ensure Transmission is bound directly to Gluetun (`network_mode: "service:gluetun"`) with iptables DROP policies enforced on disconnect.
  4. Route external indexer search requests from Prowlarr through Gluetun's local HTTP proxy (`:8888`).
* **Verification:** Run `curl -s --proxy http://192.168.1.52:8888 https://ipinfo.io/json` from the container to verify VPN public exit IP.

### RSK-02: Stale NFS File Handles from MergerFS Inode Mutation
* **Description:** Cross-device file moves or multi-drive rebalancing on `prd-nas-01.lan` can cause client file handles on `prd-seed-01.lan` to become stale (`ESTALE`), breaking hardlinks.
* **Mechanism:** When files are moved between underlying physical disks in the MergerFS union, inode numbers may change dynamically while cached in hypervisor NFS client mounts.
* **Mitigation Strategy:**
  1. Mandate `use_ino` in the upstream MergerFS mount configuration on `prd-nas-01.lan` ([ADR 0035](file:///home/michael/src/github.com/mnaatjes/homelab-ops/architecture/adr/0035_hybrid_media_acquisition_pipeline_architecture.md)).
  2. Direct all download staging (`/media/downloads/complete/roms`) and library storage (`/media/games`) to the identical root mount point (`/media`), guaranteeing atomic `link()` execution on the same underlying branch.
  3. Apply strict directory mask `0775` and ownership `999:110` (`seedbox-sys:media`).

### RSK-03: Port Collision with Existing Arr Applications
* **Description:** Gamarr defaults to port `8787`, which directly collides with Readarr (`READARR_PORT=8787`) hosted on the same worker node (`prd-seed-01.lan`).
* **Mechanism:** Docker Compose engine fails to bind host port `8787:8787` if Readarr holds the active socket.
* **Mitigation Strategy:**
  1. Re-map Gamarr host ingress port to `8790` in `compose.yml` (`GAMARR_PORT:-8790`).
  2. Pin RomM web UI to host port `8082` (`ROMM_PORT:-8082`), avoiding conflicts with Jellyfin (`8096`) or OpenObserve (`5080`).
  3. Document all exposed ports in the central SSOT catalog (`group_vars/all/workloads/compose_stacks.yml`).

### RSK-04: RAM Exhaustion on Worker Host `prd-seed-01.lan`
* **Description:** Running MariaDB 11, RomM, and Gamarr alongside six existing Arr services and Transmission on a memory-constrained LXC container could trigger the Linux OOM killer.
* **Mechanism:** MariaDB buffer pools and RomM scanning processes consume high transient memory during bulk library ingestion.
* **Mitigation Strategy:**
  1. Configure MariaDB memory ceiling (`innodb_buffer_pool_size = 128M`, `innodb_log_buffer_size = 8M`) in container config.
  2. Apply Docker Compose memory reservation and limit constraints (`mem_limit: 512m` for RomM, `mem_limit: 256m` for MariaDB).
  3. Monitor host container memory via Prometheus/Node-Exporter (`prd-mon-02.lan`).

### RSK-05: Rate-Limiting and Authentication Failure on Twitch IGDB API
* **Description:** RomM depends on Twitch developer tokens for IGDB metadata scraping. Invalid credentials or API rate-limiting prevent box art hydration.
* **Mechanism:** Bulk scraping of retro game libraries generates hundreds of sequential metadata requests to Twitch OAuth endpoints.
* **Mitigation Strategy:**
  1. Store Twitch client credentials securely in Ansible Vault (`vault_romm.igdb_client_id`, `vault_romm.igdb_client_secret`).
  2. Rely on RomM's internal token caching to prevent redundant authentication round-trips.
  3. Support manual fallback matching via SteamGridDB and local image placement under `${DATA_DIR}/romm/resources/`.
