# Kimberly's opinion — Ouroboros agent pack (v7.4.7)

**Pinned version:** `7.4.7` (razzant/ouroboros)  
**Audience:** LLM agents editing or extending this repo  
**Evidence:** upstream `docs/ARCHITECTURE.md` + architecture chapters, `DOMAIN_MAP.md`, on-disk module paths

AGENT: read this when starting any repo task. Prefer path-cited claims; do not invent modules, gates, or process roles.

## Contents

| File | Open when… | Purpose |
|------|------------|---------|
| [how-it-works.md](how-it-works.md) | Need the end-to-end picture | Guided tour: topology, spine, 20 domains, pipelines, gates, invariants |
| [ouroboros-map.html](ouroboros-map.html) | Exploring modules interactively | Clickable domain/module map (open in a browser) |

## Source pointers (upstream only)

- `docs/ARCHITECTURE.md` — chapter index
- `architecture/*.md` — mechanism detail
- `DOMAIN_MAP.md` / `ouroboros/domains.toml` — module→domain SSOT
- `ouroboros/contracts/` — frozen ABI
