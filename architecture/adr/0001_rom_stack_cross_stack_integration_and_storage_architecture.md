---
title: "ADR 0001: ROM Stack Architecture, VPN Isolation, and Storage Pipeline"
status: "accepted"
date: "2026-11-24"
deciders: ["@mnaatjes", "@antigravity"]
consulted: ["@engineering-all"]
informed: ["@engineering-all"]
tags: ["adr", "architecture", "roms", "romm", "gamarr", "gluetun", "transmission", "prowlarr", "nfs", "storage"]
created_at: "2026-11-24"
last_updated_at: "2026-11-24"
---

# ADR 0001: ROM Stack Architecture, VPN Isolation, and Storage Pipeline

## 1. Context and Problem Statement

Automated emulation management requires a specialized pipeline to acquire, organize, catalog, and serve video game ROMs across historical console platforms (e.g., Sony PlayStation 2, Nintendo SNES, Sega Genesis). While media stacks (`docker-arr-suite` for movies, series, books, and music) and network downloaders (`docker-torrent`) are operational in the homelab fleet, retro gaming presents distinct requirements:

1. **Non-Standard Metadata & Release Schemas:** Video game ROM releases follow preservation taxonomies (No-Intro, Redump, TOSEC) rather than scene film/TV numbering. Standard `*arr` tools (Radarr, Sonarr) cannot index or parse these formats.
2. **Preservation of Network Anonymity (VPN & Killswitch):** Public tracker and indexer downloads for gaming ROMs carry significant privacy risks and must route strictly through an encrypted WireGuard tunnel equipped with an automated firewall killswitch.
3. **Storage Pipeline & Inode Hardlinks:** ROM files and disc images (PS2 `.iso` images ranging from 1 GB to 8 GB) must be transferred from staging download folders into organized platform libraries without incurring multi-gigabyte cross-filesystem copies, while continuing to seed where required.
4. **Unified In-Browser & Handheld Client Access:** The homelab operator requires a central catalog capable of IGDB metadata matching, browser-based emulation (EmulatorJS), and wireless save/state synchronization to handheld gaming consoles (e.g., Anbernic, Retroid Pocket, Steam Deck).

This record defines the architecture of `docker-rom-stack`, its host provisioning placement, cross-stack integration with `docker-torrent` and `docker-arr-suite`, and its filesystem storage layout.

---

## 2. Decision Drivers

* **Zero-Leak WireGuard Enforcement:** All peer-to-peer Bittorrent network traffic must be isolated behind Gluetun's iptables killswitch.
* **Service Decoupling & Reusability:** The ROM automation stack must reuse existing privacy infrastructure (`docker-torrent`) and tracker indexing infrastructure (`docker-arr-suite:prowlarr`) rather than duplicating VPN gateways or indexer sync daemons.
* **Single Source of Truth (SSOT) Compatibility:** Deployment configurations must comply with the Homelab Systems Lifecycle (HSL), Ansible `roles/compose_stack`, and the central catalog (`group_vars/all/workloads/compose_stacks.yml`).
* **Zero-Storage Atomic Hardlinking:** Download imports on the central NAS must use native POSIX hardlinks across shared filesystem boundaries.
* **Deterministic UID/GID Alignment:** All container tasks must execute as `PUID=999` and `PGID=110` (`seedbox-sys:media`) to maintain read/write parity across NFS exports from `prd-nas-01.lan`.

---

## 3. Considered Options

* **Option A: Monolithic Embedded Stack.** Bundle a dedicated VPN client (Gluetun), torrent downloader, indexer, and ROM manager inside `docker-rom-stack`.
  * *Disadvantages:* Severe resource waste, multiple redundant VPN tunnels against external provider limits, separate torrent queues, and redundant indexer configurations.
* **Option B: Cross-Stack Decoupled Orchestration on `prd-seed-01.lan` (Accepted).** Deploy `docker-rom-stack` (Gamarr, RomM, MariaDB) as a distinct compose stack on the existing torrent/arr node (`prd-seed-01.lan`), interfacing over host localhost network ports to Transmission and Prowlarr while sharing the underlying NFS mount (`/mnt/nas/media`).
* **Option C: Centralized NAS Execution.** Deploy RomM and download utilities directly on bare-metal storage node `prd-nas-01.lan`.
  * *Disadvantages:* Violates HSL policy barring container execution on bare-metal storage appliances; risks stability of the MergerFS pool.

---

## 4. Decision Outcome

Chosen option: **Option B: Cross-Stack Decoupled Orchestration on `prd-seed-01.lan`**.

### 4.1 Target Host Provisioning

`docker-rom-stack` is assigned to target host **`prd-seed-01.lan`** (`192.168.1.52`, Proxmox LXC Container ID `108`).

* **Rationale:** `prd-seed-01.lan` already hosts both `docker-torrent` (Gluetun + Transmission) and `docker-arr-suite` (Prowlarr, Sonarr, Radarr, etc.). Colocating `docker-rom-stack` on this node provides:
  1. Ultra-low latency localhost/LAN RPC communication between Gamarr, Prowlarr (`:9696`), and Transmission (`:9091`).
  2. Direct mount of the shared NFS volume `/mnt/nas/media` already validated for UID `999:110` permissions.
  3. No cross-host network bottlenecks when performing hardlinks or file verifications.

### 4.2 Cross-Stack VPN Killswitch Integration (`docker-torrent`)

The ROM management pipeline ensures complete privacy without maintaining a second VPN connection by leveraging the existing `docker-torrent` stack:

