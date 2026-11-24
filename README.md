# docker-rom-stack

ROM automation, metadata management, and in-browser emulation suite (Gamarr, RomM, MariaDB) integrated with Prowlarr indexing and Transmission/Gluetun VPN downloading.

Adheres to the **Homelab Systems Lifecycle (HSL)** and **ADR 0025** container deployment standards.

---

## 1. Stack Components & Ports

| Service | Container Image | Port Binding | Purpose |
| :--- | :--- | :--- | :--- |
| **RomM** | `ghcr.io/rommapp/romm:3.7.2` | `8082:8080` | ROM browser, IGDB metadata scraper, web emulator, save manager |
| **romm-db** | `mariadb:11.4` | Internal | Relational database backend for RomM state |
| **Gamarr** | `ghcr.io/gamarr/gamarr:latest` | `8790:8787` | Automated ROM download manager & organizer (`*arr` interface) |

---

## 2. Cross-Stack Integration Architecture

This stack operates on **`prd-seed-01.lan`** alongside `docker-arr-suite` and `docker-torrent`:

```text
┌─────────────────┐       Query / Torznab Sync        ┌──────────────────────────────────┐
│     Gamarr      ├──────────────────────────────────►│ docker-arr-suite: Prowlarr       │
│  (Port 8790)    │                                   │ (Port 9696 via Gluetun Proxy)    │
└────────┬────────┘                                   └──────────────────────────────────┘
         │
         │ Download RPC (Category: roms)
         ▼
┌──────────────────────────────────┐                  ┌──────────────────────────────────┐
│ docker-torrent: Transmission    │◄─ (Network Mode) ─┤ docker-torrent: Gluetun VPN      │
│ (Port 9091)                      │                   │ (WireGuard + Killswitch)         │
└────────┬─────────────────────────┘                  └──────────────────────────────────┘
         │
         │ Download Payloads to /media/downloads/complete/roms
         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Shared NFS Pool (/mnt/nas/media)                                                        │
│  ├── downloads/complete/roms/ (Transmission payload / continuous seeding)              │
│  └── games/<platform>/<title>/ (Gamarr atomic hardlink target)                         │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼ Scans & Indexes
                              ┌────────────────────┐
                              │        RomM        │
                              │    (Port 8082)     │
                              └────────────────────┘
```

* **VPN Killswitch Isolation:** Gamarr dispatches torrent downloads exclusively via Transmission RPC (`http://192.168.1.52:9091/transmission/rpc`). Transmission runs attached to Gluetun's network namespace (`network_mode: service:gluetun`), guaranteeing that torrent traffic is dropped if the VPN connection disconnects.
* **Tracker Query Privacy:** Prowlarr routes tracker search requests through Gluetun's local HTTP proxy (`http://192.168.1.52:8888`), shielding indexer requests behind the VPN endpoint IP.
* **Atomic Hardlinks:** Gamarr mounts `/mnt/nas/media` as `/media`, enabling zero-storage atomic hardlinks (`link()`) from `/media/downloads/complete/roms` to `/media/games/<platform>/<title>`, preserving continuous torrent seeding without consuming double disk space.

---

## 3. Directory Layout

```text
docker-rom-stack/
├── compose.yml              # Container orchestration specification
├── .env.example             # Environment variable contract
├── config/                  # Configuration templates
├── architecture/            # Unified Process engineering plane
│   ├── adr/                 # Architecture Decision Records
│   │   └── 0001_rom_stack_cross_stack_integration_and_storage_architecture.md
│   ├── designs/             # Software Design Documents
│   ├── use-cases/           # Use Case Specifications
│   └── risk/                # Boehm Risk Register
└── docs/                    # Diataxis operator documentation
    ├── tutorials/           # Step-by-step beginner guides
    ├── how-to/              # Problem-oriented runbooks
    ├── reference/           # Technical specifications
    └── explanation/         # High-level background discussions
```

---

## 4. Quick Start (Local Development)

```bash
# 1. Copy environment template
cp .env.example .env

# 2. Start container stack
docker compose up -d

# 3. View logs
docker compose logs -f
```

Stack will be available at `http://localhost:8082` (RomM) and `http://localhost:8790` (Gamarr).
