---
title: "SDD-002: ROM Acquisition Pipeline, VPN Killswitch Isolation, and Cross-Stack Orchestration"
status: "approved"
authors: ["@mnaatjes", "@antigravity"]
reviewers: ["@mnaatjes"]
created_at: "2026-11-24"
last_updated_at: "2026-11-24"
related_adrs: [
  "architecture/adr/0001_rom_stack_cross_stack_integration_and_storage_architecture.md",
  "architecture/adr/0025_docker_compose_application_lifecycle_and_deployment_architecture.md",
  "architecture/adr/0035_hybrid_media_acquisition_pipeline_architecture.md"
]
related_rfcs: []
tags: ["sdd", "architecture", "ieee-1016", "docker-rom-stack", "gluetun", "transmission", "romm", "gamarr"]
---

# SDD-002: ROM Acquisition Pipeline, VPN Killswitch Isolation, and Cross-Stack Orchestration

## 1. Executive Summary & Policy Distillation

### 1.1 Policy Context
This Software Design Document (SDD) establishes the authoritative technical blueprint for `docker-rom-stack`. Per [ADR 0001](file:///home/michael/src/github.com/mnaatjes/docker-rom-stack/architecture/adr/0001_rom_stack_cross_stack_integration_and_storage_architecture.md), the emulation automation subsystem is deployed as an independent container stack hosted on worker node `prd-seed-01.lan` (`192.168.1.52`, LXC 108 on hypervisor `prd-pve-02.lan`), co-located with `docker-torrent` and `docker-arr-suite`.

### 1.2 Objectives
1. **Automate Game Acquisition:** Interface Gamarr with Prowlarr (`:9696`) to discover and index console/computer ROM releases across Usenet and BitTorrent.
2. **Guarantee Privacy via VPN Killswitch:** Confine all BitTorrent peer traffic to Gluetun's WireGuard tunnel (`docker-torrent`) via Transmission RPC delegation.
3. **Prevent Storage Duplication:** Enforce atomic zero-copy hardlinks (`link()`) between download staging directories and library storage across the central NFS pool (`/mnt/nas/media`).
4. **Deliver Centralized Emulation:** Host RomM 3.7.2 and MariaDB 11.4 to provide in-browser WebAssembly emulation, box art scraping (IGDB), and save state synchronization.

### 1.3 Design Invariants & Guarantees
* **Invariant 1 (Zero-Leak BitTorrent Execution):** `docker-rom-stack` shall contain no internal BitTorrent client engines. All downloads are dispatched to Transmission over RPC (`:9091`), which is strictly bound to Gluetun's network namespace (`network_mode: "service:gluetun"`).
* **Invariant 2 (Cross-Volume Isolation Banned):** Both Gamarr and RomM shall mount the central NAS media root at `/media` or `/mnt/nas/media/games`. Staging and final libraries share the same physical filesystem branch, guaranteeing that file moves execute as $O(1)$ atomic hardlinks without data duplication.
* **Invariant 3 (Identity Consistency):** All container processes shall run with `PUID=999` and `PGID=110` (`seedbox-sys:media`), matching permissions across `prd-nas-01.lan` and hypervisor LXC mounts.
* **Invariant 4 (Port Uniqueness):** Gamarr binds host port `8790` (preventing collision with Readarr on `8787`), and RomM binds host port `8082`.

---

## 2. Architectural Blueprint & Component Models

### 2.1 Multi-Stack Component & Network Architecture

```mermaid
flowchart TD
    subgraph StorageNode ["Storage Host: prd-nas-01.lan (192.168.1.103)"]
        Pool["MergerFS Pool: /storage/media"]
        NFS["nfs-kernel-server (Port 2049)"]
        Pool --> NFS
    end

    subgraph HypervisorNode ["Hypervisor: prd-pve-02.lan (192.168.1.111)"]
        FstabNFS["NFS Mount: /mnt/nas/media"]
        LXCConf["LXC 108 Bind Mount: mp0 -> /mnt/nas/media"]
        FstabNFS --> LXCConf
    end

    subgraph SeedboxHost ["Worker Node: prd-seed-01.lan (192.168.1.52) - LXC 108"]
        HostFS["Local Mount: /mnt/nas/media"]

        subgraph TorrentStack ["docker-torrent"]
            Gluetun["gluetun (VPN Gateway)\nWireGuard Tunnel (Mullvad)\nHTTP Proxy: 8888\nKillswitch: iptables DROP"]
            Trans["transmission (BitTorrent)\nnetwork_mode: service:gluetun\nRPC Port: 9091\nStaging: /media/downloads"]
            Gluetun --- Trans
        end

        subgraph ArrStack ["docker-arr-suite"]
            Prowlarr["prowlarr:9696\nIndexer Sync & Torznab\nProxy: http://127.0.0.1:8888"]
        end

        subgraph RomStack ["docker-rom-stack"]
            Gamarr["gamarr:8790\nROM Automation\nRPC Client: Transmission\nTorznab Client: Prowlarr"]
            RomM["romm:8082\nLibrary Web UI & EmulatorJS\nMetadata Scraper (IGDB)"]
            MariaDB["romm-db:3306\nMariaDB 11.4 Backend"]
            MariaDB --- RomM
        end

        HostFS -->|/mnt/nas/media:/media| TorrentStack
        HostFS -->|/mnt/nas/media:/media| ArrStack
        HostFS -->|/mnt/nas/media:/media| RomStack

        Gamarr -->|Torznab API: :9696| Prowlarr
        Prowlarr -->|Scraper Proxy: :8888| Gluetun
        Gamarr -->|RPC Dispatch: :9091| Trans
        RomM -->|Scans: /media/games| HostFS
    end

    NFS --> FstabNFS
```

---

## 3. Data Storage & Filesystem Architecture

### 3.1 Directory Topology on Central NAS Pool (`prd-nas-01.lan`)

```text
/mnt/nas/media/
├── downloads/
│   ├── complete/
│   │   ├── movies/
│   │   ├── tv/
│   │   └── roms/               <-- Staging destination for Transmission payloads
│   └── incomplete/             <-- Active BitTorrent download chunks
└── games/                      <-- Clean ROM and disc image repository
    ├── ps2/                    <-- Sony PlayStation 2 ISOs / CHDs
    │   ├── Metal Gear Solid 2/
    │   └── THPS3/
    ├── snes/                   <-- Super Nintendo SFC / SMC ROMs
    ├── gba/                    <-- Game Boy Advance GBA ROMs
    └── saves/                  <-- Centralized RomM and emulator save states
```

### 3.2 Persistent State Mapping (HSL Compliance)
All mutable configuration files, relational databases, and static assets reside under the HSL persistent path `/opt/homelab/prod/data/docker-rom-stack/`:

| Directory | Volume Target | Host Path | Owner:Group | Mode |
| :--- | :--- | :--- | :--- | :--- |
| **Gamarr Config** | `/config` | `/opt/homelab/prod/data/docker-rom-stack/gamarr` | `999:110` | `0775` |
| **RomM Config** | `/romm/config` | `/opt/homelab/prod/data/docker-rom-stack/romm/config` | `999:110` | `0775` |
| **RomM Assets** | `/romm/resources`| `/opt/homelab/prod/data/docker-rom-stack/romm/resources` | `999:110` | `0775` |
| **MariaDB State** | `/var/lib/mysql` | `/opt/homelab/prod/data/docker-rom-stack/mysql` | `999:110` | `0750` |

---

## 4. Cross-Stack Dynamic Behavior & Workflows

### 4.1 Ingestion and Hardlink Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant GMR as Gamarr
    participant TRN as Transmission (via Gluetun)
    participant NFS as Central NAS (/mnt/nas/media)
    participant RMM as RomM

    GMR->>TRN: Add Torrent (Tag: roms, Path: /media/downloads/complete/roms)
    TRN->>NFS: Download blocks to /media/downloads/incomplete/
    TRN->>NFS: Finalize to /media/downloads/complete/roms/
    TRN-->>GMR: Download Complete Event (via RPC)
    GMR->>NFS: Inspect downloaded file structure
    GMR->>NFS: link(/media/downloads/complete/roms/game.iso, /media/games/ps2/game.iso)
    Note over GMR,NFS: Inode hardlink created instantaneously without copying bytes
    GMR->>RMM: Notify Library Refresh
    RMM->>NFS: Scan /romm/library/ps2/game.iso
    RMM->>RMM: Compute SHA1/CRC32 & Query IGDB
    RMM-->>RMM: Persist record to MariaDB
```

---

## 5. Operational Governance & Runbook Verification

### 5.1 Preflight & Deployment Verification
Prior to starting `docker-rom-stack`, the operator or Ansible control runner must verify:
1. `docker-torrent` is active and Gluetun displays `HEALTHY`:
   ```bash
   docker inspect --format='{{.State.Health.Status}}' gluetun
   ```
2. Transmission RPC responds with HTTP 401 (Authentication Required):
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://192.168.1.52:9091/transmission/rpc
   ```
3. Prowlarr web API responds on port `9696`:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://192.168.1.52:9696/api/v1/system/status
   ```

### 5.2 Hardlink Integrity Audit
Verify that imported games share the exact physical inode with the active seeding torrent:
```bash
ls -i "/mnt/nas/media/downloads/complete/roms/THPS3.iso" "/mnt/nas/media/games/ps2/THPS3.iso"
# Expected: Both outputs display identical inode integer numbers.
```
