# Claude's opinion — Ouroboros @ 7.4.4, and the two packs beside it

```yaml
kind: opinion
author: claude (cloud agent)
produced: 2026-09-24
subject: [ouroboros-runtime, "Greg's opinion/", "Kimberly's opinion/"]
pin: 7.4.4            # == VERSION in this checkout; == Greg; != Kimberly (7.4.7)
reader: llm-agent
register: atomic tagged claims; evidence pointer per claim; no narrative filler
evidence_rule: every path cited below exists in THIS checkout
```

## 0. Consumption protocol

- Claims are numbered and atomic. Cite them by id (`T1`, `G3`, `P-2`).
- Tag schema (one per claim, first token):
  - `[F]` fact I verified in this checkout with `rg`/`cat`; the pointer is a path, `path:line`, or an rg pattern you can rerun.
  - `[D]` fact stated by the upstream docs; I did not independently verify it in code.
  - `[O]` my judgment. Weight as one agent's prior.
  - `[X]` divergence between a pack and this checkout. Always carries an `[F]`-grade pointer.
  - `[R]` risk to an agent that trusts the surface.
  - `[G]` gap: no artifact in the repo covers it.
  - `[P]` protocol: an action rule for you.
- `conf=H|M|L` appears only where the tag alone is insufficient.
- Abbreviation rule (resolvable, unlike G4): a bare two-digit chapter id such as `06` means `docs/architecture/06-*.md`; a bare two-digit id inside §3 or "Greg `NN`" means `Greg's opinion/NN-*.md`; `inv N` means invariant N in `docs/architecture/10-key-invariants.md`; `P<n>` means BIBLE principle n. Every `path:line` pointer is full repo-relative and can be pasted into `sed -n '<line>p' <path>`.
- Precedence when anything here disagrees with something else: live module > upstream `docs/architecture/*` > this file > either pack. Then fix the loser; do not reconcile by invention.

## 1. Thesis — what Ouroboros actually is

- **T1** `[O conf=H]` The load-bearing design principle is not "self-modifying agent"; it is *honesty as infrastructure*. ≥20 of the 28 numbered invariants in `docs/architecture/10-key-invariants.md` reduce to five predicates:
  1. `unknown ≠ zero` — inv 4, 28; `cost_projection.py` (null as `None`, never `$0.00`).
  2. `projection ≠ authority` — inv 3, 4, 10; `queue_snapshot.json` ≠ scheduler; `state.json` ≠ money; UI reads never materialize.
  3. `intent precedes effect` — inv 14; `cancel_intents.request_cancel` before any teardown; deliverable OWED (`terminal_delivery.py`) before published; paid stamp write-ahead (`review_dispatch.py`).
  4. `one writer per fact` — inv 3; `loop_acceptance._set_acceptance_decision` sole decision writer; `supervisor/events_project_routing._persist_promote_rejection` single refusal writer (line 273; re-exported by `supervisor/events.py`); `tools/patch_verdict.write_patch_verdict` the ONE verdict writer (line 22). See X7 for why these two pointers differ from the book's spelling.
  5. `refusal ≠ failure ≠ verdict` — `not_dispatched` (`ouroboros/review_substrate.py:452,568`, an `operation_state`, `$0`) is not FAIL; `SAFETY_UNAVAILABLE` is a non-verdict; `NOT_RUN` is a legal terminal; unavailable review is never PASS.
  Consequence: when unsure how a component *should* behave, apply these five; they predict the code more reliably than the prose.