1. **Bittorrent Download Traffic:**
   * Gamarr does not execute internal download clients or open raw peer sockets.
   * Instead, Gamarr communicates over HTTP RPC (`http://192.168.1.52:9091/transmission/rpc`) to Transmission.
   * In `docker-torrent`, Transmission is bound to Gluetun via `network_mode: "service:gluetun"`.
   * Gluetun enforces strict iptables firewall rules where all outbound traffic except encrypted WireGuard packets to Mullvad endpoints is unconditionally dropped (`killswitch`).
   * Therefore, all BitTorrent download traffic initiated by Gamarr is physically impossible to route outside the VPN tunnel.
2. **External Indexer / Tracker Search Traffic:**
   * Gamarr performs game search queries through Prowlarr (`http://192.168.1.52:9696`).
   * Prowlarr routes its external indexer scrapers through Gluetun's built-in HTTP proxy (`http://192.168.1.52:8888`), ensuring that search requests to gaming trackers remain masked behind the VPN IP.

### 4.3 Transmission RPC & Category Mapping

To prevent collisions between gaming downloads and standard video/music media, Transmission employs isolated category directories:

* **Category Declaration:** Gamarr is assigned download category `roms`.
* **Staging Directory:** Transmission stores active downloads at `/media/downloads/incomplete` and completed downloads at `/media/downloads/complete/roms/`.
* **Seeding Retention:** After download completion, Transmission retains the downloaded payloads in `/media/downloads/complete/roms/` to maintain seeding ratios.

### 4.4 Filesystem Storage Architecture & Download Destination

All persistent data is segmented between local container state and the shared central storage array:

```text
/mnt/nas/media/ (Central MergerFS Storage Pool on prd-nas-01.lan)
├── downloads/
│   ├── complete/
│   │   ├── movies/
│   │   ├── tv/
│   │   └── roms/                   <-- Staging directory for Transmission payloads
│   └── incomplete/                 <-- Active BitTorrent chunks
└── games/                          <-- Persistent Game Library
    ├── ps2/                        <-- PlayStation 2 disc images (.iso, .chd)
    │   ├── Metal Gear Solid 2/
    │   └── THPS3/
    ├── snes/                       <-- Super Nintendo ROMs (.sfc)
    ├── gba/                        <-- Game Boy Advance ROMs (.gba)
    └── saves/                      <-- RomM centralized save states
```

1. **Atomic Inode Hardlinking:**
   * In `compose.yml`, Gamarr mounts `${MEDIA_DIR:-/mnt/nas/media}:/media`.
   * Because both `/media/downloads/complete/roms` and `/media/games` exist on the identical underlying filesystem mount (`/mnt/nas/media`), Gamarr executes an atomic `link()` operation.
   * File data blocks are not copied; a second directory inode is created in `/mnt/nas/media/games/<platform>/<title>/`. Seeding continues unhindered in the downloads folder while RomM consumes the library folder.
2. **RomM Presentation Mount:**
   * RomM mounts `${MEDIA_DIR:-/mnt/nas/media}/games` directly into its container at `/romm/library`.
   * RomM watches `/romm/library`, queries the IGDB API for artwork and descriptions, stores metadata in the local `romm-db` (MariaDB 11.4) instance, and exposes games for browser play and client download.
3. **Application State Mounts (HSL Compliant):**
   * Stack configuration and persistent databases reside under `/opt/homelab/prod/data/docker-rom-stack/`:
     * `gamarr/`: Gamarr configuration and application database.
     * `romm/config/`: RomM YAML configuration and session keys.
     * `romm/resources/`: Downloaded box art, manuals, and screenshots.
     * `mysql/`: MariaDB relational storage for RomM.

---

## 5. Consequences

### Positive
* **Unified Privacy Protection:** Reuses the existing Gluetun killswitch without incurring additional VPN subscriptions or tunnel overhead.
* **Zero Storage Duplication:** Native hardlinking permits continuous torrent seeding without consuming additional gigabytes on the NAS pool.
* **Preservation of Core Principles:** Complies with HSL by maintaining separation between application containers and bare-metal storage nodes.
* **Automated & Manual Flexibility:** Operators can automate ROM acquisition via Gamarr or manually drop homebrew dumps into `/mnt/nas/media/games/` for instant RomM indexing.

### Negative
* **Resource Contention on `prd-seed-01.lan`:** Running MariaDB and RomM alongside six Arr applications and Transmission increases memory utilization by approximately 1.2 GB RAM.
* **Third-Party API Dependency:** RomM artwork scraping depends on active Twitch/IGDB API developer credentials.

### Neutral
* Prowlarr indexers must be tagged with game tracker categories (Console/1000, PC/4000) for Gamarr synchronization.

---

## 6. Procedure & Implementation

1. **Repository Generation:** Scaffold `docker-rom-stack` using canonical template `template-docker-stack`.
2. **Service Definition:** Author `compose.yml` declaring `romm-db` (MariaDB 11.4), `romm` (RomM 3.7.2), and `gamarr` (latest).
3. **SSOT Registration:** Add `docker-rom-stack` entry into `homelab-ops/group_vars/all/workloads/compose_stacks.yml` targeting `prd-seed-01.lan`.
4. **Ansible Deployment:** Execute `playbooks/deploy_stack.yml --extra-vars "stack_name=docker-rom-stack"`.
5. **Pipeline Integration:**
   * Configure Transmission in Gamarr settings (`host: 192.168.1.52`, `port: 9091`, `category: roms`).
   * Add Gamarr sync profile in Prowlarr (`host: 192.168.1.52:8790`).
   * Authenticate RomM with Twitch IGDB Client ID/Secret.
