# 01 — What runs where

AGENT: read this when… answering “which process owns X?”, debugging spawn/reap, ports, or on-disk state. Must be able to name PID-lock owner, supervisor tick host, agent-loop host, cancel settle, Presence runner.

Evidence: `architecture/01-high-level-architecture.md` (Runtime topology, Data layout), `architecture/05-supervisor-loop.md`, `architecture/09-shutdown-and-process-cleanup.md`, `architecture/12-host-service-companions-and-chat-ids.md`, `launcher.py`, `server.py`, `ouroboros/config.py`.

## Two continuity roles

| Role | Entry | Owns | Does not own |
|------|-------|------|--------------|
| **Launcher** | `launcher.py` (D18) | PID lock, bundle→managed repo, spawn/reap of `server.py`, presentation (pywebview/browser), restart signal (exit 42), Job/pgid cleanup, same-install stray reaper | Self-editable business logic; editing `server.py` internals |
| **Server** | `server.py` (D11 composition) | HTTP/WS gateway, lifespan boot, supervisor tick thread, worker pool, extensions/companions, Host Service, Presence settings ingress | Immutable packaged shell / PID lock |

Native systemd user unit (if used) is **alternate ingress**, not a third role — no restart policy; launcher owns managed restart, crash fuse, panic-to-complete-stop.

## Spawn / custody (who spawns whom)

```
launcher.py
  acquire PID lock (platform_layer / acquire_pid_lock)
  launcher_bootstrap → managed repo under APP_ROOT/repo
  launcher_server_reaper.reap_same_install_strays  (preflight + each generation)
  spawn server.py
      POSIX: new session / process group
      Windows: create suspended → kill-on-close Job → resume
      write data/state/server_process.json  (pid, pgid, paths, ports, argv, creation time)
      poll PORT_FILE + GET /api/health
server.py lifespan
  settings → reconcile_projects → init_global_supervisor
  Host Service / extensions / review-job reconcile
  _start_supervisor_if_needed → _run_supervisor (D08 tick)
  gateway routes (ouroboros/gateway/*)
  worker pool (supervisor/workers*) → OuroborosAgent.handle_task (D01)
  Presence side-lane (ouroboros/presence_runner.py) via Host Service / bindings
```

### PID lock

- **Owner:** launcher generation (`launcher.py` + `ouroboros/config.py` / `platform_layer` exclusive file locks).
- **Meaning:** one launcher generation per install; losing the lock → soft-poll port file and open last-read loopback URL (repeat launch).
- **Release:** registered `atexit`; SIGINT/SIGTERM in browser mode only set shutdown event (do not double-release).

### Same-install reaper (`ouroboros/launcher_server_reaper.py`)

- Holding PID lock **licenses** the reap (main preflight + top of every launcher generation).
- Kill only if **/proc-proven** all three: exact `<REPO_DIR>/server.py` argv token, `OUROBOROS_DATA_DIR`, `OUROBOROS_MANAGED_BY_LAUNCHER=1`.
- Never on Panic or window-close; spares retained daemon descendants. Non-/proc hosts stay report-only.

### Durable process custody (`ouroboros/process_custody.py`)

- `spawn_supervised()` → `data/state/process_ledger.jsonl` fingerprint rows.
- Custody reaper: server startup + ~10-minute supervisor maintenance; kill only when generation/task owner gone — **strict fingerprint**, never command-line class.
- Daemon roots spared; skill companions reaped on uninstall / foreign generation (log-only default).

## Where the supervisor tick runs

| Fact | Location |
|------|----------|
| Tick function | `server.py::_run_supervisor` |
| Domain | D08 (`supervisor/*`) |
| Threading | Server process; dedicated supervisor loop (not a worker) |
| Healthy tick order | liveness → rotate logs → worker health → **drain events** (bridge before timeouts) → deadlines/schedules → reconciliation/evolution → **assign** → persist `state/queue_snapshot.json` → **consciousness alarm last** |
| Failure policy | Three consecutive loop failures clear readiness + notify owner |
| Watchdog | `ouroboros/server_liveness.py` — observes loop / native actors; does not invent a second scheduler |

**Consciousness wake** is admitted **only** from this tick — no private consciousness thread (`architecture/05-supervisor-loop.md`).

## Where the agent loop runs

| Fact | Location |
|------|----------|
| Assign | `supervisor/worker_assignment.assign_tasks` under queue lock + D17 `project_lease` |
| Worker process | `supervisor/workers.py`, `worker_process.py`, `worker_pool_lifecycle.py` |
| Agent entry | `ouroboros/agent.py` — `OuroborosAgent.handle_task` |
| LLM↔tools loop | `ouroboros/loop.py::run_llm_loop` (ordinary tasks) |
| Special bypass | `task_type == "deep_self_review"` skips tool loop |
| Start method | Linux **forkserver**; macOS/Windows **spawn** — never fork from multi-threaded supervisor |

Direct-chat / native actors: `supervisor/worker_chat_lane.py`, `direct_roots.py` — in-process actors, still not a second scheduler.

## Where cancel settles

| Stage | Path |
|-------|------|
| Intent ingress | `ouroboros/cancel_intents.request_cancel` → `data/state/cancel_intents.json` (**never** mutates canonical status) |
| Queue claim/settle | `supervisor/task_lifecycle.py` under D08 `_queue_lock` |
| Publication | `supervisor/cancel_publication.py` |
| Graceful wrap-up | `supervisor/owner_stop.py` (`finalize_then_cancel`) |
| Off-tick kill/join/respawn | `supervisor/task_reaper.py` (reaper thread; tick stays fast) |
| Process trees | `process_custody` + `worker_pool_lifecycle.kill_worker_tree` |
| Terminal outbox | `supervisor/terminal_delivery.py` → `data/state/terminal_deliveries.json` |

