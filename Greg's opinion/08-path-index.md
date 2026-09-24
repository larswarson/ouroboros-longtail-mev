# 08 — Path index

AGENT: read this when… jumping from a symptom or feature name to the owning file. Paths are repo-relative.

Evidence: `DOMAIN_MAP.md` module lists, architecture ch.01/05/12, `launcher.py` / `server.py` headers.

## Launcher / packaging (D18)

| Path | Role |
|------|------|
| `launcher.py` | Immutable outer shell: PID lock, bootstrap, spawn/reap server, UI presentation, restart/panic |
| `ouroboros/launcher_bootstrap.py` | Bundle→managed repo, deps, launch options, external host update hook |
| `ouroboros/launcher_server_reaper.py` | Same-install /proc-proven stray server reap |
| `ouroboros/launcher_windows_runtime.py` | Windows webview runtime helpers |
| `ouroboros/launcher_onboarding.py` | First-run settings presentation helpers |
| `ouroboros/platform_layer.py` | OS locks, Job/pgid, birth identity primitives |
| `ouroboros/cli.py` | Headless/CLI gateway client; refuses non-root `delegation_role` |
| `ouroboros/packaged_cli.py` / `packaged_cli_install.py` | Packaged CLI wrapper + installer |
| `ouroboros/node_runtime.py` | Node health / skill-family Node precedence |
| `ouroboros/verified_download.py` | Digest-verified cached fetch |
| `ouroboros/utils.py` | Shared helpers (cycle-free leaf) |

## Server / gateway / Web (D11)

| Path | Role |
|------|------|
| `server.py` | Starlette/uvicorn composition root; lifespan; `_run_supervisor` |
| `ouroboros/server_entrypoint.py` | Arg parse, free port, write port file |
| `ouroboros/server_web.py` | Static SPA / index |
| `ouroboros/server_auth.py` | `NetworkAuthGate` |
| `ouroboros/server_runtime.py` | Provider-ready / runtime helpers |
| `ouroboros/server_owner_routing.py` | Mailbox, promote/route, attachments |
| `ouroboros/server_control.py` | Panic / restart process control |
| `ouroboros/server_liveness.py` | Supervisor/chat wedge watchdog |
| `ouroboros/server_maintenance.py` | Startup custody, prune, periodic reconcile |
| `ouroboros/server_restart.py` | Restart coordination |
| `ouroboros/server_process.py` | Process-local flags, DATA_DIR, binding records |
| `ouroboros/gateway/router.py` | Route table SSOT |
| `ouroboros/gateway/endpoint_index.py` | Endpoint index |
| `ouroboros/gateway/ws.py` | WebSocket control plane |
| `ouroboros/gateway/tasks.py` | Task create/list/cancel/resume HTTP |
| `ouroboros/gateway/task_decision.py` | Owner quiz/decision cards |
| `ouroboros/gateway/routing_decision.py` | Routing decision cards |
| `ouroboros/gateway/task_model_wait.py` | Model-wait cards |
| `ouroboros/gateway/task_hurry.py` | Hurry latch HTTP |
| `ouroboros/gateway/task_events.py` | SSE/task event projection |
| `ouroboros/gateway/history.py` / `history_paging.py` | Chat/progress history |
| `ouroboros/gateway/onboarding.py` / `onboarding_host.py` | Wizard complete transaction |
| `ouroboros/gateway/settings.py` / `owner_settings.py` | Settings HTTP |
| `ouroboros/gateway/projects.py` | Projects HTTP |
| `ouroboros/gateway/extensions.py` | **Skills catalogue** APIs |
| `ouroboros/gateway/widgets.py` | **Widgets** passive live projection |
| `ouroboros/gateway/marketplace.py` | Hub install/list |
| `ouroboros/gateway/skill_publish.py` | Publish HTTP |
| `ouroboros/gateway/host_service.py` | Loopback Host Service (default :8767) |
| `ouroboros/gateway/presence_settings.py` | Presence owner knobs |
| `ouroboros/gateway/cost_breakdown.py` | Cost tables from D16 |
| `ouroboros/gateway/contracts.py` | Browser envelope ABI |
| `ouroboros/gateway/control.py` | Control / update surfaces |
| `web/` | SPA; ES modules under `web/modules/` |
| `web/modules/api_types.js` | JSDoc mirror of gateway contracts |
| `web/modules/skills.js` / `widgets.js` | Catalogue vs widgets UI |

