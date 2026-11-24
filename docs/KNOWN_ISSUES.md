---
title: "Known Issue: RomM 3.7.2 IGDB KeyError 'rating' During Metadata Scan"
tags: ["known-issue", "romm", "igdb", "docker-rom-stack"]
created_at: "2026-11-24"
last_updated_at: "2026-11-24"
---

# Known Issue: RomM 3.7.2 IGDB KeyError 'rating' During Metadata Scan

## Context & Symptoms
When running RomM (`ghcr.io/rommapp/romm:3.7.2`) with Twitch IGDB metadata enabled, scanning consoles with certain titles (e.g., *Metal Gear Solid 2: Sons of Liberty* on PS2) terminates prematurely with:
```
ERROR: [RomM][scan] 'rating'
```
This halts platform ROM cataloging, leaving newly added ROMs unindexed in the web UI.

## Root Cause
In RomM 3.7.2, `extract_metadata_from_igdb_rom()` in `/backend/handler/metadata/igdb_handler.py` assumes every item inside `rom.get("age_ratings", [])` contains the key `"rating"`:
```python
"age_ratings": [
    IGDB_AGE_RATINGS[r["rating"]]
    for r in rom.get("age_ratings", [])
    if r["rating"] in IGDB_AGE_RATINGS
],
```
When IGDB returns age rating reference ID dictionaries without an expanded rating payload (e.g. `[{'id': 165959}, ...]`), accessing `r["rating"]` raises an unhandled `KeyError: 'rating'`.

## Upstream Status
* **Fixed In:** Upstream RomM refactored IGDB query schemas to `age_ratings.rating_category` and defensive extraction in RomM 4.0.0+ / 5.x.
* **Current Deployment Version:** RomM 3.7.2.
* **Upstream GitHub:** [rommapp/romm](https://github.com/rommapp/romm).

## Resolution (Option B: Bind-Mount Patch)
To maintain runtime immutability without altering the base container image or requiring immediate major version migration:
1. Extracted `igdb_handler.py` from RomM 3.7.2.
2. Hardened `extract_metadata_from_igdb_rom()` defensively:
   ```python
   "age_ratings": [
       IGDB_AGE_RATINGS[r["rating"]]
       for r in rom.get("age_ratings", [])
       if isinstance(r, dict) and r.get("rating") in IGDB_AGE_RATINGS
   ],
   ```
3. Stored the patched file under `patches/igdb_handler.py`.
4. Mounted the patch read-only into RomM via `compose.yml`:
   ```yaml
   volumes:
     - ./patches/igdb_handler.py:/backend/handler/metadata/igdb_handler.py:ro
   ```
