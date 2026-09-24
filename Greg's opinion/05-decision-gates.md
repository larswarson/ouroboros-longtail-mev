# 05 — Decision gates

AGENT: read this when… changing admission, spend, cancel, evolution, skills, Presence, or wire recovery. Each gate lists question, owner, pass/fail consequence, related domains.

Evidence: architecture ch.02/05/06/11/12/13, domain inventory decision index, module paths below.

## How to use

- **Pass** = proceed along the happy path named in consequence.
- **Fail** = typed refusal / durable fact / fence — do not invent a bypass channel.
- Cross-link pipelines: [04-pipelines.md](04-pipelines.md).

---

## Load-bearing gates (required set)

### Review-wave budget gate

| | |
|--|--|
| **Question** | May this review wave spend another paid cycle / attempt under the review ceiling and usage ledger? |
| **Owner** | D06 `review_dispatch.stamp_review_paid_on_dispatch`, `review_cycles`, `tools/commit_gate.check_review_cycles_ceiling` / `count_paid_review_cycles`; meter D16 `usage_accounting` |
| **Pass** | Paid fact stamped at **physical dispatch**; wave runs; attempts appear on ledger |
| **Fail** | Ceiling / budget refusal; no silent unpaid “extra” wave; identical-diff free refusal where implemented |
| **Domains** | D06 ↔ D16 (adjacent D10 commit path, D12 reviewer slots) |

### Promote / enqueue evolution

| | |
|--|--|
| **Question** | May post-task improvement promote into an evolution campaign / enqueue under supervisor admission? |
| **Owner** | D15 `post_task_evolution`, `evolution_fingerprint`, `evolution_checkpoints`; D08 `supervisor/evolution_lifecycle.py`; signal file `state/post_task_evolution_request.json` |
| **Pass** | Supervisor idle tick consumes request; lifecycle start/checkpoint proceeds under fences |
| **Fail** | Dropped while `evolution_owner_stopped`; Light-mode / fingerprint / budget fences block; no private evolution scheduler |
| **Domains** | D15 ↔ D08 (D16 spend, D10 writer gate during updates) |

### Onboarding complete?

| | |
|--|--|
| **Question** | Are Accounts/Models/Review/Budget (+ fresh-install proofs) complete so settings may persist and supervisor may hot-start? |
| **Owner** | D11 `gateway/onboarding.complete`; D12 `settings_setup_contract` / wizard validators / `config` save |
| **Pass** | Single settings transaction; next-boot runtime mode + safety default + preset marker; supervisor start if provider structurally ready |
| **Fail** | Overlay remains; gateway-only mode; neither launcher nor boot invents `settings.json` |
| **Domains** | D11 → D12 → D08 |

### Owner card answer?

| | |
|--|--|
| **Question** | How does the owner resolve quiz / routing / model-wait cards for a live task? |
| **Owner** | D11 `gateway/task_decision.py`, `routing_decision.py`, `task_model_wait.py` (+ related owner routing) |
| **Pass** | Decision posted on ordinary mailbox/decision HTTP; task continues / routes / waits resume |
| **Fail** | Card remains owed; **no** shadow side-channel; cancel/steer rules still apply (D09) |
| **Domains** | D11, D08, D09, D17 |

### Assign now?

| | |
|--|--|
| **Question** | May this PENDING task claim a free worker right now? |
| **Owner** | D08 `worker_assignment.assign_tasks`; D17 `project_lease`; also cancel settle, evolution authority, repo-writer gate |
| **Pass** | Task → RUNNING mirrored into durable task result; worker starts `handle_task` |
| **Fail** | Stays PENDING (lease held, cancelled, writer gate closed, pool disabled/exhausted, budget fence, …) |
| **Domains** | D08 + D17 (+ D09/D10/D15 fences) |

### Cancel confirmed?

| | |
|--|--|
| **Question** | What is the honest custody outcome of a cancel intent (not the UI status string)? |
| **Owner** | D09 `cancel_intents` → `task_lifecycle` → `cancel_publication` / reaper / process custody |
| **Pass / outcomes** | Vocabulary: `confirmed` \| `requested` \| `failed` \| `containment_fault` \| `fault-may-still-live` |
| **Fail (anti-pattern)** | Treating canonical task **status** as cancel proof; tearing down without durable intent; silent drop of terminal outbox |
| **Domains** | D09 (cross-cuts D08 queue, D07 custody, D06 slots, D18 Job/pgid, D16 usage rows) |

### Consciousness wake?

