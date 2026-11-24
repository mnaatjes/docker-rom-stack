# Master Implementation Plan: docker-rom-stack

This document establishes the authoritative, phased master implementation plan and checklist for deploying `docker-rom-stack`, codifying Ansible orchestration across the homelab fleet, and validating the end-to-end emulation streaming pipeline.

Governed by:
* [ADR 0001: ROM Stack Architecture, VPN Isolation, and Storage Pipeline](architecture/adr/0001_rom_stack_cross_stack_integration_and_storage_architecture.md)
* [SDD-001: ROM Stack Lifecycle, NAS Storage Allocation, and Ansible Automation](architecture/designs/0001_rom_stack_nas_storage_and_ansible_automation.md)
* [SDD-002: ROM Acquisition Pipeline, VPN Killswitch Isolation, and Cross-Stack Orchestration](architecture/designs/0002_rom_stack_cross_stack_pipeline_and_vpn_architecture.md)
* [Master Risk Register (risk-list.md)](architecture/risk/risk-list.md)

---

## Operational Execution Constraints

> [!IMPORTANT]
> **Ansible Execution Boundary:** All Ansible roles, inventory definitions, and variable declarations are authored strictly within `~/src/github.com/mnaatjes/homelab-ops/`. **NO live execution commands (`ansible-playbook`, ad-hoc modules) shall be initiated by automated agents in that repository.** The agent will stage, commit, and notify the operator when playbooks are ready; the operator retains sole authority to merge, deploy, and execute live Ansible routines.

---

## Phase 1: Repository Finalization & Governance Alignment (`docker-rom-stack`)

- [x] **Repository Scaffolding:** Instantiate project structure via `template-docker-stack` (Copier).
- [x] **Service Orchestration Specification:** Author `compose.yml` declaring `romm` (`:8082`), `romm-db` (`mariadb:11.4`), and `gamarr` (`:8790`).
- [x] **Environment Contract:** Author `.env.example` declaring `PUID=999`, `PGID=110`, persistent data paths, and service endpoints.
- [x] **Inception Artifacts:** Author Barry Boehm Master Risk Register (`architecture/risk/risk-list.md`) and Cockburn Sea-Level Use Cases (`UC-01`, `UC-02`).
- [x] **Elaboration Artifacts:** Author MADR 3.0 decision record (`ADR 0001`) and IEEE 1016 design blueprints (`SDD-001`, `SDD-002`).
- [ ] **Diátaxis Documentation Plane:**
  - [ ] Populate `docs/how-to/deploy_locally.md` with local development commands.
  - [ ] Populate `docs/reference/environment_variables.md` with exhaustive `.env` parameter dictionary.
  - [ ] Populate `docs/explanation/architecture_overview.md` cross-referencing SDD-001 and SDD-002.
- [ ] **Commit & Remote Synchronization:** Commit all finalized documentation and push to GitHub `origin/main`.

---

## Phase 2: Ansible SSOT Catalog & Role Codification (`homelab-ops`)

*Development strictly in `~/src/github.com/mnaatjes/homelab-ops/` on dedicated branch `feat/rom-stack-deployment`.*

- [x] **Branch Initialization:** Checkout new working branch `git checkout -b feat/rom-stack-deployment`.
- [x] **SSOT Workload Registration:**
  - [x] Add `docker-rom-stack` declaration to `group_vars/all/workloads/compose_stacks.yml`:
    - Target host: `prd-seed-01.lan`
    - Ingress ports: `[8082, 8790]`
    - PUID: `999`, PGID: `110`
    - Volume mappings: `/mnt/nas/media` and `/opt/homelab/prod/data/docker-rom-stack`
    - Health check: `http://192.168.1.52:8082/api/health`
    - Backup retention: 5 snapshots
- [x] **Ansible Vault Declarations:**
  - [x] Add encrypted credential templates in `group_vars/all/vault.yml`:
    - `vault_romm.auth_secret_key`
    - `vault_romm.db_root_passwd`
    - `vault_romm.db_passwd`
    - `vault_romm.igdb_client_id`
    - `vault_romm.igdb_client_secret`
    - `vault_romm.steamgriddb_api_key`
