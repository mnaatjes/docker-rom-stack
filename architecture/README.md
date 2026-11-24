# Architecture Plane: docker-rom-stack

Authoritative engineering documentation and technical governance for `docker-rom-stack`, governed by the **Homelab Systems Lifecycle (HSL)** and the **Unified Process (UP)**.

---

## 1. Governance Taxonomy

```text
architecture/
├── adr/             # Immutable MADR 3.0 Architectural Decision Records
├── designs/         # Formal IEEE 1016 Software Design Documents (SDDs)
├── use-cases/       # RUP Cockburn Sea-Level Functional Use Cases
└── risk/            # Barry Boehm Master Risk Register (RE = P x I)
```

---

## 2. Document Registry

### Architectural Decision Records (`architecture/adr/`)
* [ADR 0001: ROM Stack Architecture, VPN Isolation, and Storage Pipeline](adr/0001_rom_stack_cross_stack_integration_and_storage_architecture.md)

### Software Design Documents (`architecture/designs/`)
* [SDD-001: ROM Stack Lifecycle, NAS Storage Allocation, and Ansible Automation](designs/0001_rom_stack_nas_storage_and_ansible_automation.md)
* [SDD-002: ROM Acquisition Pipeline, VPN Killswitch Isolation, and Cross-Stack Orchestration](designs/0002_rom_stack_cross_stack_pipeline_and_vpn_architecture.md)

### Functional Use Cases (`architecture/use-cases/`)
* [UC-01: Automated ROM Acquisition, BitTorrent Download, and Inode Ingest](use-cases/UC-01_acquire_and_seed_rom.md)
* [UC-02: Scrape Metadata, Index ROMs, and Serve Emulation Clients](use-cases/UC-02_scrape_and_serve_rom_library.md)

### Risk Management (`architecture/risk/`)
* [Master Risk Register (risk-list.md)](risk/risk-list.md)