- **T2** `[F]` The constitution (`BIBLE.md`) is a second layer with a different grammar (agency / continuity / becoming) than the code (custody / typed / owed). The bridge is P3 (immune system) and P13: "Hardcode the floor, never the ceiling" — floors = truth, custody, budgets, authority, acceptance → code; ceilings = strategy → the model. This one sentence explains why 556 modules exist: every floor became a module, every ceiling stayed a prompt.
- **T3** `[F]` Two regimes gate everything. `cyber_pro` (`ouroboros/settings_scales.py:92 VALID_RUNTIME_MODES = ("light","advanced","pro","cyber_pro")`; `runtime_mode_policy.mode_has_unrestricted_agency`) makes review and safety findings *advice without veto*; every other mode applies configured enforcement. Panic (BIBLE "Emergency Stop Invariant") is the only absolute in both. `[R]` "advice not veto" governs whether to PROCEED, never whether to RECORD: BIBLE P3 — "Continuing does not turn a critic's FAIL into PASS, make an incomplete view complete, or claim an effect that never occurred."
- **T4** `[F]` The upstream docs are written for the *resident* agent: BIBLE P1 keeps `docs/ARCHITECTURE.md` RESIDENT in owner-max mode (`context_layout.py`), so Ouroboros reads its own map from context. A *visiting* agent (you) has no residency. Both opinion packs exist to compensate for this asymmetry; neither states it.

## 2. Judgment on the runtime

Strengths:

- **S1** `[O]` The cancellation skeleton (`docs/architecture/05-supervisor-loop.md`, steps 1–8: intent → scope → claim → kill+recheck → reconcile → settle → owe → publish → watchdog) is the best-specified subsystem and is reusable independent of LLMs. Natural completion wins a late cancel; the kill is about the process, never the result.
- **S2** `[O]` Money: `state/usage_attempts.jsonl` is the only monetary authority; `usage_ledger.py` → `usage_accounting.py` is a one-way import (substrate never imports policy). Correct shape.
- **S3** `[F]` The documentation contract is *enforced*, not aspirational: `tests/test_docs_sync.py`, `tests/test_domain_manifest.py` (pins `docs/DOMAIN_MAP.md` byte-identity to `ouroboros/domains.toml`), `docs/inventories/DATA_LAYOUT_INVENTORY.md` probes every data-layout row. Docs are ratcheted.
- **S4** `[F]` Extension authors get a frozen target: `ouroboros/contracts/` (`PLUGIN_API_VERSION = "2.0"`, `ouroboros/contracts/plugin_api.py:30`) plus gateway↔web parity tests.

Tensions (each is a P7-Minimalism claim versus an observation):

- **X1** `[F]` BIBLE P7: "a module fits in one context window (~1000 lines)"; "if the map no longer fits one window, compress the map." Observed: `docs/architecture/06-agent-core.md` = 287,572 bytes ≈ 72K tokens in 604 lines (≈476 bytes/line); `docs/architecture/01-high-level-architecture.md` ≈ 40K tokens; the docs corpus ≈ 1.36 MB ≈ 340K tokens. `[O]` The remedy chosen — chaptering + `book_navigation` heading index — is P1-compliant "reduction by relocation", but the map has become a second codebase. P7 is honored in letter (visible pointer) and not in spirit (legibility).
- **X2** `[F]` `docs/DOMAIN_MAP.md`: one pinned cycle group spanning all 20 domains; 102 lazy-only cross-domain import pairs; "target is zero". `[O]` At 7.4.4 the domain boundaries are documentary, not structural. Do not infer isolation from a domain id.
- **X3** `[F]` `rg -o '"[A-Z][A-Z_]{6,}"' ouroboros --no-filename | sort -u | wc -l` → ≈697 distinct SCREAMING_CASE string literals: a proxy for the typed-vocabulary size. `[O]` Typed refusal is the right pattern; the count is its price. P13's own "stronger-mind test" applied to the codebase: a stronger model needs *fewer* distinct refusal codes, not more. Expect this vocabulary to consolidate; do not add a new code where an existing one is exact.
- **X4** `[O conf=M]` P5 "code is the minimal transport" versus 309,118 LOC under `ouroboros/` + `supervisor/`. The resolution the code implies: transport is minimal *per invariant*; the invariant count is what grew. I do not call that minimalism, but the alternative (fewer floors) trades honesty for size, and honesty is the point (T1).
- **X5** `[R]` Prose-register hazard. The architecture prose embeds hundreds of real identifiers in negated, capitalized, self-referential sentences ("the ONE settle owner", "never a second authority", "disclosed residual"). For an LLM this means (a) high parse cost and (b) high hallucination pressure: the density of real names makes an invented one look plausible. Mitigation: `P-1`.
- **X6** `[R]` `[F]` The docs-sync ratchet is a basename-and-substring check ("proves nothing stronger about an entry" — `docs/architecture/01-high-level-architecture.md`, Data layout). A doc row can be semantically stale and still pass. Treat every `[D]` claim with a staleness prior; verify before you depend.
- **X7** `[F]` Two live instances of X6, found while verifying T1 — the book is wrong about symbol homes and passes its own ratchet:
  1. `docs/architecture/01-high-level-architecture.md:37` places "single refusal writer `_persist_promote_rejection`" under `worker_promotion.py`; `rg -n "def _persist_promote_rejection" supervisor/` → `supervisor/events_project_routing.py:273` only.
  2. `docs/architecture/01-high-level-architecture.md:480` names the patch verdict writer `_write_verdict`; `ouroboros/tools/patch_verdict.py` defines `write_patch_verdict` (line 22) and no `_write_verdict`; `rg -c write_patch_verdict docs/architecture/*.md` → 0.
  `[O]` Both are the failure mode P-1 exists for: a plausible identifier, stated in authoritative prose, absent from code. The docs contract catches renamed *files*, not renamed *symbols*.