## Supervisor (D08 / D09 / D10 / D15)

| Path | Role |
|------|------|
| `supervisor/queue.py` | Queue authority + `_queue_lock`; enqueue / contract attach |
| `supervisor/task_admission.py` | Reserve/release admission slots |
| `supervisor/worker_assignment.py` | **Assign now?** gates |
| `supervisor/workers.py` | Worker pool façade; repo writer gate |
| `supervisor/worker_pool_lifecycle.py` | Spawn/kill trees; readiness windows |
| `supervisor/worker_process.py` | Worker process entry |
| `supervisor/worker_health.py` | Death recovery / terminal file recovery enqueue |
| `supervisor/worker_chat_lane.py` | Direct-chat lane |
| `supervisor/worker_owner_wait.py` | Owner-wait capacity lend |
| `supervisor/worker_promotion.py` | Chat→task promotion |
| `supervisor/events.py` | `dispatch_event` fan-in |
| `supervisor/events_*.py` | Typed event handlers (budget, chat delivery, evolution done, schedules, subagent admission, task done, …) |
| `supervisor/message_bus.py` | Messaging owner |
| `supervisor/queue_snapshot.py` | Persist/restore `queue_snapshot.json` |
| `supervisor/queue_timeouts.py` | Idle/deadline → reaper |
| `supervisor/queue_transitions.py` | Status transitions under lock |
| `supervisor/task_lifecycle.py` | Cancel claim/settle (D09) |
| `supervisor/task_reaper.py` | Off-tick kill/join/respawn (D09) |
| `supervisor/cancel_publication.py` | Typed cancel outcomes (D09) |
| `supervisor/owner_stop.py` | Graceful finalize-then-cancel (D09) |
| `supervisor/steering.py` | Steer vs cancel refusal (D09) |
| `supervisor/terminal_delivery.py` | Terminal outbox (D09) |
| `supervisor/evolution_lifecycle.py` | Evolution campaign admission (D15↔D08) |
| `supervisor/git_ops*.py` / `update_*.py` | Managed git/update (D10) |
| `supervisor/state.py` | Runtime state helpers / external owner slots |
| `supervisor/direct_roots.py` | Direct-chat root tracking |
| `supervisor/active_activity.py` | Activity / progress evidence |
| `supervisor/plan_obligation.py` | Plan obligation tracking |
| `supervisor/subagent_task_truth.py` | Subagent task truth projection |

## Agent core / loop (D01)

| Path | Role |
|------|------|
| `ouroboros/agent.py` | `OuroborosAgent.handle_task` |
| `ouroboros/agent_dispatch.py` | Dispatch axes / executor note |
| `ouroboros/agent_startup_checks.py` | Pre-loop startup gates |
| `ouroboros/agent_task_pipeline.py` | Emit task results / post-task synthesis recovery |
| `ouroboros/loop.py` | `run_llm_loop` |
| `ouroboros/loop_llm_call.py` / `loop_model_call.py` | Model call + retry |
| `ouroboros/loop_tool_execution.py` | Tool call handling |
| `ouroboros/loop_budget.py` / `loop_round_limits.py` / `loop_forced_finalization.py` | Soft-land / force final |
| `ouroboros/loop_acceptance.py` / `loop_acceptance_review.py` | Acceptance fence |
| `ouroboros/loop_transport.py` | Transport wait episodes |
| `ouroboros/outcomes.py` / `outcome_receipt_store.py` | Terminal outcome axes / receipts |
| `ouroboros/owner_wait.py` / `owner_mailbox.py` | Owner wait / mailbox |
| `ouroboros/task_finalization.py` | Finalization helpers |
| `ouroboros/post_task_synthesis.py` / `post_task_checkpoint.py` | Post-task synthesis |

