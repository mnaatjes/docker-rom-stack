---
title: "RomM Architecture, Filesystem Layout, and API Reference"
tags: ["romm", "reference", "api", "filesystem", "docker-rom-stack"]
created_at: "2026-11-24"
last_updated_at: "2026-11-24"
---

# RomM Architecture, Filesystem Layout, and API Reference

This document provides a comprehensive operational and architectural reference for RomM (v3.7.2), detailing filesystem bindings, container internals, authentication and CSRF mechanics, REST/WebSocket API endpoints, and metadata scraper integration.

---

## 1. Container Topology & Subsystem Overview

RomM operates as a multi-tier container application running within a single container image (`ghcr.io/rommapp/romm:3.7.2`):

```
+---------------------------------------------------------------------------------------+
| RomM Container (:8080 internal)                                                       |
|                                                                                       |
|  +--------------------+      +-----------------------------------------------------+  |
|  | Nginx (:8080)      | ---> | Gunicorn + Uvicorn Workers (:5000 / unix domain sk) |  |
|  | (Static Frontend & |      | (FastAPI Application Backend)                       |  |
|  |  Reverse Proxy)    |      +-----------------------------------------------------+  |
|  +--------------------+                                 |                             |
|            |                                            v                             |
|            |                        +-----------------------------------------------+ |
|            +----------------------> | Valkey / Redis (:6379)                        | |
|                                     | (Task Queue, Session Store, Cache)            | |
|                                     +-----------------------------------------------+ |
|                                                         ^                             |
|                               +-------------------------+--------------------+        |
|                               |                         |                    |        |
|                               v                         v                    v        |
|                     +------------------+      +------------------+ +----------------+ |
|                     | worker.py (rq)   |      | watcher.py       | | scheduler.py   | |
|                     | Background tasks |      | Inotify fs watch | | Cron scheduler | |
|                     +------------------+      +------------------+ +----------------+ |
+---------------------------------------------------------------------------------------+
                               |                                  |
                               v                                  v
                    +--------------------+             +--------------------+
                    | MariaDB (romm-db)  |             | Host / NAS Storage |
                    | Metadata & Schema  |             | /library & configs |
                    +--------------------+             +--------------------+
```

### Process Tree Inside Container
1. **PID 1 (`/init`):** Process orchestrator managing daemon processes.
2. **`valkey-server *:6379`:** In-memory key-value cache and Redis Queue (RQ) broker.
3. **`gunicorn` (master + 2 Uvicorn workers):** FastAPI REST backend serving on `0.0.0.0:5000` and `/tmp/gunicorn.sock`.
4. **`nginx` (master + 4 workers):** Web server binding to container port `8080`, hosting pre-compiled Vue frontend assets (`/assets`) and reverse-proxying `/api` and `/ws` to Gunicorn and Valkey.
5. **`worker.py`:** Python `rq` worker consuming asynchronous background tasks (`high`, `default`, `low` queues).
6. **`watcher.py`:** Filesystem event monitor watching `/library/roms` using inotify to schedule scans automatically on change.
7. **`scheduler.py`:** Periodic task executor (scheduled library rescans, Nintendo Switch TitleDB updates).

---

## 2. Filesystem Mount Standards & Directory Structure

RomM internally anchors its operational directories directly under the root filesystem and `/romm`.

### Governing Volume Mappings

| Host / NAS Path | Container Target Path | Access Mode | Purpose |
| :--- | :--- | :--- | :--- |
| `${DATA_DIR}/romm/resources` | `/romm/resources` | Read-Write | Downloaded box art, screenshots, logos, and generated asset caches. |
| `${DATA_DIR}/romm/redis_data` | `/redis_data` | Read-Write | Persistent Valkey/Redis snapshot (`dump.rdb`). |
| `${DATA_DIR}/romm/config` | `/romm/config` | Read-Write | Application configuration overrides (`config.yml`). |
| `${MEDIA_DIR}/games` | `/library` | Read-Write | Root ROM and BIOS library managed by RomM and Gamarr. |
| `${MEDIA_DIR}/games/saves` | `/romm/saves` | Read-Write | In-browser emulator save states and battery RAM saves. |
| `./patches/igdb_handler.py` | `/backend/handler/metadata/igdb_handler.py` | Read-Only | Runtime hotfix mounting (guards against IGDB `KeyError: 'rating'`). |