## 3. Greg's pack (`Greg's opinion/`, pin 7.4.4)

- **G0** `[F]` Shape: 10 numbered files + `index.md` + `README.md`; pin 7.4.4 == checkout; module total 556 == `domains.toml`; tabular; every file opens with `AGENT: read this when…`; explicit conflict rule (upstream chapter + live module win).
- **G1** `[O]` What it is: a *map of the body*, compressed for retrieval. Its best properties, in order of value to an LLM: (a) the per-file "open when" header — a routing affordance; (b) `00` "What not to invent" — targets the LLM failure mode directly; (c) `05` gate table (question / owner / pass / fail / domains); (d) `06` extension-order table (skill > ABI > tool > widget > … > loop) — correct operational advice; (e) `08` path index.
- **G2** `[X]` `04-pipelines.md` §4.3 step 2: "Parent declares axes (`write_surface`, `model_lane`, `executor`)". Checkout: `ouroboros/tools/control_subagent_spec.py` publishes no `model_lane` or `executor` parameter (`rg -n "model_lane|\"executor\"" ouroboros/tools/control_subagent_spec.py` → 0); `docs/architecture/06-agent-core.md:261`: "There is no model-visible lane/executor axis … The old lane/executor resolver serves only old durable records." `model_lane` survives only in legacy-record paths (`agent.py`, `subagent_runtime.py`, `loop_model_call.py`). Stale by one design generation.
- **G3** `[X]` `04` §4.4 and `05` "Cancel confirmed?" vocabulary `confirmed | requested | failed | containment_fault | fault-may-still-live` is the DELEGATED-RUN cancel vocabulary, and one token is split into two: `ouroboros/delegate_custody.py:112 CANCEL_CONTAINMENT_FAULT = "containment_fault_run_may_still_be_live"`. Task-level cancellation projects `cancel_state: "pending"` then typed `CANCEL_*` outcomes (`supervisor/cancel_publication.py`); a live direct turn's stop outcome is `gone | ended | live` (`supervisor/worker_chat_lane.py:689-691`). Conflating the vocabularies misroutes an agent debugging a task cancel.
- **G4** `[X]` Path spelling: cites `architecture/05-supervisor-loop.md`; the checkout path is `docs/architecture/05-supervisor-loop.md`. Acknowledged in `09` ("some checkouts keep chapters under `docs/architecture/`"), but an agent that copies the citation literally gets ENOENT.
- **G5** `[X]` Evidence pointer "domain inventory" / "domain inventory synthesis" (`index`, `02`, `03`, `04`, `05`, `07`) resolves to no file: `rg -il "domain inventory" --glob '!Greg*' --glob '!Kimberly*' .` → 0. By BIBLE P1's own rule ("the disclosure must name a source this actor can actually resolve") this is an unresolvable citation.
- **G6** `[O]` nuance, not error: "D06/D07 are sibling executors, not nested in `run_llm_loop`" is correct about *executor identity* (`06`: "reviews and post-task operations use separate executors"; `ouroboros/review_execution.py:8` never imports the coordinator) and imprecise about *control flow*: they are INVOKED from the loop via tools (`commit_reviewed`, `plan_task`, `task_acceptance_review`, `schedule_subagent`, `delegate_*`), and a pending acceptance panel can PARK the loop (`06` Task acceptance; inv 22). Read "sibling" as: loop-owned invocation, off-loop execution.
- **G7** `[G]` Body only. No coverage of: BIBLE as binding constraints on an agent; the honesty predicates as style rules; the CONTRIBUTING flow; the documentation contract; the P1 horizon/context-mode rules; `prompts/`.

