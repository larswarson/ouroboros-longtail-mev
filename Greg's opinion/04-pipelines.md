# 04 — End-to-end pipelines

AGENT: read this when… implementing, debugging, or describing a user/runtime flow. Pair with [05-decision-gates.md](05-decision-gates.md) for named gates.

Evidence: architecture ch.02/05/06/12/13, gateway + supervisor + presence modules, domain inventory pipelines.

## 4.1 Chat / managed task (admit → queue → agent)

```
Owner UI / CLI
  → launcher health (optional) → SPA/WS or HTTP
  → D11 gateway (ws / tasks / owner_routing)
  → D17 workspace_admission (+ attachments → task drive)
  → D08 queue.enqueue_task + task_admission (under _queue_lock)
  → D08 assign_tasks (+ D17 project_lease)
  → worker: D01 OuroborosAgent.handle_task
      → D03 context → D16 seal+reserve → D02 LLM → D04/D05 tools (D13)
      → emit_task_results → D17 task_results / artifacts
      → D15 post-task hooks
  → D11 cost_breakdown / history / WS push
```

Steps:

1. Launcher waits `/api/health`, opens UI (`server_web`).
2. If no provider: onboarding → `POST /api/onboarding/complete` (D11→D12) — **onboarding complete?**
3. Chat WS (`gateway/ws.py` → `server_owner_routing`) or `POST /api/tasks` (`gateway/tasks.py`).
4. **Admit workspace?** D17 `workspace_admission`; queue reserves id + slot.
5. Tick **assign now?** under `project_lease`; worker runs `handle_task`.
6. Per round: context → seal + reserve → provider → settle → tools → budget/acceptance.
7. Finalize `agent_task_pipeline.emit_task_results`; costs via D16.
8. Owner may hurry (`task_hurry`), answer **owner cards** (`task_decision` / `routing_decision` / `task_model_wait`), or cancel (D09).

Headless / `ouroboros run` shares admission + result seams (`headless.py`, D17).

---

## 4.2 Review gate (sibling executor)

Reviews do **not** run inside `run_llm_loop`.

1. Tool or commit path requests advisory / plan / scope / triad / acceptance / commit review (D06 tools).
2. Deterministic **commit_admission** / preflight before paid spend.
3. **Review-wave budget gate** (D06↔D16): `review_dispatch` stamps paid at **physical dispatch**; cycles via `review_cycles` / `OUROBOROS_REVIEW_MAX_CYCLES`; attempts hit usage ledger.
4. `ReviewCoordinator` / execution routes run the wave; slot cancel parallels task cancel vocabulary (`review_slot_cancel`).
5. Acceptance wallet: D17 `claim_task_acceptance_review_cycle` — **UNKNOWN** without terminal host run (no silent re-dispatch).
6. Passing commit path → D10 reviewed git / publish (`commit_gate`, `tools/git*`).

---

## 4.3 Delegation / Claudexor (sibling executor)

1. Only `schedule_subagent` (D07) may create children; gateway rejects caller lineage/subagent labels.
2. Parent declares axes (`write_surface`, `model_lane`, `executor`); lane/effort/tools derived — not a second parent ask.
3. Durable `delegate_custody*` (OWNED/FOREIGN/UNKNOWN); nanny over Claudexor (`tools/delegate.py`); worktrees; owned daemon (`claudexor_daemon.py`).
4. Parent keeps **project lease**; usage observers write D16 attempts.
5. Cancel uses D09 vocabulary over custody — never status-as-cancel.
6. Harness output is a **claim**; verification receipts stay host-authored.

---

## 4.4 Cancel / custody

Intent-then-custody:

| Concern | Authority |
|---------|-----------|
| Cancel intent | D09 `cancel_intents.request_cancel` (durable projection; never canonical status) |
| Queue settle | D09 `task_lifecycle` + D08 lock → `cancel_publication` |
| Graceful stop | D09 `owner_stop` (finalize-then-cancel / “Wrap up”) |
| Off-tick teardown | D09 `task_reaper` |
| Process ledger | D09 `process_custody` (fingerprint reap; daemon roots spared) |
| Launcher Job/pgid | D18 + `server_process.json` |
| Review slots | D06 `review_slot_cancel` / custody modules |
| Delegation | D07 `delegate_custody*` |
| Usage rows | D16 settle/unresolve/release/terminalize — never silent drop |
| Queue restore | D08 snapshot + `direct_roots` fence interrupted RUNNING as cancel intents |

**Cancel confirmed?** vocabulary: `confirmed` | `requested` | `failed` | `containment_fault` | `fault-may-still-live`.

Timeout reaping is **not** a cancel ingress — reaper keeps its own custody over the `reaping` slot marker.

---

## 4.5 Skills / extensions (catalogue vs widgets)

1. Discover/load: `skill_loader` / `extension_loader` (D14); frozen shapes D19 (`skill_manifest`, `plugin_api`).
2. **Skill may execute?** — review ≠ grant ≠ deps ≠ enable ≠ exec (`skill_readiness`).
3. Marketplace: `marketplace/*` + `gateway/marketplace` with provenance; publish `skill_publish_*`.
4. UI split:
   - **Skills catalogue** = `gateway/extensions` (index/toggle/review/grants)
   - **Widgets** = `gateway/widgets` passive cards from live loader only
5. Exec: `tools/skill_exec` / `extension_process_runner` / companions; privileged callbacks via Host Service (`gateway/host_service.py`).
6. Worker enable after boot: `state/extension_reconcile/` markers; generation in `extension_generation.json`.

Authoring: `docs/CREATING_SKILLS.md`, `architecture/13-external-skills-layer.md`. Reference layout: `skills/telegram/`.

---

## 4.6 Presence (bind → admit → run → deliver)

Side-lane — not pooled Chat:

1. Review/enable Presence skill (D14) → `presence_profile` accepts `presence:` manifest.
2. Bind room→skill (`presence_bindings`); optional folder via D17 `workspace_admission`.
3. `admit_presence_turn` → immutable ceiling (`presence_authority`, includes cognitive-memory tool baseline).
4. `presence_runner`: per-conversation serialization, installation cap, idempotent results; attaches `task_contract` + ceiling; runs fresh agent turn.
5. `presence_delivery` receipt-backed history (mode0 authored-unconfirmed / mode1 provider receipts); `tools/presence` finish/configure/initiate/cancel.
6. Host Service routes: `POST /presence/turn`, `GET /presence/work/{work_ref}`, `POST /presence/delivery` (`architecture/12-*.md`).

---

## 4.7 Memory / evolution / consciousness enqueue

| Flow | Path |
|------|------|
| Post-task memory | D01 finalization → D15 consolidator / reflection / knowledge tools |
| Loud gaps | Missing generation → MEMORY GAP block — never silent cursor reset |
| Evolution promote | Worker may write `state/post_task_evolution_request.json`; supervisor idle tick consumes (+ deletes); dropped while `evolution_owner_stopped` |
| Evolution lifecycle | D15↔D08 `supervisor/evolution_lifecycle.py` — fingerprints, checkpoints, Light-mode fences |
| Consciousness wake | **Only** D08 supervisor tick → `consciousness_allowance` + `consciousness_authority` → ordinary Main turn / optional enqueue |

Consciousness is not a free daemon thread.

---

## Related

- Topology → [01-what-runs-where.md](01-what-runs-where.md)
- Spine diagram rules → [02-runtime-spine.md](02-runtime-spine.md)
- Gate table → [05-decision-gates.md](05-decision-gates.md)
