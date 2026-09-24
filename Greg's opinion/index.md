# Ouroboros v7.4.4 — LLM agent documentation pack

**Pinned version:** `7.4.4` (razzant/ouroboros)  
**Audience:** LLM agents editing or extending this repo  
**Evidence:** upstream `docs/ARCHITECTURE.md` + `architecture/*.md`, `DOMAIN_MAP.md`, domain inventory, on-disk module paths

AGENT: read this when… starting any repo task. Follow the reading order below; do not invent modules, gates, or process roles.

## Reading order

| Order | File | Open when… | Purpose (one line) |
|------:|------|------------|--------------------|
| 0 | [00-agent-contract.md](00-agent-contract.md) | Before any edit or claim | How to use this pack: evidence rules, version pin, what not to invent |
| 1 | [01-what-runs-where.md](01-what-runs-where.md) | Locating PID lock, supervisor, workers, Presence, Host Service | Process topology, spawn/reap, state roots, ports/defaults |
| 2 | [02-runtime-spine.md](02-runtime-spine.md) | Tracing a request end-to-end | D18→D11→D08→D01 spine; D06/D07 siblings; D20 side-lane; Skills≠Widgets |
| 3 | [03-domain-map.md](03-domain-map.md) | Finding ownership for a path | All D01–D20: modules, prefixes, key decisions |
| 4 | [04-pipelines.md](04-pipelines.md) | Implementing or debugging a flow | Chat/task, review, delegation, cancel, skills, Presence, memory/evolution |
| 5 | [05-decision-gates.md](05-decision-gates.md) | Changing a load-bearing gate | Named gates: question, owner path, pass/fail, related domains |
| 6 | [06-extension-surface.md](06-extension-surface.md) | Adding product capability | Safe extend via D14/D19; what not to fork; ABI pointers |
| 7 | [07-invariants.md](07-invariants.md) | Before merge / review | Never-break rules with path citations |
| 8 | [08-path-index.md](08-path-index.md) | Jumping to a concrete file | Dense path → role table |
| 9 | [09-upstream-doc-map.md](09-upstream-doc-map.md) | Need deeper mechanism detail | Upstream architecture chapters → these files |

## Cross-links (quick)

- Topology answers (“who owns PID lock?”) → [01](01-what-runs-where.md)
- “Is review inside the LLM loop?” → [02](02-runtime-spine.md) (no — sibling executor)
- Gate checklist → [05](05-decision-gates.md)
- Ship a skill, not a loop fork → [06](06-extension-surface.md)

## Source pointers (upstream only)

- `docs/ARCHITECTURE.md` → chapter index
- `architecture/01-high-level-architecture.md` … `12-host-service-companions-and-chat-ids.md` (and remaining chapters named in the index)
- `DOMAIN_MAP.md` / `ouroboros/domains.toml` — module→domain SSOT (556 modules)
- `ouroboros/contracts/` — frozen ABI (D19)
