# Ouroboros — How It All Works

*A guided tour of `razzant/ouroboros` at **v7.4.7** — 569 Python modules, ~320,000 lines of code, grouped into 20 domains. Every claim in this document was verified against the repository source and its own architecture book (`docs/ARCHITECTURE.md`, 13 chapters). Companion to the interactive module map.*

---

## 1. What this thing is

Ouroboros is a self-creating, general-purpose AI agent: it holds a persistent identity and memory across restarts, works on external projects, coordinates a swarm of specialist sub-agents, and can evolve by rewriting the very code it runs on. Despite that scope, the runtime has a surprisingly legible shape: **one launcher, one server, one scheduler, one agent loop** — surrounded by five satellites (review, delegation, cancellation, skills, presence) and a strict constitution of invariants that keep the self-modifying parts honest.

## 2. Process topology — who is alive, and where

```
you (browser / CLI / Telegram)
        │
        ▼
launcher.py                    desktop/native shell; immutable package; exit code 42 = managed restart
        │  spawns
        ▼
server.py                      Starlette + uvicorn, HTTP + WebSocket on 127.0.0.1:8765
        ├── supervisor/        ONE background scheduler thread (the tick)
        │      └── workers.py  multiprocessing pool (forkserver on Linux, spawn on macOS/Windows)
        │             └── each worker runs ouroboros/agent.py — the task engine
        └── ouroboros/gateway/ the HTTP/WS API surface the SPA and CLI talk to
```

Two facts explain most of the design:

- **The supervisor is the only scheduler.** Its tick drains worker events, enforces deadlines, assigns queued work under one lock, persists a snapshot, and — last — ticks the consciousness alarm clock. Consciousness owns no thread; only the tick can start a wake-up.
- **Workers are disposable, the ledger is not.** Money, cancellation and results live in durable, locked files; processes are custody, and custody is always provable after a crash.

## 3. The spine

Four domains form the load-bearing path every ordinary task travels:

| Step | Domain | What it does |
|---|---|---|
| **D18** | Launcher, packaging & platform substrate | Boots the installation, spawns the server, owns the restart handshake; provides the shared utilities (`utils.py` is imported by 286 modules). |
| **D11** | Gateway, server & Web UI | Admits the work: `POST /api/tasks`, workspace validation, onboarding gate, owner decision cards. |
| **D08** | Supervisor | Queues it (PENDING→RUNNING under `_queue_lock`), assigns it to a worker, watches health, mirrors status durably. |
| **D01** | Agent core & main loop | Runs it: `agent.py` orchestrates; `loop.py::run_llm_loop` alternates model calls (D02) and tool execution (D04/D05) until a typed `FINAL ANSWER:`, a budget wall, or a stop. |

Around the loop sit its constant companions: **D03** (context assembly, fitting and compaction — what the model sees), **D02** (the multi-provider LLM client with its fallback ladder), **D16** (every physical send is a paid ledger row), **D04/D05** (tool policy and tool surfaces), **D13** (the safety veto).

## 4. The siblings and the side-lane

Three hard rules the map enforces, all verified in code:

1. **Review and delegation are sibling executors, not loop internals.** D06 (review) and D07 (delegation) are invoked through tools and gates; `review_execution.py` explicitly "never imports the coordinator." They have their own custody, budgets and evidence trails.
2. **Skills ≠ Widgets.** The skills *catalogue* (`/api/extensions`) is the real extension state; the Widgets page (`/api/widgets`) is a passive, read-only projection of the live loader — no discovery, no writes.
3. **Presence is a side-lane.** Messenger turns (e.g. Telegram) travel bind → admit → run → deliver through the loopback host service, under an immutable per-turn capability ceiling. They are never pooled with ordinary chat.

## 5. The 20 domains, in plain language