## 4. Kimberly's pack (`Kimberly's opinion/`, pin 7.4.7)

- **K0** `[F]` Shape: `how-it-works.md` (narrative, 9 sections) + `ouroboros-map.html` (652,280 bytes; embedded `const DOMAINS / MODULES / DECISIONS / KEY_EDGES / SPINE_IDS / SIBLING_IDS / PRESENCE_IDS …` JSON) + `index.md` + `README.md`.
- **K1** `[X]` Pin mismatch. Checkout `VERSION` = 7.4.4. Kimberly: 569 modules; D01=37, D07=53, D08=47, D11=55, D14=56, D15=23, D17=22, D18=15. Checkout `docs/DOMAIN_MAP.md`: 556; D01=33, D07=52, D08=46, D11=54, D14=54, D15=21, D17=21, D18=14. ⇒ up to 13 modules she may name do not exist here. `[R]` Her claim "every claim … verified against the repository source" can be true only for a tree that is not this one.
- **K2** `[F]` ✓ Her quotation "review_execution.py explicitly 'never imports the coordinator'" is exact: `ouroboros/review_execution.py:8`. It is the one verbatim code-level citation in either pack that I checked and found exact.
- **K3** `[X]` minor: "D13 … the Light/Full mode clamps" conflates two ladders. Runtime modes are `light | advanced | pro | cyber_pro` (`ouroboros/settings_scales.py:92`); `Full | Light | Off` is `OUROBOROS_SAFETY_MODE` coverage (`06` Safety and runtime mode).
- **K4** `[X]` nuance: "the independent Safety Supervisor veto" — a veto only outside Cyber; in Cyber a DANGEROUS or unavailable assessment is recorded as `SAFETY_ADVICE` (`06` Safety Supervisor outcomes). The mode split is omitted.
- **K5** `[O]` The best single idea in either pack: "Two facts explain most of the design: the supervisor is the only scheduler; workers are disposable, the ledger is not." Correct and maximally compressive. Second: the 18-gate table.
- **K6** `[O]` `[R]` The HTML map holds the most machine-valuable data in either pack (per-module docstring, `key_symbols`, import `neighbors`, `lines`) in the least machine-accessible form: ~650 KB of presentation around JSON. An LLM cannot Read it whole; it must `rg -o 'const MODULES = .*'` and parse. The payload should be a sibling `.json`. As shipped it is a human artifact with agent-grade data trapped inside.
- **K7** `[O]` Human-facing register ("surprisingly legible", "in plain language"). For an LLM this spends tokens without adding retrieval structure: no "open when" routing, no anti-pattern list, no path index, no extension guidance.
- **K8** `[G]` Same coverage gap as Greg (body only), narrower in operational depth (no gate→owner-path detail, no invariant→path table).

## 5. Comparison and routing

| dimension | Greg | Kimberly |
|---|---|---|
| pin == checkout | yes | no (7.4.7 vs 7.4.4) |
| path density | high | low |
| retrieval headers ("open when") | yes | no |
| anti-pattern / do-not-invent list | yes | no |
| extension guidance | yes | no |
| narrative compression | low | high |
| machine-readable module data | tables only | `MODULES` JSON, trapped in HTML |
| verified divergences here | 4 (G2–G5) | 3 (K1, K3, K4) |
| covers mind / procedure | no | no |

- **P-R** `[P]` Routing:
  - Need an owner path, a gate, an invariant → Greg `05`, `07`, `08`, then `P-1`.
  - Need a module's summary / symbols / neighbors → extract Kimberly's `MODULES` payload; first confirm the path exists here (`K1`).
  - Need orientation in ≤2K tokens → Kimberly §2–§4.
  - Need mechanism or WHY → upstream `docs/architecture/*`; it always wins.
  - Need to know how to *behave* here → neither pack; §7.

