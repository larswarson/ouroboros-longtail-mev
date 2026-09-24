# 02 — Runtime spine

AGENT: read this when… tracing a request, drawing a diagram, or deciding whether a change belongs in the loop vs a sibling executor / side-lane.

Evidence: `architecture/01-high-level-architecture.md`, `architecture/05-supervisor-loop.md`, `architecture/06-agent-core.md`, domain inventory + `DOMAIN_MAP.md`.

## Canonical spine (D18 → D11 → D08 → D01)

```
launcher.py (D18)
  → bootstrap / platform_layer / launcher_server_reaper
  → spawn server.py
      → lifespan: settings → reconcile_projects → init_global_supervisor
         → Host Service / extensions / review-job reconcile
         → _start_supervisor_if_needed → _run_supervisor (D08 tick)
      → gateway (D11): HTTP/WS /api/tasks, chat, settings
         → Skills catalogue (gateway/extensions)
              ≠ Widgets live projection (gateway/widgets)
         → Presence settings ── side-lane ──► D20 admit→run→deliver
      → supervisor queue + workers (D08)
          → worker: OuroborosAgent.handle_task (D01)
              → context assembly/fit (D03)
              → run_llm_loop
                  → LLM providers (D02) + model_send_seal / usage reserve (D16)
                  → tools registry/access (D04) + surfaces (D05)
                     ↳ safety/guards (D13) on shell/write paths
              → emit_task_results / finalization (D01) → task_results (D17)
              → post-task memory/evolution hooks (D15)
      ★ sibling executors (NOT nested inside run_llm_loop):
          → D06 review panels (advisory/plan/scope/commit/triad/acceptance)
          → D07 delegate + Claudexor (nanny / durable custody)
      → cancel/custody (D09) cross-cuts queue, workers, processes
ABI shapes: ouroboros/contracts/* (D19)
CLI (D18): same gateway client; refuses non-root delegation_role
```

## Stage map

### 1. Launcher → server (D18)

- Modules: `launcher.py`, `launcher_bootstrap.py`, `launcher_server_reaper.py`, `platform_layer.py`
- Moves: PID lock, bundle→managed repo, spawn `server.py`, write `server_process.json`, poll port + `/api/health`
- Decisions: Panic vs Restart vs window-close; same-install reaper only /proc-proven strays

### 2. Server / gateway ingress (D11)

- Modules: `server.py` lifespan; `ouroboros/gateway/*`
- Moves: settings load → project reconcile → supervisor init → onboarding after gateway up → UI on authoritative port
- Decisions:
  - **Start supervisor?** provider structurally sufficient → else gateway-only (`_start_supervisor_if_needed`)
  - **Admit task?** `gateway/tasks` + D17 `workspace_admission` + queue lock
  - **Create child?** only `schedule_subagent` (D07); gateway rejects caller lineage labels

### 3. Supervisor tick → workers (D08)

- Modules: `supervisor/queue.py`, `workers.py`, `events*`, `worker_assignment.py`, `message_bus.py`
- Tick: liveness → drain events (bridge first) → deadlines/schedules/custody → **assign** → `queue_snapshot.json` → consciousness alarm
- Decisions:
  - **Assign now?** D17 `project_lease` — one top-level writer per `project_id`; subagent swarms exempt
  - **Repo writer gate?** closed around managed update (D10 ↔ D08)
  - Three consecutive loop failures clear readiness + notify owner

### 4. Agent loop → LLM → tools (D01 + D02/D03/D04/D05/D13/D16)

- Per round: build/fit context → seal send → `usage_accounting.reserve_attempt` → provider call → settle usage → tool execute (guards) → reclaim/budget/acceptance
- Decisions:
  - **Send to model?** seal mismatch → durable fact, **not** a second dispatch gate; identity re-check may refuse
  - **Pay for attempt?** reserve under ledger lock
  - **Tool allow?** D13 → D04 access → D05 handler
  - **Force final / soft-land?** budget, round limits, owner-stop drain, transport wait

### 5. Review / delegate — siblings (D06 / D07)

**Hard rule:** draw **outside** the `run_llm_loop` box. They share evidence / tool-authority contracts with the main loop but run as separate executors / nanny paths.

- D06: advisory / plan / scope / triad / acceptance / commit via `review_*`, `tools/plan_review*`, `commit_admission`, paid stamp at physical dispatch
- D07: `schedule_subagent`, `delegate_*`, Claudexor daemon; durable custody OWNED/FOREIGN/UNKNOWN; parent keeps project lease

### 6. Memory / UI / Presence

- D15: post-task synthesis, consolidator, consciousness (tick-only wake), evolution promote
- D17: task results / artifacts / leases
- D20: bindings → admission → `presence_runner` → receipt-backed delivery
- D11: WS/SSE/history/cost UI

## Skills catalogue ≠ Widgets

| Surface | Path | Nature |
|---------|------|--------|
| Skills catalogue | `ouroboros/gateway/extensions.py` | Index / toggle / review / grants / lifecycle APIs |
| Widgets | `ouroboros/gateway/widgets.py` | **Passive** live projection of loader cards (`revision` = content_hash) |

Do not merge these nodes in docs or UI mental models. SPA: `web/modules/skills.js` vs `web/modules/widgets.js`.

## Presence side-lane (D20)

Not pooled Chat:

1. Review/enable Presence skill (D14) → `presence_profile` accepts `presence:` manifest  
2. Bind room→skill (`presence_bindings`)  
3. `admit_presence_turn` → immutable ceiling (`presence_authority`)  
4. `presence_runner` (serial, capped, idempotent)  
5. `presence_delivery` (+ Host Service `/presence/*`)

See [04-pipelines.md](04-pipelines.md) §Presence and [01-what-runs-where.md](01-what-runs-where.md).

## Domain roles on the spine

| Domain | Spine role |
|--------|------------|
| D18 | Launcher / packaging / platform substrate |
| D11 | Gateway, server entry, Web UI |
| D08 | Supervisor queue, workers, EVENT_Q |
| D09 | Cancel intents, owner stop, process custody, terminal delivery |
| D01 | Agent + LLM tool loop + outcomes/finalization |
| D03 | Context assembly, fit, compaction |
| D02 | LLM routing & providers |
| D16 | Seal, usage ledger, cost projection |
| D04/D05 | Tool registry/access & concrete surfaces |
| D13 | Safety / shell / runtime-mode guards |
| D06 | Review stack (**sibling**) |
| D07 | Delegation / Claudexor (**sibling**) |
| D15 | Memory / consciousness / evolution |
| D17 | Projects, leases, task results, workspace |
| D20 | Presence lane (**side**) |
| D19 | Frozen ABI at every boundary |
| D10 | Git / managed update (writer gate) |
| D12 | Settings feeding gates |
| D14 | Skills/extensions on tool & gateway surfaces |

## Control plane vs cognition

| Plane | Domains | Rule |
|-------|---------|------|
| Control | D08, D09, parts D10/D16 | Enqueue work; never invent a second queue |
| Gateway / SPA | D11, D12, D17 | Prefer widgets/skills over SPA forks |
| Cognition | D01–D07, D15 | New behavior = tool/skill/task contract |
| Extension | D14, D19, D20 | Core product lane |

## Related

- Topology detail → [01-what-runs-where.md](01-what-runs-where.md)
- Pipelines → [04-pipelines.md](04-pipelines.md)
- Gates → [05-decision-gates.md](05-decision-gates.md)