- **D01 Agent core & main loop (37 modules)** — the worker-side engine: orchestration, the LLM↔tools loop, acceptance fence, budget rails, finalization, startup checks, result pipeline.
- **D02 LLM client, routing & providers (38)** — provider routing (OpenRouter, OpenAI-compatible, native Anthropic, GigaChat, DeepSeek, local llama.cpp, Claudexor), wire shaping, streaming, exact-route wire repair, fallback ladder.
- **D03 Context assembly, fit & compaction (11)** — builds the prompt; deterministic Max/Low/Nano projections; transactional compaction with provenance capsules; reduction by relocation, never silent truncation.
- **D04 Tool execution core (21)** — the registry, access policy and typed outcome classification; a policy denial never masquerades as a tool crash.
- **D05 Tool surfaces (28)** — the actual verbs: files, code, shell, search, media, schedules, control, review, presence, MCP-bridged externals.
- **D06 Review stack (67)** — paid multi-model review (plan, acceptance, commit, skill) sharing one monetary ceiling counted at physical dispatch; replays of identical material are free.
- **D07 Delegation, subagents & Claudexor (53)** — spawning child agents, durable custody answering OWNED/FOREIGN/UNKNOWN per run, nanny pacing, evidence capture; recursion never widens authority.
- **D08 Supervisor (47)** — the single scheduler tick, the queue, the worker pool, event handlers, schedules, runtime control.
- **D09 Cancellation & custody (13)** — intent-then-custody: a durable intent row before any teardown; one settle owner; natural completion wins; the answer is owed before it's published.
- **D10 Git, update & release (28)** — bounded git runner, managed updates (channel → exact SHA → rescue → merge → smoke → verified rollback), release metadata sync.
- **D11 Gateway, server & Web UI (55)** — the front door: tasks, history, settings, onboarding, files, logs, costs, extensions, widgets, marketplace, MCP, Claudexor accounts.
- **D12 Settings & configuration (15)** — `config.py` is the one import surface; leaves own the vocabularies; one key, one home, one default.
- **D13 Safety, guards & runtime mode (9)** — the independent Safety Supervisor veto (fail-closed on oversized subjects; a 429 is infrastructure, not a verdict) and the Light/Full mode clamps.
- **D14 Skills & extensions (56)** — the plugin plane: discovery → preflight → hash-bound review → grants → dependencies → enablement → execution; the gates never collapse.
- **D15 Memory, knowledge, consciousness & self-evolution (23)** — scratchpad/identity/history, revision-checked knowledge notes, dialogue consolidation with loud `[MEMORY GAP]` honesty, the consciousness alarm clock, evolution campaigns.
- **D16 Observability, usage & cost (11)** — the append-only attempt ledger is the monetary authority; all totals are projections; unknown cost is `null`, never `$0.00`.
- **D17 Projects, workspaces & task results (22)** — durable project registry with tombstones, one-writer-per-project leases, workspace admission, the schema-guarded result store.
- **D18 Launcher & shared substrate (15)** — the desktop shell, packaging, platform locks, and the utilities everyone imports.
- **D19 Frozen contracts / ABI (10)** — the versioned backward-compatible shapes (`contracts/`) that skills, extensions and the browser rely on; extend explicitly or version a successor.
- **D20 Presence (10)** — the messenger side-lane with per-turn immutable capability ceilings and typed delivery receipts.

## 6. The seven pipelines