Vocabulary for **cancel confirmed?** → [05-decision-gates.md](05-decision-gates.md).

## Where Presence runs

| Stage | Path |
|-------|------|
| Owner knobs | `ouroboros/gateway/presence_settings.py` (D11) |
| Bindings | `ouroboros/presence_bindings.py` + `data/state/presence_bindings.json` |
| Admit | `ouroboros/presence_admission.admit_presence_turn` |
| Ceiling | `ouroboros/presence_authority.py` |
| Runner | `ouroboros/presence_runner.py` — serial, installation-capped, idempotent; attaches task_contract + ceiling; runs a **fresh** agent turn |
| Delivery | `ouroboros/presence_delivery.py` (+ Host Service `/presence/*`) |
| Tools | `ouroboros/tools/presence.py` |

Presence is **not** a pooled Chat worker. Ingress often via Host Service `POST /presence/turn` (`architecture/12-*.md`).

## Host Service

| Item | Fact |
|------|------|
| Module | `ouroboros/gateway/host_service.py` |
| Bind | Loopback `127.0.0.1:${OUROBOROS_HOST_SERVICE_PORT:-8767}` |
| Auth | `x-skill-token` hash-bound (`ouroboros/skill_token.py`) — secrets never in token |
| Frozen route family | `/identity`, `/tools/schemas`, `/chat/*`, `/presence/*`, `/ui/ws-message`, WS `/events` |
| Purpose | Callback boundary for **reviewed** skills / companions — not a public WAN API |

Companion processes: manifest-declared, host-supervised; live projection `data/state/extension_companions.json`.

## Ports / defaults

| Port / file | Default / path | Owner |
|-------------|----------------|-------|
| Agent HTTP/WS | `8765` (`AGENT_SERVER_PORT`) | `server.py` / uvicorn |
| Port handoff file | `data/state/server_port` (`PORT_FILE` / `OUROBOROS_PORT_FILE`) | Server writes; launcher polls |
| Bindings snapshot | `data/state/server_port.bindings.json` | Informational host/port facts for main / Host Service / local-model — **not** a grant |
| Host Service | `8767` (`OUROBOROS_HOST_SERVICE_PORT`) | `gateway/host_service.py` |
| Non-loopback | `OUROBOROS_SERVER_HOST=0.0.0.0` (Docker etc.) | Requires `NetworkAuthGate` password when configured |

Health: launcher waits `GET /api/health`; CLI `run --start` waits health + `supervisor_ready`.

## On-disk state roots (agent-critical)

Default under `~/Ouroboros/data/` (`DATA_DIR`):

| Path | Role |
|------|------|
| `settings.json` | Owner settings (API keys, models, budget) — D12 |
| `state/state.json` | Runtime + compatibility cost projection — **not** monetary authority |
| `state/queue_snapshot.json` | Recovery/diagnostic projection of PENDING/RUNNING — **not** a second scheduler |
| `state/usage_attempts.jsonl` | **Monetary authority** (D16) |
| `state/cancel_intents.json` | Active cancel intent projection (D09) |
| `state/terminal_deliveries.json` | Terminal answer outbox / dedupe |
| `state/server_process.json` | Launcher-owned server identity |
| `state/process_ledger.jsonl` | Durable process custody |
| `state/projects.json` | Project registry (tombstones never age-pruned) |
| `state/skills/<name>/` | Per-skill review/enable/grant/deps/health/token |
| `state/presence_bindings.json` | Presence room→skill bindings |
| `state/extension_companions.json` | Live companion snapshot |
| `state/extension_reconcile/` | Worker→server reconcile markers |
| `state/post_task_evolution_request.json` | One-shot evolution promote signal (worker→supervisor idle tick) |
| `task_results/` | Durable task results (`_schema_version: 1`; quarantine bad rows) |
| `task_drives/<task_id>/` | Task-scoped scratch |
| `memory/` | Identity, scratchpad, dialogue blocks, knowledge shelf |
| `claudexor/` | Ouroboros-owned Claudexor home (not `~/.claudexor`) |

Repo (self-editable): `~/Ouroboros/repo/` — `ouroboros/`, `supervisor/`, `web/`, `docs/`, `prompts/`.

## Shutdown classes (launcher / `server_control`)

| Class | Effect |
|-------|--------|
| Window-close | Keep shared Claudexor daemon; launcher Job/cleanup rules apply |
| Restart (exit 42) | Clean-stop this generation; refresh bundle/deps; relaunch |
| Panic | Complete explicit stop; full cleanup; outer process terminates (no retry loop) |

Crash fuse: five ordinary crashes within 120s stop automatic restart (`architecture/01-*.md`).

## Answer cheat-sheet

| Question | Answer |
|----------|--------|
| Who owns PID lock? | Launcher (`launcher.py`) |
| Who spawns server? | Launcher → `server.py` |
| Where does supervisor tick run? | Inside server: `server.py::_run_supervisor` |
| Where does agent loop run? | Worker process → `ouroboros/agent.py` / `loop.py` |
| Where does cancel settle? | D09 intent → `task_lifecycle` under queue lock → reaper / publication |
| Where does Presence run? | `presence_runner.py` side-lane (+ Host Service `/presence/*`) |

## Related

- Spine → [02-runtime-spine.md](02-runtime-spine.md)
- Gates → [05-decision-gates.md](05-decision-gates.md)
- Paths → [08-path-index.md](08-path-index.md)
