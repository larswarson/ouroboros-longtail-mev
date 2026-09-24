# 07 — Invariants (never-break)

AGENT: read this when… before merging behavioral changes touching queue, cancel, money, skills, ABI, review, or process custody. Cite paths in the PR.

Evidence: `architecture/01-high-level-architecture.md`, `architecture/05-supervisor-loop.md`, `architecture/09-shutdown-and-process-cleanup.md`, `architecture/11-frozen-contracts-v1.md`, `architecture/12-host-service-companions-and-chat-ids.md`, `architecture/13-external-skills-layer.md` (via chapter index), `DOMAIN_MAP.md`, domain inventory. Full numbered list lives in upstream **Key invariants** chapter (`docs/ARCHITECTURE.md` → ch.10) when present in the checkout — this file distills load-bearing rules with path citations.

## Cancel & lifecycle

| Rule | Citation |
|------|----------|
| **Intent ≠ status** — cancel ingress writes durable intent only; never mutates canonical task status as proof of cancel | `ouroboros/cancel_intents.py`, `supervisor/task_lifecycle.py` |
| **Intent-then-custody** — no teardown without durable intent; natural completion can win; outbox owed before intent settle | `cancel_intents.py`, `terminal_delivery.py`, `cancel_publication.py` |
| **Queue is task-state authority** for PENDING/RUNNING under `_queue_lock`; `queue_snapshot.json` is recovery/diagnostic projection, **not** a second scheduler | `supervisor/queue.py`, `architecture/05-supervisor-loop.md` |
| Timeout reaping ≠ cancel ingress — reaper owns `reaping` slot custody | `supervisor/task_reaper.py`, ch.05 |
| Boot restores snapshot into empty pending; surviving RUNNING is **fenced** with cancel intent (`server_shutdown`), not resurrected as live work | ch.05 / `queue_snapshot` restore path |
| Interrupted parent leaves no orphan — stock custody/healers; don’t DIY child GC | `supervisor/workers.py`, `queue_snapshot.py`, D07 `delegate_custody*` |

## Money & seal

| Rule | Citation |
|------|----------|
| **Attempt ledger is monetary authority** — `state/usage_attempts.jsonl`; `llm_usage` / `state.json` are projections | `ouroboros/usage_accounting.py` |
| Unknown spend ≠ `$0.00` — null stays null; open upper bound disclosed | `cost_projection.py`, gateway `cost_breakdown.py` |
| Reserve before dispatch; never silent-drop reserved rows (settle / unresolve / release / terminalize) | `usage_accounting.py` |
| **Seal mismatch is a durable fact, not a second dispatch gate**; identity re-check may still refuse | `model_send_seal.py`, [05-decision-gates.md](05-decision-gates.md) |
| Money never reads a snapshot; display never waits on money | usage memo / `supervisor/state.py` |

## Scheduler & process

| Rule | Citation |
|------|----------|
| **One supervisor tick** — `server.py::_run_supervisor`; product features enqueue, never invent a second queue | ch.05, `supervisor/*` |
| Consciousness wake **only** from supervisor tick | ch.05, `consciousness_*.py` |
| Worker start: Linux forkserver / macOS+Windows spawn — never fork from multi-threaded supervisor | `worker_pool_lifecycle.py` |
| Process kill by **strict fingerprint**, never command-line class | `process_custody.py`, `launcher_server_reaper.py` |
| Same-install reaper requires /proc-proven argv + `OUROBOROS_DATA_DIR` + `OUROBOROS_MANAGED_BY_LAUNCHER=1` | `launcher_server_reaper.py` |
| Panic / Restart / window-close are distinct shutdown classes | `launcher.py`, `server_control.py`, ch.09 |
| Launcher owns PID lock and managed restart; packaged `server` alone is refused (bypasses custody) | `launcher.py`, `packaged_cli.py` |

## Review & skills

| Rule | Citation |
|------|----------|
| D06 / D07 are **sibling executors**, not nested in `run_llm_loop` | [02-runtime-spine.md](02-runtime-spine.md), D06/D07 modules |
| One paid review ceiling (`OUROBOROS_REVIEW_MAX_CYCLES`); paid stamp at physical dispatch | `review_dispatch.py`, `review_cycles.py`, `commit_gate.py` |
| Skill gates do not collapse — discover ≠ preflight ≠ review ≠ grant ≠ deps ≠ enable ≠ execute | ch.13, `skill_loader.py`, `skill_readiness.py` |
| Skills catalogue ≠ Widgets live projection | `gateway/extensions.py` vs `gateway/widgets.py` |
| Acceptance wallet: UNKNOWN if no terminal host run — never silent re-dispatch | `task_results.claim_task_acceptance_review_cycle` |

## Projects & Presence

| Rule | Citation |
|------|----------|
| One top-level writer per `project_id` (`project_lease`); subagent swarms exempt | `project_lease.py`, `assign_tasks` |
| Tombstones never resurrect; never age-pruned from registry | `projects_registry.py` |
| Presence is a **side-lane** (bind→admit→run→deliver), not pooled Chat | `presence_runner.py`, ch.12 |
| Workspace set-but-unusable → typed refusal | `workspace_admission.py` |

## ABI & constitution

| Rule | Citation |
|------|----------|
| Frozen contracts extend **explicitly** — additive or versioned successor | `ouroboros/contracts/`, ch.11 |
| Bad durable schema rows quarantine; don’t invent silent converters | `task_result_schema.py`, usage quarantine paths |
| `BIBLE.md` persists; `identity.md` stays a file | D15 memory paths, constitution references in update path |
| Single settings owner (`ouroboros/config.py` + leaves); single messaging bus (`supervisor/message_bus.py`) | D12, D08 |
| Chat id is a value class (web / hidden / A2A / project) — not a boolean | `contracts/chat_id_policy.py`, ch.12 |
| Host Service tokens hash-bound; secrets never in token; payload edit stales token | `skill_token.py`, `host_service.py` |

## Safety

| Rule | Citation |
|------|----------|
| Tool/shell mutations pass D13 then D04 access then D05 handler | `safety.py`, `runtime_mode_policy.py`, `tool_access.py` |
| Protected artifacts / frozen contracts / bible / release paths stay gated by runtime mode | `protected_artifacts.py`, `runtime_mode_policy.py` |
| Repo writer gate closed around managed update | `workers.close_repo_writer_admission`, D10 update modules |

## Memory / continuity

| Rule | Citation |
|------|----------|
| Knowledge authority = files with revision-checked writes | `knowledge.py`, `memory.py` |
| Missing consolidation generation → loud MEMORY GAP, never silent cursor reset | `consolidator.py` |

## Diagram / docs discipline

| Rule | Citation |
|------|----------|
| Do not nest D06/D07 inside `run_llm_loop` in docs or diagrams | [02-runtime-spine.md](02-runtime-spine.md) |
| Do not equate Skills catalogue with Widgets | [04-pipelines.md](04-pipelines.md) |
| Module→domain assignment SSOT = `ouroboros/domains.toml` / `DOMAIN_MAP.md` | `scripts/check_domains.py` |

## Related

- Gates → [05-decision-gates.md](05-decision-gates.md)
- Extend without breaking → [06-extension-surface.md](06-extension-surface.md)
- Upstream chapter map → [09-upstream-doc-map.md](09-upstream-doc-map.md)
