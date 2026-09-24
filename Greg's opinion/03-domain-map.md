# 03 — Domain map (D01–D20)

AGENT: read this when… locating ownership for a path, checking import direction, or listing key decisions before a change.

Evidence: `DOMAIN_MAP.md` (module lists + dependency matrix), domain inventory synthesis, upstream architecture chapters. Module counts are pinned totals from the domain inventory (**556**).

## Index

| ID | Name | Modules | Key prefixes |
|----|------|--------:|--------------|
| [D01](#d01--agent-core--main-loop) | Agent core & main loop | 33 | `ouroboros/agent.py`, `loop*.py`, `outcomes*` |
| [D02](#d02--llm-client-routing--providers) | LLM client, routing & providers | 38 | `ouroboros/llm*.py`, `request_wire_*`, `provider_models.py` |
| [D03](#d03--context-assembly-fit--compaction) | Context assembly, fit & compaction | 11 | `ouroboros/context*.py` |
| [D04](#d04--tool-execution-registry-access--typed-results) | Tool execution: registry, access & typed results | 21 | `ouroboros/tools/registry*`, `tool_access*` |
| [D05](#d05--tool-surfaces-files-code-shell-media-external) | Tool surfaces: files, code, shell, media, external | 28 | `ouroboros/tools/shell.py`, `browser.py`, `core_*` |
| [D06](#d06--review-stack) | Review stack | 67 | `ouroboros/review*`, `tools/plan_review*`, `commit_admission.py` |
| [D07](#d07--delegation-subagents--claudexor) | Delegation, subagents & Claudexor | 52 | `ouroboros/delegate_*`, `claudexor_*`, `subagent*` |
| [D08](#d08--supervisor-queue-workers-events--runtime-control) | Supervisor: queue, workers, events & runtime control | 46 | `supervisor/queue*.py`, `workers*.py`, `events*.py` |
| [D09](#d09--cancellation-owner-control--process-custody) | Cancellation, owner control & process custody | 13 | `cancel_intents.py`, `supervisor/task_lifecycle.py`, `task_reaper.py` |
| [D10](#d10--git-update--release-machinery) | Git, update & release machinery | 28 | `ouroboros/tools/git*`, `supervisor/update_*` |
| [D11](#d11--gateway-server--web-ui) | Gateway, server & Web UI | 54 | `server.py`, `ouroboros/gateway/*`, `server_*.py` |
| [D12](#d12--settings--configuration) | Settings & configuration | 15 | `ouroboros/config.py`, `settings_*.py`, `onboarding_*` |
| [D13](#d13--safety-guards--runtime-mode) | Safety, guards & runtime mode | 9 | `safety.py`, `runtime_mode_policy.py`, `tools/shell_guards.py` |
| [D14](#d14--skills--extensions) | Skills & extensions | 54 | `skill_*`, `extension_*`, `marketplace/*` |
| [D15](#d15--memory-knowledge-consciousness--self-evolution) | Memory, knowledge, consciousness & self-evolution | 21 | `memory.py`, `consciousness_*`, `evolution_*` |
| [D16](#d16--observability-usage-accounting--cost) | Observability, usage accounting & cost | 11 | `usage_accounting.py`, `model_send_seal.py`, `cost_projection.py` |
| [D17](#d17--projects-workspaces--task-results) | Projects, workspaces & task results | 21 | `projects_registry.py`, `workspace_*`, `task_results.py` |
| [D18](#d18--launcher-packaging-platform--shared-substrate) | Launcher, packaging, platform & shared substrate | 14 | `launcher.py`, `platform_layer.py`, `cli.py` |
| [D19](#d19--frozen-contracts-abi) | Frozen contracts (ABI) | 10 | `ouroboros/contracts/*` |
| [D20](#d20--presence) | Presence | 10 | `presence_*.py`, `tools/presence.py` |
| | **total** | **556** | |

Dependency direction matrix (strict): see `DOMAIN_MAP.md`. One pinned cycle group still spans all 20 domains (ceiling; target zero). Prefer respecting `[graph].allowed` when adding imports.

---

## D01 — Agent core & main loop

| Field | Value |
|-------|-------|
| Modules | 33 |
| Ownership | Worker-side orchestrator for a managed task: admit task dict, assemble context (via D03), run LLM↔tools loop, settle typed outcomes, emit durable results / post-task work. Does **not** own queue lock (D08) or cancel projection (D09). |
| Key prefixes | `ouroboros/agent.py`, `agent_dispatch.py`, `agent_task_pipeline.py`, `loop.py`, `loop_*.py`, `outcomes.py`, `owner_wait.py`, `task_finalization.py` |
| Key decisions | Enter tool loop vs deep-self-review bypass; dispatch axes for delegated child; startup gates; call model this round?; run tools/parallel?; acceptance fence; soft-land/force final; owner wait vs continue; terminal axes; emit results |

Cross: D08→D01 assign; D01→D03/D02/D04/D16/D06/D07/D17.

---

## D02 — LLM client, routing & providers

| Field | Value |
|-------|-------|
| Modules | 38 |
| Ownership | Physical model I/O: provider/credentials/base URL resolution, on-wire candidate build, same-route wire repair, cross-model fallback ladder, provider lanes (OpenAI-compatible, Anthropic, GigaChat, local, Claudexor). Not monetary ledger (D16) or context assembly (D03). |
| Key prefixes | `ouroboros/llm.py`, `llm_*.py`, `request_wire_*`, `provider_models.py`, `fallback_cooldown.py`, `transport_custody.py`, `vision_routing.py` |
| Key decisions | Which provider/route?; failed-account preference; physical dispatch window; model concurrency slot; cooldown skip?; **wire-repair vs fallback hop?**; transient retry/reroute; vision vs caption; loopback=local?; pre-dispatch transport death → release |

Seal mismatch is a durable fact elsewhere (D16) — D02 must not invent a second send gate.

---

## D03 — Context assembly, fit & compaction

| Field | Value |
|-------|-------|
| Modules | 11 |
| Ownership | Main-path working prompt within budget: assemble sections, measure fit vs capability evidence, reclaim/compact under typed receipts. Agent-context budgets ≠ review-prompt budgets (D06). |
| Key prefixes | `ouroboros/context.py`, `context_*.py`, `capability_evidence.py`, `main_context_authority.py`, `tools/compact_context.py` |
| Key decisions | What enters the prompt?; context mode enum; fit/overflow disposition; window claim trustworthy?; reclaim/compact?; manual compact tool; predecessor authority vs MEMORY GAP |

---

## D04 — Tool execution: registry, access & typed results

| Field | Value |
|-------|-------|
| Modules | 21 |
| Ownership | Host-owned tool execution authority between LLM loop and D05/D14: load schemas, task-local schema residency, pre-dispatch guards, access matrix, dispatch + typed `ToolResult`. |
| Key prefixes | `ouroboros/tools/registry.py`, `registry_core.py`, `registry_guards.py`, `tool_access*.py`, `tool_policy.py`, `tool_capabilities.py`, `extension_dispatch.py`, `tool_result.py` |
| Key decisions | Which schemas does this task see?; profile capability ceilings; allow op on resource root?; protected/managed-update/skill-payload write?; unknown tool → typed refusal; shell tripwire?; success/failure classification; extension vs MCP dispatch |

---

## D05 — Tool surfaces: files, code, shell, media, external

| Field | Value |
|-------|-------|
| Modules | 28 |
| Ownership | Concrete `get_tools()` handlers + libraries imported by D04 registry: files/edit/secrets, shell/process, search/code intel, browser/media/vision, MCP/services/verify. Access policy stays in D04. |
| Key prefixes | `ouroboros/tools/shell.py`, `core_file_tools.py`, `edit_ops.py`, `browser.py`, `media.py`, `mcp_client.py`, `query_code.py`, `verify.py` |
| Key decisions | run_command vs run_script cwd/backend; browser URL allowed?; listing escape root?; skill control-plane write block?; patch target protected?; MCP session; task service teardown; host-attested verify receipt |

---

## D06 — Review stack

| Field | Value |
|-------|-------|
| Modules | 67 |
| Ownership | Multi-review substrate + every surface that spends review budget. Turns plan/advisory/commit/acceptance intentions into paid, custodied reviewer waves with durable verdicts. **Sibling executor** of `run_llm_loop`. |
| Key prefixes | `ouroboros/review*.py`, `commit_admission.py`, `preflight_*.py`, `triad_review.py`, `tools/plan_review*.py`, `tools/scope_review*.py`, `tools/claude_advisory_review.py`, `tools/review*.py` |
| Key decisions | May tree spend paid review? (admission/preflight); under `OUROBOROS_REVIEW_MAX_CYCLES`?; stamp paid at physical dispatch; triad quorum PASS/FAIL; acceptance wallet mint (D17 claim); hosted-slot cancel honesty |

Cross: D06↔D16 review-wave budget; D06→D10 commit_reviewed.

---

## D07 — Delegation, subagents & Claudexor

| Field | Value |
|-------|-------|
| Modules | 52 |
| Ownership | Child work shapes outside parent main metered loop: subagent axes, configured roster, Claudexor nanny lane, durable `delegate_custody`, owned daemon. **Sibling executor**. |
| Key prefixes | `ouroboros/delegate_*.py`, `subagent*.py`, `claudexor_*.py`, `tools/delegate*.py`, `tools/control_delegation.py`, `gateways/claudexor.py` |
| Key decisions | Who may create a child? (`schedule_subagent` only); depth/budget admission; derive lane/executor; exact configured start; start Claudexor nanny; OWNED/FOREIGN/UNKNOWN custody; warm owned daemon; target drift |

---

## D08 — Supervisor: queue, workers, events & runtime control

| Field | Value |
|-------|-------|
| Modules | 46 |
| Ownership | Server-side scheduler and control bus: one supervisor tick drains events, enforces deadlines, assigns under queue lock, persists `queue_snapshot.json`, ticks consciousness. Workers run D01. |
| Key prefixes | `supervisor/queue.py`, `task_admission.py`, `worker_assignment.py`, `workers.py`, `worker_*.py`, `events.py`, `events_*.py`, `message_bus.py`, `tools/control*.py` |
| Key decisions | Start supervisor/pool?; enqueue/admit?; **assign now?** (+ project_lease); repo writer open?; dispatch event?; promote chat→task?; schedule subagent/followup; budget pause fence; idle/timeout reap; direct-chat vs pooled; **consciousness wake?** (tick only) |

Live lifecycle authority: PENDING/RUNNING under `supervisor.queue._queue_lock`. Snapshot ≠ second scheduler.

---

## D09 — Cancellation, owner control & process custody

| Field | Value |
|-------|-------|
| Modules | 13 |
| Ownership | Intent-then-custody cancellation and long-lived process identity. **Never** conflate cancel intent with canonical task status. |
| Key prefixes | `ouroboros/cancel_intents.py`, `process_custody.py`, `owner_hurry.py`, `server_control.py`, `supervisor/task_lifecycle.py`, `task_reaper.py`, `owner_stop.py`, `cancel_publication.py`, `terminal_delivery.py`, `steering.py` |
| Key decisions | Accept cancel ingress (intent only); stop policy immediate vs finalize-then-cancel; claim/settle; **cancel confirmed?** vocabulary; graceful wrap-up; steer while cancelling?; reap off-tick; deliver/salvage; spawn supervised; Panic vs Restart; hurry latch |

---

## D10 — Git, update & release machinery

| Field | Value |
|-------|-------|
| Modules | 28 |
| Ownership | VCS write + managed-update + release. Reviewed commits require D06 gates before publish. Intersects D08 repo-writer gate during updates. |
| Key prefixes | `ouroboros/tools/git*.py`, `commit_gate.py`, `github.py`, `release_sync.py`, `supervisor/git_ops*.py`, `update_*.py`, `version.py` |
| Key decisions | Advisory+tests gate proceed?; classify blocked review; max review cycles / identical-diff refusal; fingerprint revalidation; official update target; clean vs assisted merge; apply vs rollback; owner Restore; version carrier sync |

---

## D11 — Gateway, server & Web UI

| Field | Value |
|-------|-------|
| Modules | 54 |
| Ownership | HTTP/WS browser gateway + `server.py` composition root after launcher health. Chat/tasks, Projects, Skills/Widgets/marketplace, Host Service, Settings/onboarding/Presence knobs, costs, control/liveness. Presence **configured** here, **executed** in D20. |
| Key prefixes | `server.py`, `ouroboros/server_*.py`, `ouroboros/gateway/*.py`, `web/` (SPA, adjacent) |
| Key decisions | Start supervisor?; NetworkAuthGate; admit task/workspace?; onboarding complete?; mailbox vs promote vs decision-card; update preflight vs apply |

---

## D12 — Settings & configuration

| Field | Value |
|-------|-------|
| Modules | 15 |
| Ownership | Settings SSOT and setup contracts. Onboarding after gateway up (Accounts→Models→Review→Budget→final confirm); completion is one HTTP transaction through D11 writing this authority. Neither launcher nor boot may create `settings.json` (fresh-install latches). |
| Key prefixes | `ouroboros/config.py`, `settings_*.py`, `onboarding_wizard.py`, `launcher_onboarding.py`, `model_slots.py`, `runtime_limits.py`, `subscription_install_presets.py` |
| Key decisions | Payload complete for wizard steps?; structural provider ready? (`has_startup_ready_provider`); single transaction save; subscription preset eligibility; runtime mode rank after save |

---

## D13 — Safety, guards & runtime mode

| Field | Value |
|-------|-------|
| Modules | 9 |
| Ownership | Deterministic + advisory safety floor in front of tool/shell mutation. Does not execute tools (D04/D05) or own cancel (D09). |
| Key prefixes | `ouroboros/safety.py`, `runtime_mode_policy.py`, `git_shell_policy.py`, `credential_shapes.py`, `argv_budget.py`, `tools/shell_guards.py`, `write_shape.py`, `deliverables_shell.py` |
| Key decisions | Skip/check/conditional tool?; advisory allow/deny?; runtime mode allows path write?; command write-shaped?; shell guard block?; credential mutation?; argv OS budget |

---

## D14 — Skills & extensions

| Field | Value |
|-------|-------|
| Modules | 54 |
| Ownership | Discover→gate→load/exec→publish for skills and `register(api)` extensions; marketplace install provenance; event bus. State under `data/state/skills/<name>/`. |
| Key prefixes | `ouroboros/skill_*.py`, `extension_*.py`, `marketplace/*`, `tools/skill_exec.py`, `skill_preflight.py`, `event_bus.py` |
| Key decisions | **Skill may execute?** (review≠grant≠deps≠enable≠exec); aggregate review verdict; readiness blockers; review job lifecycle; exec/toggle/owner action; in-process vs isolated child; companion supervised?; marketplace install; publish eligibility |

---

## D15 — Memory, knowledge, consciousness & self-evolution

| Field | Value |
|-------|-------|
| Modules | 21 |
| Ownership | Durable self-knowledge and background autonomy: memory stores, consolidation, consciousness wakes (**supervisor-tick only**), owner-gated post-task evolution. Files are knowledge authority; gaps disclosed as loud MEMORY GAP. |
| Key prefixes | `ouroboros/memory.py`, `consolidator.py`, `knowledge.py`, `consciousness*.py`, `post_task_evolution.py`, `evolution_*.py`, `supervisor/evolution_lifecycle.py`, `tools/memory_tools.py` |
| Key decisions | Should consolidate?; MEMORY GAP vs silent reset; consciousness allowance; observe/act/full authority; wake only from tick; **promote/enqueue evolution?**; evolution lifecycle start/checkpoint/pause |

---

## D16 — Observability, usage accounting & cost

| Field | Value |
|-------|-------|
| Modules | 11 |
| Ownership | Attempt ledger = monetary authority. Seal physical send candidates. Cost projection for UI (never fabricate `$0` for unknown). `llm_usage` / `state.json` are projections carrying attempt ids — not a second charge source. |
| Key prefixes | `ouroboros/usage_accounting.py`, `usage_*.py`, `model_send_seal.py`, `cost_projection.py`, `observability.py` |
| Key decisions | Seal OK? (mismatch → fact); identity re-check refuse?; reserve_attempt budget?; settle vs unresolve vs release; compact ledger; known amount vs None for display |

---

## D17 — Projects, workspaces & task results

| Field | Value |
|-------|-------|
| Modules | 21 |
| Ownership | Where work lands: project registry (active\|deleting\|tombstoned), `project_lease` (one top-level writer per `project_id`), workspace admission, executors, headless finalize, task_results + acceptance-review cycle wallet. |
| Key prefixes | `ouroboros/projects_registry.py`, `project_lease.py`, `workspace_*.py`, `task_results.py`, `task_result_schema.py`, `task_status.py`, `headless.py`, `gateway/projects.py` (HTTP in D11) |
| Key decisions | Workspace folder usable?; create/update/delete vs tombstone; lease held?; claim acceptance review cycle / UNKNOWN; schema stamp or quarantine; finalize artifacts |

---

## D18 — Launcher, packaging, platform & shared substrate

| Field | Value |
|-------|-------|
| Modules | 14 |
| Ownership | Immutable outer shell: launcher birth, Job/pgid custody, platform primitives, bootstrap/reaper, CLI clients, cycle-free leaves (`utils`, `markdown_source`, …). |
| Key prefixes | `launcher.py`, `ouroboros/launcher_*.py`, `platform_layer.py`, `cli.py`, `packaged_cli*.py`, `node_runtime.py`, `verified_download.py` |
| Key decisions | Acquire single-instance lock?; bootstrap managed repo?; reap same-install strays?; spawn custody model?; shutdown class?; automatic launch?; CLI non-root delegation_role refuse; need local server for CLI? |

---

## D19 — Frozen contracts (ABI)

| Field | Value |
|-------|-------|
| Modules | 10 |
| Ownership | Frozen, additive ABI package: Protocols/TypedDicts/helpers that pin cross-domain shapes. Behaviour-changing code stays outside. Gate: `tests/test_contracts.py`. |
| Key prefixes | `ouroboros/contracts/plugin_api.py`, `skill_manifest.py`, `skill_payload_policy.py`, `tool_abi.py`, `tool_context.py`, `task_contract.py`, `task_constraint.py`, `chat_id_policy.py`, `schema_versions.py` |
| Key decisions | Attach/build task contract; constraint mode; PluginAPI version negotiate; manifest valid?; payload path allowed?; chat id class; schema stamp |

See also browser envelopes: `ouroboros/gateway/contracts.py` ↔ `web/modules/api_types.js` (parity-tested; separate from package ABI).

---

## D20 — Presence

| Field | Value |
|-------|-------|
| Modules | 10 |
| Ownership | Presence **side-lane**: reviewed behavior skills bound to transport rooms as fresh-agent, host-admitted path — **not** pooled Chat. |
| Key prefixes | `ouroboros/presence_admission.py`, `presence_authority.py`, `presence_bindings.py`, `presence_runner.py`, `presence_delivery.py`, `presence_profile.py`, `tools/presence.py` |
| Key decisions | Binding exists + skill reviewed/enabled?; admit_presence_turn ready?; authority ceiling; PresenceTurnGate (cap/serialization/idempotency); folder via workspace_admission?; delivery receipt mode; tools finish/cancel/configure/initiate |

---

## Related

- Spine placement → [02-runtime-spine.md](02-runtime-spine.md)
- Pipelines → [04-pipelines.md](04-pipelines.md)
- Extend safely → [06-extension-surface.md](06-extension-surface.md)
- Full module lists → upstream `DOMAIN_MAP.md`