- [x] **NAS Storage Allocation Role (`roles/nfs_exports` or dedicated):**
  - [x] Ensure `/storage/media/games/{roms,bios,saves}` directory structure is codified.
  - [x] Codify declarative export rule for `/storage/media/games` with `all_squash,anonuid=999,anongid=110`.
- [x] **Workstation Client Mount Role (`roles/workstation_emulation`):**
  - [x] Codify persistent NFSv4 mount task targeting `192.168.1.103:/storage/media/games` -> `/home/michael/Emulation`.
  - [x] Codify Flatpak sandbox filesystem override: `flatpak override --filesystem=/home/michael/Emulation net.pcsx2.PCSX2`.
- [x] **Operator Handoff Gate:** Notify operator that code changes are staged and ready for review/merge.
- [x] **PR Review & Integration:** PR #124 merged, `main` fast-forwarded, and deployed to `/opt/homelab/prod/ops/`.

---

## Phase 3: NAS Storage & NFS Export Allocation (`prd-nas-01.lan`)

*To be applied by operator via Ansible or manual runbook.*

- [ ] **Directory Provisioning:** Create underlying storage skeleton on `prd-nas-01.lan`:
  ```bash
  mkdir -p /storage/media/games/{roms/ps2,roms/snes,roms/gba,bios,saves}
  mkdir -p /storage/media/downloads/complete/roms
  chown -R 999:110 /storage/media/games /storage/media/downloads/complete/roms
  chmod -R 775 /storage/media/games /storage/media/downloads/complete/roms
  ```
- [ ] **Firmware Migration:** Move discovered PS2 BIOS files from `/storage/media/audio/PCSX2/ps2_bios/` into `/storage/media/games/bios/ps2/`.
- [ ] **Game ISO Migration:** Move `THPS3.iso` and `Metal Gear Solid 2.iso` from `/storage/media/audio/PCSX2/` into `/storage/media/games/roms/ps2/`.
- [ ] **Export Refresh:** Verify NFS export rules in `/etc/exports` and execute `exportfs -ra`.
- [ ] **NFS Validation:** Verify exports visible on network: `showmount -e 192.168.1.103`.

---

## Phase 4: Container Stack Deployment & Convergence (`prd-seed-01.lan`)

*Executed by operator via `ansible-playbook playbooks/deploy_stack.yml -e stack_name=docker-rom-stack`.*

- [ ] **Preflight Verification:**
  - [ ] Verify `docker-torrent` is active and Gluetun reports `healthy`.
  - [ ] Verify `docker-arr-suite:prowlarr` is accessible on `:9696`.
  - [ ] Verify NFS mount `/mnt/nas/media` is active and writable by UID `999:110`.
- [ ] **Stack Deployment:**
  - [ ] Ansible synchronizes repository code to `/opt/homelab/prod/compose/docker-rom-stack/`.
  - [ ] Ansible renders production `.env` with vaulted secrets.
  - [ ] Ansible provisions persistent directories under `/opt/homelab/prod/data/docker-rom-stack/`.
  - [ ] Docker Compose pulls images and converges containers (`romm-db`, `romm`, `gamarr`).
- [ ] **Health Check Verification:**
  - [ ] Confirm `romm-db` passes internal healthcheck.
  - [ ] Confirm `romm` responds on `http://192.168.1.52:8082`.
  - [ ] Confirm `gamarr` responds on `http://192.168.1.52:8790`.

---

## Phase 5: Pipeline Integration & Cross-Stack Verification

- [ ] **Transmission Integration:**
  - [ ] In Gamarr settings, register Download Client:
    - Type: Transmission
    - Host: `192.168.1.52`
    - Port: `9091`
    - Category: `roms`
    - Directory: `/media/downloads/complete/roms`
  - [ ] Test RPC connection and verify authentication.
