# 00 — Agent contract

AGENT: read this when… before any code edit, architecture claim, or PR description. Re-read if unsure whether a path/gate is real.

## Mission

Use this pack as the **operational map** for Ouroboros **v7.4.4**. Prefer path-cited facts over prose memory. When mechanism detail is missing here, open the cited upstream architecture chapter or module docstring — do not invent.

## Version pin

| Item | Value |
|------|-------|
| Product / tag | **v7.4.4** |
| Repo | `razzant/ouroboros` |
| Domain inventory | 20 domains, **556** tracked runtime modules (`DOMAIN_MAP.md`) |
| Default app root | `~/Ouroboros/` (`APP_ROOT` / `DATA_DIR` / `SETTINGS_PATH` env-overridable via `ouroboros/config.py`) |
| Default HTTP port | `8765` (`AGENT_SERVER_PORT` in `ouroboros/config.py`) |
| Host Service default | `127.0.0.1:${OUROBOROS_HOST_SERVICE_PORT:-8767}` (`architecture/12-host-service-companions-and-chat-ids.md`) |

If working tree `VERSION` ≠ `7.4.4`, stop and re-pin sources before trusting this pack.

## How to use the pack

1. **Orient** — [01-what-runs-where.md](01-what-runs-where.md) then [02-runtime-spine.md](02-runtime-spine.md).
2. **Own the change** — find domain in [03-domain-map.md](03-domain-map.md); prefer D14/D19 for product work ([06-extension-surface.md](06-extension-surface.md)).
3. **Trace the pipeline** — [04-pipelines.md](04-pipelines.md).
4. **Do not weaken gates** — [05-decision-gates.md](05-decision-gates.md) + [07-invariants.md](07-invariants.md).
5. **Cite paths** in commits/PRs: repo-relative (`launcher.py`, `supervisor/queue.py`, `ouroboros/loop.py`).

## Evidence / citation rules

| Do | Do not |
|----|--------|
| Cite concrete module paths and upstream chapters (`docs/ARCHITECTURE.md`, `architecture/05-supervisor-loop.md`, `DOMAIN_MAP.md`) | Invent modules, ports, state files, or gate outcomes |
| Quote decision names that appear in this pack / upstream | Collapse skill lifecycle stages “for ship speed” |
| Treat `DOMAIN_MAP.md` as module→domain SSOT | Move a module to another domain without updating `ouroboros/domains.toml` + regenerating the map |
| Prefer additive ABI changes under `ouroboros/contracts/` | Silent reshape of frozen contracts |
| Record seal / usage / cancel facts as durable state | Treat UI status or snapshot rows as second authorities |

## What not to invent

- A **second scheduler** beside `server.py::_run_supervisor` / `supervisor/*`
- Nesting **D06 review** or **D07 delegation** *inside* `run_llm_loop` (they are sibling executors)
- Equating **Skills catalogue** (`gateway/extensions`) with **Widgets** (`gateway/widgets`)
- Treating **Presence** as pooled Chat (it is a side-lane: bind→admit→run→deliver)
- **Cancel via status** (intent-then-custody only — see [07-invariants.md](07-invariants.md))
- **`$0.00` for unknown cost** (ledger null stays null)
- Fake Host Service / Presence / chat-id routes beyond `architecture/12-*.md` and `ouroboros/contracts/`

## Hard diagram rules (carry into any diagram or digest)

1. **D06** and **D07** = sibling executors of the main tool loop — share evidence/authority contracts; not nested in `run_llm_loop`.
2. **Skills catalogue ≠ Widgets live projection**.
3. **Presence** = side-lane, not Chat worker pool.

## Related

- Spine picture → [02-runtime-spine.md](02-runtime-spine.md)
- Index → [index.md](index.md)