1. **Chat task.** UI/WS or `POST /api/tasks` → workspace admission → queue (PENDING→RUNNING) → worker → agent loop → acceptance fence → `task_results/<id>.json` + usage ledger → terminal answer delivered (registered as *owed* before it's published, so a crash replays rather than loses it).
2. **Review gate.** Sibling executor D06: admission → paid stamp counted at dispatch against the shared cycle ceiling → reviewer panel wave → acceptance wallet → git integration when the gate is a commit gate. Missing evidence is DEGRADED/NOT_RUN, never PASS.
3. **Delegation.** Sibling executor D07: `schedule_subagent` is the only door (the gateway rejects lineage labels from anyone else) → Claudexor transport → custody ledger → nanny pacing → evidence and output capture back to the parent.
4. **Skills & extensions.** Discovery → deterministic preflight → hash-bound multi-model review → owner grants → dependency readiness → enablement → execution. A PASS installs nothing; `enabled=true` proves nothing about readiness.
5. **Cancel / custody.** Every ingress writes one durable intent row first (fail-closed) → the settle owner claims it → kill → re-check (natural completion wins) → honest cost → owner's answer owed → published → watchdog sweeps strays. Status is never used as a cancel signal.
6. **Presence.** Transport event → `/presence/turn` → binding check → admission compiles the immutable ceiling → serialized fresh-agent run → typed provider delivery receipts into canonical history.
7. **Consciousness & evolution.** The supervisor tick is the only clock: `consciousness.tick(now)` → rolling-24h allowance read off the usage ledger → an ordinary Main direct turn with dispatch-bound authority. Evolution campaigns admit through `evolution_lifecycle`, checkpoint deterministically, fingerprint against repeats, and stand down in Light mode.

## 7. Where things get decided — the 18 gates

| # | Gate | Decided at | Rule |
|---|---|---|---|
| 1 | Start the supervisor? | `server.py` lifespan | Only if provider config is structurally sufficient |
| 2 | Onboarding complete? | `gateway/onboarding` | One settings transaction |
| 3 | Admit task/workspace? | `gateway/tasks` + `workspace_admission` | Loud typed failure, repair in `detail` |
| 4 | Owner card answer? | `task_decision` / `routing_decision` | One receipt seam; no shadow channel |
| 5 | Create a child? | D07 `schedule_subagent` only | Gateway rejects lineage labels |
| 6 | Assign now? | D08 queue lock + D17 `project_lease` | One top-level writer per project |
| 7 | Send to model? | `model_send_seal` + D02 | model-visible ⟺ logged; mismatch is a fact, not a gate |
| 8 | Pay for attempt? | `usage_accounting.reserve_attempt` | BudgetExceeded under the ledger lock |
| 9 | Wire-repair or fallback? | `request_wire_recovery` vs `llm_fallback` | Exact-route repair, recorded only after success |
| 10 | May this tool run? | D13 → D04 → D05 | Guards, then access, then handler |
| 11 | Spend a review cycle? | D06 ↔ D16 | Shared `OUROBOROS_REVIEW_MAX_CYCLES` counted at dispatch |
| 12 | Accept review cycle? | `claim_task_acceptance_review_cycle` | No terminal host run → UNKNOWN |
| 13 | Skill may execute? | `skill_loader` / `skill_readiness` | review ≠ grant ≠ deps ≠ enable ≠ exec |
| 14 | Evolution enqueue? | D15 ↔ D08 | Triple-fenced in Light mode |
| 15 | Consciousness wake? | D08 tick → D15 | No own thread; allowance from the ledger |
| 16 | Presence turn? | `presence_admission.admit_presence_turn` | Immutable per-turn ceiling |
| 17 | Cancel confirmed? | D09 intent → custody | Intent row first; never status-as-cancel |
| 18 | Shutdown class? | launcher / `server_control` | close (verify child death) / Restart (exit 42) / Panic (ledger identity) |

## 8. The constitution — invariants worth knowing

The repo's §10 lists 28 numbered invariants; these ten do most of the work:

1. **BIBLE.md and identity.md persist** — the constitution is never deleted.
2. **One projection for release metadata** — VERSION is canonical; every carrier is synced.
3. **One owner per concern** — config has one import surface; messages one bus; locks one owner each.
4. **The attempt ledger is the monetary authority** — everything else is a projection carrying attempt identity.
5. **Skill gates never collapse** (see pipeline 4).
6. **Cancellation is intent-then-custody** (see pipeline 5).
7. **Frozen contracts extend explicitly** — the ABI promise.
8. **Provider wire adaptation is exact-route and success-confirmed** — failed experiments teach nothing durable.
9. **Projection over replay** — UI reads are bounded projections; durable owners do the one authoritative replay.
10. **Money never reads a snapshot; a display never waits on money** — admission reads under the lock; displays may serve the last validated snapshot, and a cold read answers *unknown*, never zero.

## 9. How to explore further

- The **interactive map** (companion page): click any domain for its summary, decision points and module list; search filters all 569 modules; module rows expand into mechanism descriptions with key symbols and real import neighbors, sorted by import-graph centrality.
- The repo's own book: `docs/ARCHITECTURE.md` → 13 chapters. Start with chapters 1 (structure tree), 5 (supervisor), 6 (agent core), 10 (invariants).
- Module-level truth: every module's docstring is maintained as part of the documentation contract (`tests/test_docs_sync.py` pins docs to code), which is what makes this map's summaries trustworthy.

*Generated 2026-09-24 against razzant/ouroboros @ v7.4.7 (`ouroboros` branch).*