### Critical Invariant: Structure A vs. Structure B
RomM v3.7.2 defines `LIBRARY_BASE_PATH = "/library"` in its core backend configuration.
* **Structure A (Standard across HSL):** Mount host games folder directly to `/library`. RomM expects child directories:
  * `/library/roms/<platform_fs_slug>/<rom_files>` (e.g. `/library/roms/ps2/Metal Gear Solid 2.iso`)
  * `/library/bios/<platform_fs_slug>/<firmware_files>` (e.g. `/library/bios/ps2/scph39001.bin`)
* **Anti-Pattern:** Mounting to `/romm/library` causes RomM's watcher and scanner to report 0 platforms found, because RomM does not look inside `/romm/library` unless explicitly overridden via `ROMM_BASE_PATH`.

---

## 3. Authentication, CSRF, and Session Mechanics

RomM implements a dual authentication architecture:
1. **OAuth2 Password Grant (Bearer Tokens):** Designed for programmatic API consumers, mobile clients, and automated scripts.
2. **Session Cookies (Double-Submit CSRF):** Designed for in-browser single-page application (SPA) interactions.

### OAuth2 Token Endpoint
* **Method & URL:** `POST /api/token`
* **Content-Type:** `application/x-www-form-urlencoded`
* **Exempt from CSRF:** Yes (explicitly exempted via regex `^/api/token.*`).
* **Required Parameters:**
  * `grant_type`: Must be `password` (or `refresh_token`).
  * `username`: RomM username (e.g., `gemini`).
  * `password`: RomM user password.
  * `scope`: Space-delimited string of OAuth scopes. **Important:** If `scope` is omitted or empty, the returned token will have zero scopes, causing all protected routes to return `403 Forbidden` (`{"detail": "Forbidden"}`).

#### Canonical OAuth Scopes
* `me.read`, `me.write`: User profile inspection and update.
* `platforms.read`, `platforms.write`: Querying and creating platform definitions.
* `roms.read`, `roms.write`: Querying, downloading, editing, or deleting game entries.
* `firmware.read`, `firmware.write`: Cataloging and managing BIOS firmware.
* `assets.read`, `assets.write`: Uploading and modifying box art.
* `collections.read`, `collections.write`: Managing user game collections.
* `users.read`, `users.write`: Administrative user management.
* `tasks.run`: Executing scheduled and on-demand maintenance tasks.

#### Token Retrieval Example
```bash
TOKEN=$(curl -s -X POST \
  -d "grant_type=password&username=gemini&password=YOUR_PASSWORD&scope=me.read%20me.write%20roms.read%20roms.write%20platforms.read%20platforms.write%20tasks.run" \
  http://192.168.1.52:8082/api/token | jq -r .access_token)
```

### Starlette CSRF & Session Cookies
Endpoints protected by `CustomCSRFMiddleware` (`starlette-csrf`) mandate the Double Submit Cookie technique:
1. Client requests any GET endpoint (e.g., `GET /api/heartbeat` or `GET /`).
2. Server issues cookie `romm_csrftoken`.
3. State-changing requests (`POST`, `PUT`, `DELETE`) initiated without an OAuth2 Bearer token must pass:
   * Cookie: `romm_csrftoken=<token>`
   * Header: `X-CSRFToken: <token>`

---

## 4. REST API Endpoint Reference

All REST endpoints are rooted at `/api`.

### Platform Management

#### `GET /api/platforms`
Retrieves all registered platforms, including ROM counts and associated firmware files.
* **Scope Required:** `platforms.read`
* **Sample Response:**
```json
[
  {
    "id": 1,
    "slug": "ps2",
    "fs_slug": "ps2",
    "name": "PlayStation 2",
    "custom_name": "",
    "igdb_id": 8,
    "rom_count": 2,
    "firmware": [
      {
        "id": 11,
        "file_name": "scph39001.bin",
        "file_path": "bios/ps2",
        "file_size_bytes": 4194304,
        "is_verified": false
      }
    ]
  }
]
```

