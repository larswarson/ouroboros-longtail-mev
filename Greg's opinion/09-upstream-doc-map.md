# 09 — Upstream documentation map

AGENT: read this when… a pack file is too shallow and mechanism detail is required. Open the upstream chapter; do not invent.

Pinned: **v7.4.4**. Entry: `docs/ARCHITECTURE.md`.

## Architecture book → this pack

| Upstream chapter | Path (repo) | Maps primarily to |
|------------------|-------------|-------------------|
| High-Level Architecture | `architecture/01-high-level-architecture.md` | [01](01-what-runs-where.md), [02](02-runtime-spine.md), [08](08-path-index.md) |
| Startup / Onboarding Flow | `architecture/02-startup-onboarding-flow.md` | [04](04-pipelines.md) §chat, [05](05-decision-gates.md) onboarding |
| Web UI Pages & Buttons | `architecture/03-web-ui-pages-and-buttons.md` | [04](04-pipelines.md), D11 notes in [03](03-domain-map.md) |
| Server API Endpoints | `architecture/04-server-api-endpoints.md` | [08](08-path-index.md) gateway rows, [04](04-pipelines.md) |
| Supervisor Loop | `architecture/05-supervisor-loop.md` | [01](01-what-runs-where.md), [02](02-runtime-spine.md), [05](05-decision-gates.md) assign/consciousness |
| Agent Core | `architecture/06-agent-core.md` | [02](02-runtime-spine.md), D01–D07 in [03](03-domain-map.md), [04](04-pipelines.md) |
| Configuration | `architecture/07-configuration.md` | D12 in [03](03-domain-map.md), [05](05-decision-gates.md) |
| Git / CI / Build | `architecture/08-git-branching-ci-and-build.md` | D10 in [03](03-domain-map.md), [04](04-pipelines.md) review→git |
| Shutdown & Process Cleanup | `architecture/09-shutdown-and-process-cleanup.md` | [01](01-what-runs-where.md), [07](07-invariants.md) |
| Key Invariants | `architecture/10-key-invariants.md` | [07](07-invariants.md) (authoritative numbered list) |
| Frozen Contracts v1 | `architecture/11-frozen-contracts-v1.md` | [06](06-extension-surface.md), D19 in [03](03-domain-map.md) |
| Host Service, Companions, Chat IDs | `architecture/12-host-service-companions-and-chat-ids.md` | [01](01-what-runs-where.md) Host Service, [04](04-pipelines.md) Presence |
| External Skills Layer | `architecture/13-external-skills-layer.md` | [06](06-extension-surface.md), skill gate in [05](05-decision-gates.md) |

> Some checkouts keep chapters under `docs/architecture/`; the index in `docs/ARCHITECTURE.md` is authoritative for relative links.

## Other upstream docs

| Doc | Use for |
|-----|---------|
| `DOMAIN_MAP.md` / `docs/DOMAIN_MAP.md` | Full module lists + dependency matrix → [03](03-domain-map.md) |
| `ouroboros/domains.toml` | SSOT for module→domain; regenerate map via `scripts/check_domains.py` |
| `docs/CREATING_SKILLS.md` | Skill authorship → [06](06-extension-surface.md) |
| `docs/CHECKLISTS.md` | Review / skill checklists |
| `docs/DEVELOPMENT.md` (+ `development/` if present) | Engineering handbook |
| `docs/DESIGN.md` / `DEPLOYMENT.md` | Design / deploy notes |
| `BIBLE.md` | Constitution invariants |
| `ouroboros/contracts/` | Live ABI (prefer over paraphrases) |

## How this pack relates

| This pack | Role vs upstream |
|-----------|------------------|
| [00](00-agent-contract.md)–[08](08-path-index.md) | Agent-oriented operational digest pinned to v7.4.4 |
| Upstream `architecture/*.md` | Mechanism depth + WHY; win on conflict of detail |
| Module docstrings | Local SSOT for a single file’s contract |

**Conflict rule:** if this pack and an upstream chapter disagree on mechanism, **believe the upstream chapter + live module**, then fix the pack — do not “reconcile” by invention.

## Related

- Start → [index.md](index.md)
- Contract → [00-agent-contract.md](00-agent-contract.md)