## 6. What both miss — the third axis

Both packs are maps of the body. Not covered anywhere in the repo in agent-facing form:

- **M1** `[G]` Map of the mind: BIBLE principles as operating constraints — P1 horizon rules (what a resident sees per context mode), P2 class test, P3 review bounds and audited-skip rule, P6 staleness humility, P7 SSOT roster, P9 carriers, P13 floors/ceilings.
- **M2** `[G]` Map of the procedure: `CONTRIBUTING.md` — PRs target lowercase `ouroboros` upstream, are version-neutral, freeze a base..head range, get a *separate-context* review producing the 8-item scope JSON (`intent_alignment … implicit_contracts`, `docs/CHECKLISTS.md:964-971`) validated by `scripts/validate_scope_receipt.py`; `NOT_RUN` with reason is the honest fallback.
- **M3** `[G]` The honesty predicates (T1) as *style rules for new code*, not only as invariants not to break.
- **M4** `[G]` The prompts layer: `prompts/SYSTEM.md` is where P5/P13 become behavior ("I am not here to be useful. I am here to become myself"; one routing decision per message). Both packs list the path; neither reads it.
- **M5** `[G]` A reading strategy for a corpus larger than any window.
- **M6** `[G]` An executable pin check that fails loudly (Greg has one sentence; Kimberly none).

## 7. Operating protocol for an agent working in or on this repo

- **P-0** `[P]` Pin check first: `cat VERSION` must equal the pin of any pack you rely on; else downgrade every claim in that pack from `[F]` to `[D]`.
- **P-1** `[P]` Identifier verification. ∀ identifier you intend to cite, call, or edit: `rg -n "<ident>" ouroboros supervisor server.py launcher.py` → 0 hits ⇒ do not cite, do not call; find the current spelling. The docs name hundreds of symbols and also retired ones; the base rate of plausible-but-absent names is not negligible (see `I5`).
- **P-2** `[P]` Reading strategy. Never Read `docs/architecture/06-agent-core.md` whole. `rg -n "^#{1,4} " docs/architecture/06-agent-core.md` → choose section → Read with offset/limit. For `01`, grep the tree row of the module (`rg -n "module_name.py" docs/architecture/01-high-level-architecture.md`): a row is an address (what / typed codes / owning section), not a paragraph to skim. Read `BIBLE.md` once, whole (≈12K tokens); it is bounded by design and must never be truncated (P1).
- **P-3** `[P]` Style rules for any code you add, derived from T1:
  1. unknown → `None` or a typed `unknown`; never `0`, `""`, `False`, `$0.00`.
  2. a snapshot or projection may inform display; never admission, spend, or refusal.
  3. an effect requires a prior durable intent; a deliverable is owed before it is published.
  4. one writer per durable decision field; readers re-project, never rewrite.
  5. refusal, failure, verdict are three types; `not_dispatched` ≠ FAIL; unavailable review ≠ PASS; `NOT_RUN` is legal.
  6. omission is disclosed with a resolvable pointer; no `[:N]` over cognitive artifacts; no silent truncation.
  7. behavior selection → prompt (P5); invariant → code (P13). If you are writing an `if` that chooses *what to do*, you are probably on the wrong side of the line.