- [ ] **Prowlarr Torznab Integration:**
  - [ ] In Prowlarr, configure Gamarr sync profile or add Torznab category mapping (Console/1000, PC/4000).
  - [ ] Verify search queries route through Gluetun HTTP proxy (`192.168.1.52:8888`).
- [ ] **RomM Configuration:**
  - [ ] Complete initial setup wizard at `http://192.168.1.52:8082`.
  - [ ] Authenticate Twitch IGDB API credentials.
  - [ ] Add library platform `Sony PlayStation 2` pointing to `/romm/library/roms/ps2`.
  - [ ] Trigger manual scan; verify `THPS3.iso` and `Metal Gear Solid 2.iso` populate with box art and metadata.
- [ ] **Hardlink Verification:**
  - [ ] Trigger a test download via Gamarr.
  - [ ] Verify downloaded file in `/mnt/nas/media/downloads/complete/roms` and hardlinked file in `/mnt/nas/media/games/roms/ps2/` share identical inode integers (`ls -i`).

---

## Phase 6: Client Workstation & Emulation Streaming Setup (`michael-desktop`)

- [ ] **Client Mount Configuration:**
  - [ ] Execute client playbook or add mount entry in `/etc/fstab`:
    ```ini
    192.168.1.103:/storage/media/games /home/michael/Emulation nfs4 _netdev,auto,nofail,soft,intr,rsize=1048576,wsize=1048576 0 0
    ```
  - [ ] Verify mount access: `ls -la /home/michael/Emulation/roms/ps2`.
- [ ] **PCSX2 Flatpak Integration:**
  - [ ] Apply Flatpak permission override:
    ```bash
    flatpak override --user --filesystem=/home/michael/Emulation net.pcsx2.PCSX2
    ```
  - [ ] Open PCSX2; configure BIOS directory to `/home/michael/Emulation/bios/ps2`.
  - [ ] Add game search directory: `/home/michael/Emulation/roms/ps2`.
  - [ ] Verify game list populates and launches successfully with 60 FPS rendering.
- [ ] **Steam Remote Play / Steam Link Validation:**
  - [ ] Add PCSX2 non-Steam shortcut to Steam desktop client.
  - [ ] Connect tablet running Steam Link app to `michael-desktop`.
  - [ ] Launch `THPS3` from tablet; verify 60 FPS video stream, audio clarity, and controller input mapping.

---

## Quality & Acceptance Test Matrix

| ID | Phase | Target System | Verification Command / Procedure | Acceptance Criteria |
| :--- | :--- | :--- | :--- | :--- |
| **TC-01** | Phase 1 | `docker-rom-stack` | `git status && git log -n 1` | Clean working tree; all ADR/SDD docs committed. |
| **TC-02** | Phase 2 | `homelab-ops` | `ansible-lint` / YAML validation | Valid schema in `compose_stacks.yml` and `vault.yml`. |
| **TC-03** | Phase 3 | `prd-nas-01.lan` | `showmount -e 192.168.1.103` | `/storage/media/games` exported to subnet. |
| **TC-04** | Phase 4 | `prd-seed-01.lan` | `curl -s -o /dev/null -w "%{http_code}" http://192.168.1.52:8082` | HTTP 200 returned from RomM web frontend. |
| **TC-05** | Phase 4 | `prd-seed-01.lan` | `curl -s -o /dev/null -w "%{http_code}" http://192.168.1.52:8790` | HTTP 200 returned from Gamarr web frontend. |
| **TC-06** | Phase 5 | `prd-seed-01.lan` | `ls -i /mnt/nas/media/downloads/... /mnt/nas/media/games/...` | Inode numbers match identically (zero storage copy). |
| **TC-07** | Phase 6 | `michael-desktop`| `mountpoint -q /home/michael/Emulation` | Exit code 0 (active NFS mount). |
| **TC-08** | Phase 6 | Steam Link / Tablet| Launch game from Steam Link UI | 60 FPS video stream, gamepad input recognized. |