## LLM / context / tools / safety

| Path | Role |
|------|------|
| `ouroboros/llm.py` + `llm_*.py` | Provider clients / routing / fallback (D02) |
| `ouroboros/request_wire_*.py` | Wire contract / recovery (D02) |
| `ouroboros/context.py` + `context_*.py` | Context assembly/fit (D03) |
| `ouroboros/tools/registry*.py` | Tool registry (D04) |
| `ouroboros/tool_access*.py` / `tool_policy.py` | Access + schema policy (D04) |
| `ouroboros/tools/extension_dispatch.py` | Extension/MCP tool dispatch (D04) |
| `ouroboros/tools/shell.py` / `browser.py` / `core_*.py` / `mcp_client.py` | Surfaces (D05) |
| `ouroboros/safety.py` / `runtime_mode_policy.py` / `tools/shell_guards.py` | Safety floor (D13) |

## Review / delegation

| Path | Role |
|------|------|
| `ouroboros/review*.py` / `commit_admission.py` / `triad_review.py` | Review substrate (D06) |
| `ouroboros/tools/plan_review*.py` / `scope_review*.py` / `review*.py` | Review tools (D06) |
| `ouroboros/delegate_*.py` / `subagent*.py` / `claudexor_*.py` | Delegation (D07) |
| `ouroboros/tools/delegate*.py` / `control_delegation.py` / `control_scheduling.py` | Delegation tools (D07) |

## Skills / contracts / Presence / memory / cost / projects

| Path | Role |
|------|------|
| `ouroboros/skill_loader.py` / `skill_readiness.py` / `skill_review*.py` | Skill gates (D14) |
| `ouroboros/extension_*.py` / `marketplace/*` | Extensions + marketplace (D14) |
| `ouroboros/tools/skill_exec.py` / `skill_preflight.py` | Skill exec tools (D14) |
| `ouroboros/contracts/*` | Frozen ABI (D19) |
| `ouroboros/presence_*.py` / `tools/presence.py` | Presence side-lane (D20) |
| `ouroboros/memory.py` / `consolidator.py` / `consciousness*.py` / `evolution_*.py` | Memory/autonomy (D15) |
| `ouroboros/usage_accounting.py` / `model_send_seal.py` / `cost_projection.py` | Ledger + seal (D16) |
| `ouroboros/projects_registry.py` / `project_lease.py` / `workspace_*.py` / `task_results.py` | Projects/results (D17) |
| `ouroboros/config.py` / `settings_*.py` / `onboarding_wizard.py` | Settings (D12) |
| `ouroboros/process_custody.py` / `cancel_intents.py` | Custody + cancel intent (D09) |

## Docs (upstream)

| Path | Role |
|------|------|
| `docs/ARCHITECTURE.md` | Architecture book index |
| `architecture/*.md` | Mechanism chapters (also under `docs/architecture/` in full checkouts) |
| `DOMAIN_MAP.md` / `docs/DOMAIN_MAP.md` | Module→domain SSOT listing |
| `docs/CREATING_SKILLS.md` | Skill authoring |
| `docs/CHECKLISTS.md` | Review checklists |
| `BIBLE.md` | Constitution |
| `prompts/SYSTEM.md` / `SAFETY.md` / `CONSCIOUSNESS.md` | Runtime prompts (not this agent-doc pack) |

## Related

- Topology → [01-what-runs-where.md](01-what-runs-where.md)
- Domains → [03-domain-map.md](03-domain-map.md)
- Upstream chapter map → [09-upstream-doc-map.md](09-upstream-doc-map.md)