| | |
|--|--|
| **Question** | May Background Consciousness start a wake-up this pass? |
| **Owner** | D08 supervisor tick **only** → D15 `consciousness_allowance`, `consciousness_authority`, `consciousness_wake` |
| **Pass** | Wake under 24h allowance + observe/act/full ceiling; may enqueue ordinary Main work |
| **Fail** | No wake from private threads, worker loops, or gateway alone; allowance/authority refusal |
| **Domains** | D08 → D15 (D16 ledger for allowance) |

### Skill may execute?

| | |
|--|--|
| **Question** | Are all independent skill gates satisfied for execution? |
| **Owner** | D14 `skill_loader.review_status_allows_execution` / grants; `skill_readiness.SkillReadiness`; exec via `tools/skill_exec` / extension runner |
| **Pass** | `available_for_execution` / readiness ready; script or extension tool runs under caps |
| **Fail** | Blockers split agent-fixable vs owner-action; `enabled=true` alone is **not** ready |
| **Domains** | D14 (+ D19 manifests, D06-like skill review, D11 catalogue UI) |

**Stages stay independent:** discover ≠ preflight ≠ review ≠ grant ≠ deps ≠ enable ≠ execute (`architecture/13-external-skills-layer.md`).

### Wire-repair vs fallback hop?

| | |
|--|--|
| **Question** | On model I/O failure, adapt the **same-route request shape** or hop the **cross-model fallback ladder**? |
| **Owner** | D02 `request_wire_recovery.plan_wire_retry_*` vs `llm_fallback` recovery ladder |
| **Pass (repair)** | Wire leaf adapts request shape only — never provider/model/API choice; learnable actions pending until exact success |
| **Pass (hop)** | Ladder moves to next model/route per cooldown and preference rules |
| **Fail** | Collapsing both into one opaque retry; treating seal mismatch as a second dispatch gate |
| **Domains** | D02 (adjacent D16 seal facts, D01 loop retry) |

### Seal mismatch as durable fact (not a second dispatch gate)

| | |
|--|--|
| **Question** | What happens when sealed physical candidate ≠ bytes about to send? |
| **Owner** | D16 `model_send_seal.persist_physical_candidate` / `verify_sealed_candidate`; in-memory identity re-check in agent loop |
| **Pass / record** | Mismatch recorded as **durable fact** for observability/accounting honesty |
| **Refuse authority** | Identity re-check **may** refuse the send — but seal mismatch itself does **not** invent a parallel “second gate” API beside normal dispatch |
| **Fail (anti-pattern)** | Silent ignore of mismatch; inventing `$0` / dropping reserved rows; double-gating that bypasses reserve/settle |
| **Domains** | D16 ↔ D01 ↔ D02 |

---

## Additional spine gates (keep consistent)

| Decision | Question | Owner | Pass | Fail | Domains |
|----------|----------|-------|------|------|---------|
| Start supervisor? | Provider structurally sufficient? | `server.py` lifespan / `_start_supervisor_if_needed` | Pool + tick start | Gateway-only / onboarding overlay | D11, D12, D08 |
| Admit task / workspace? | Folder geometry usable + slot free? | `gateway/tasks` + `workspace_admission` + queue lock | Enqueued with contract | Typed refusal (e.g. set-but-unusable) | D11, D17, D08 |
| Create child? | Host schedule path only? | D07 `schedule_subagent` | Child admitted under depth/budget | Gateway rejects lineage labels | D07, D08, D11 |
| Send to model? | Seal + route + identity OK? | D16 seal + D02 client | Provider call | Refuse / durable mismatch fact | D16, D02, D01 |
| Pay for attempt? | Reserve under ledger lock? | `usage_accounting.reserve_attempt` | Attempt row reserved | BudgetExceeded / limits | D16 |
| Tool / safety allow? | Guards then access then handler? | D13 → D04 → D05 | Tool runs | Typed ToolResult refusal | D13, D04, D05 |
| Accept review cycle? | Mint acceptance wallet at first physical reviewer dispatch? | D17 `claim_task_acceptance_review_cycle` + D06 | Cycle claimed | UNKNOWN if no terminal host run | D17, D06 |
| Presence turn? | Binding/profile/ceiling ready under cap? | D20 `admit_presence_turn` + `PresenceTurnGate` | Fresh side-lane turn | Typed refuse; not pooled Chat | D20, D14, D17, D11 |
| Shutdown class? | close / Restart / Panic? | launcher / `server_control` | Matching cleanup | Do not mix Panic reaper rules with Restart | D18, D09 |

---

## Related

- Pipelines → [04-pipelines.md](04-pipelines.md)
- Invariants → [07-invariants.md](07-invariants.md)
- Domains → [03-domain-map.md](03-domain-map.md)