#### `POST /api/platforms`
Registers a new console platform manually.
* **Scope Required:** `platforms.write`
* **Request Body:**
```json
{
  "name": "PlayStation 2",
  "slug": "ps2",
  "fs_slug": "ps2",
  "igdb_id": 8
}
```

### ROM Catalog & Details

#### `GET /api/roms`
Lists indexed ROMs across the entire library with pagination, sorting, and platform filtering.
* **Scope Required:** `roms.read`
* **Query Parameters:**
  * `platform_id`: Filter by integer platform ID.
  * `platform_slug`: Filter by platform filesystem slug (e.g. `ps2`).
  * `search`: Fuzzy title query string.
  * `page_size`: Number of items per page.
  * `page`: Page index (1-based).
* **Sample Output:**
```json
[
  {
    "id": 1,
    "name": "Metal Gear Solid 2: Sons of Liberty",
    "slug": "metal-gear-solid-2-sons-of-liberty",
    "platform_slug": "ps2",
    "platform_name": "PlayStation 2",
    "igdb_id": 376,
    "file_name": "Metal Gear Solid 2 - Sons of Liberty (USA).iso",
    "file_size_bytes": 4443725824,
    "first_release_date": 1005609600,
    "total_rating": 89.43
  }
]
```

#### `GET /api/roms/{id}`
Returns complete enriched metadata for an individual ROM, including summary, involved companies, genres, franchises, screenshots, and artwork URLs.

### System & Health

#### `GET /api/heartbeat`
Returns real-time system capability states, watcher/scheduler statuses, and enabled metadata providers without requiring authentication.

---

## 5. Library Scanning Architecture (REST & WebSockets)

Library scanning in RomM can be executed through two distinct channels:

### Channel A: WebSocket Socket.IO (UI Flow)
The web frontend interacts with RomM via Socket.IO at `/ws/socket.io`:
1. Client establishes WebSocket connection.
2. Emits event `scan` with payload:
   ```json
   {
     "type": "quick",
     "platform_ids": [1]
   }
   ```
3. Server spawns an asynchronous job via Redis Queue (`rq`).
4. Server streams progress events:
   * `scan:scanning_platform`: Emits current platform slug.
   * `scan:scanning_rom`: Emits current ROM file being inspected.
   * `scan:done`: Emits final summary stats (`ScanStats`).
   * `scan:done_ko`: Emitted if an unhandled exception terminates the scan.

### Channel B: Backend Python Execution (CLI / Automation Flow)
In headless automation environments where WebSocket communication is cumbersome, a scan can be invoked directly inside the container virtual environment:
```bash
docker exec romm /src/.venv/bin/python3 -c "
import sys, asyncio
sys.path.append('/backend')
from endpoints.sockets.scan import scan_platforms
from handler.scan_handler import ScanType

asyncio.run(scan_platforms(platform_ids=[1], scan_type=ScanType.QUICK))
"
```

---

## 6. Metadata Scraper Integration (Twitch IGDB)

RomM enriches game records using Twitch IGDB API v4.

### Scraper Lifecycle
1. **Twitch App Token Acquisition:**
   * Uses `IGDB_CLIENT_ID` and `IGDB_CLIENT_SECRET`.
   * Sends `POST https://id.twitch.tv/oauth2/token?grant_type=client_credentials`.
   * Caches token in Redis until expiration.
2. **Platform Mapping:**
   * Platform record must match an IGDB platform ID (e.g. `igdb_id: 8` for Sony PlayStation 2).
3. **ROM Identification:**
   * RomM extracts game serials (e.g. `SCUS-97328` via `SONY_SERIAL_REGEX`) or sanitizes the filename.
   * Queries IGDB `/v4/games` endpoint with `search "<title>"; fields <GAMES_FIELDS>; where platforms=[<id>];`.
4. **Artwork & Asset Downloads:**
   * Downloads high-resolution box art (`t_1080p`) and screenshots directly to `/romm/resources/roms/<platform_id>/<rom_id>/`.