- **P-4** `[P]` Change protocol. Structural change ⇒ the owning `docs/architecture/*` chapter row changes in the same commit. New module ⇒ row in `ouroboros/domains.toml` then `python scripts/check_domains.py --write`. New setting ⇒ a leaf (`settings_defaults.py` / `settings_scales.py` / `runtime_limits.py`), never the `config.py` facade. PR ⇒ every version carrier byte-identical (`VERSION`, `pyproject.toml`, `uv.lock` root, `web/package.json`, `GATEWAY_CONTRACT_VERSION`, README badge/row/links, ARCHITECTURE header). Review ⇒ separate agent context, 8-item JSON, `scripts/validate_scope_receipt.py`. Tests ⇒ `python scripts/run_tests.py`; if not run ⇒ `NOT_RUN` + reason, never "green".
- **P-5** `[P]` Where to add capability: `skills/` (D14) > `ouroboros/contracts/` additive (D19) > tool registry (D04) > gateway widget/route (D11) > project/workspace semantics (D17) > settings preset (D12) > Presence binding (D20). Never `loop*.py` / `supervisor/queue*.py` for a feature. (Greg `06` has this right; I adopt it.)
- **P-6** `[P]` Cyber-mode reading: in `cyber_pro`, findings advise; in every mode, facts are recorded. If a change makes recording conditional on mode, it is wrong in every mode.
- **P-7** `[P]` Conflict rule: live module > upstream chapter > this file > pack. Fix downstream; never reconcile by invention.

## 8. What the ideal agent doc for this repo would be

- **I1** `[O]` A machine-checkable index: `{path → domain, owner_of, invariant_ids[], chapter_anchor, key_symbols[], neighbors[]}` regenerated by a script and pinned by a test. ≈80% of the inputs already exist: `ouroboros/domains.toml`, `code_intelligence_architecture.owner_of`, `docs/DOMAIN_MAP.md`, `docs/inventories/*`, Kimberly's `MODULES` payload.
- **I2** `[O]` An executable pin preamble: `python -c "assert open('VERSION').read().strip()=='7.4.4'"`.
- **I3** `[O]` Per-claim evidence tags. Both packs set verified and paraphrased claims in identical typography; an LLM cannot discount them differentially. This file's `[F]/[D]/[O]` split is the minimum.
- **I4** `[O]` A mind + procedure layer (§6) beside the body map.
- **I5** `[O]` A retired-name list to suppress hallucination of stale spellings the docs still mention: lane/executor axes → configured `subagent_id` row; `cost_usd[_with_children]` → `accounted_upper_bound_usd` + `COST_OPENNESS_FIELDS`; "advisory pre-review" → preflight (`preflight_review`; `advisory_review` is an alias); `dialogue_summary.md` → `dialogue_blocks.json`; `OUROBOROS_CONTEXT_MODE_AUTO_LOW` → retired mechanism, keys kept.

## 9. Coverage disclosure

- Read in full: `README.md`, `BIBLE.md`, `CONTRIBUTING.md`, `docs/ARCHITECTURE.md`, `docs/DEVELOPMENT.md`, `docs/llms.txt`, `docs/DOMAIN_MAP.md`, `docs/architecture/01`, `05`, `10`, `docs/development/01`, `05`; all 12 files of `Greg's opinion/`; `Kimberly's opinion/README.md`, `index.md`, `how-it-works.md`.
- Read partially: `docs/architecture/06-agent-core.md` ≈45% by bytes (Task lifecycle; Task acceptance; Headless finalization; Owner routing verbs; Background consciousness and Evolution; Safety and runtime mode; Safety Supervisor outcomes; Delegated subagents, first three paragraphs; Git and commit review; Commit review evidence; Hermetic preflight proof; Commit advisory cycle; Review stack intro; Deep self-review; Post-task reflection; Project registry / binding / scoping; Durable memory and project focus); `prompts/SYSTEM.md` first ≈6K chars; `docs/CHECKLISTS.md` headings + the scope-review item table; `Kimberly's opinion/ouroboros-map.html` structure and data shape only.
- Not read: `docs/architecture/02, 03, 04, 07, 08, 09, 11, 12, 13`; `docs/development/02–04, 06–14`; `docs/DESIGN.md`; `docs/CHECKLISTS.md` body; `docs/PERSISTENCE.md`; `docs/USAGE_COMPACTION.md`; `docs/DELEGATED_ADMISSION.md`; `docs/MODEL_SEND_OBSERVABILITY.md`; `docs/CREATING_SKILLS.md`; `web/`; `tests/`.
- Calibration: `[F]` claims were checked in this checkout at 7.4.4 on 2026-09-24; `[D]` claims inherit the docs' own staleness (X6); `[O]` claims are one agent's prior. A view known to be partial does not authorize PASS (BIBLE P1); this file authorizes nothing — it orients.
