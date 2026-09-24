# 06 — Extension surface (safe extend)

AGENT: read this when… adding product capability, a skill, widget, tool, or settings preset. Prefer this over editing the loop or supervisor.

Evidence: `architecture/11-frozen-contracts-v1.md`, `architecture/13-external-skills-layer.md`, `docs/CREATING_SKILLS.md`, `ouroboros/contracts/`, D14/D19 modules, `DOMAIN_MAP.md`.

## Prefer order (ship lane)

| Priority | Seam | Domains | Rule |
|---------:|------|---------|------|
| 1 | Skill under `skills/` or marketplace install | D14 | Vertical capability lives in a skill dir, not `loop*.py` |
| 2 | Frozen ABI + PluginAPI registration | D19 | Shapes in `ouroboros/contracts/`; silent reshape = break |
| 3 | Tool registration via registry / extension dispatch | D04 | Enter through `tools/registry*` + `extension_dispatch` |
| 4 | Gateway widget / extension route / WS | D11 | UI for the skill — not a SPA fork |
| 5 | Project / workspace semantics | D17 | Domain-shaped rooms on stock Chat |
| 6 | Settings / onboarding presets | D12 | Defaults via `config.py` leaves + onboarding |
| 7 | Presence bindings | D20 | Proactive channel on stock runner |
| — | Loop / supervisor / branded shell | D01 / D08 / D18 | **Reuse** / fork-only — not first product moves |

## What not to fork (first slice)

| Leave alone | Why |
|-------------|-----|
| `ouroboros/loop*.py`, `ouroboros/agent.py` | Cognition spine; product = tool/skill |
| `supervisor/queue*.py`, `workers.py` | Single scheduler; enqueue instead of twin |
| Wholesale `web/` SPA | Use `register_ui_tab` / widgets |
| Strip D06 / D09 / D13 | Review, cancel, safety are the immune system |
| New LLM client stack (D02) for defaults | Use D12 model slots / presets |
| Second notifier daemon | Presence (D20) + transport skill pattern |

First vertical slice success signal: job completes end-to-end on stock Chat/Projects; skill still live after one process restart; **no** second scheduler.

## Skills (D14) — core product seam

### Authoring SSOT

| Path | Why |
|------|-----|
| `docs/CREATING_SKILLS.md` | Manifest, PluginAPI, lifecycle, widgets, publish |
| `architecture/13-external-skills-layer.md` | Install→preflight→review→grants→deps→enable→execute |
| `docs/CHECKLISTS.md` (Skill Review Checklist) | Reviewer enforcement |
| `skills/telegram/` | Bundled **extension** reference |
| `skills/unix_computer_use/` | Native tool surface pattern |

### Types

| `type` | Ships | Use when |
|--------|-------|----------|
| `instruction` | Markdown-only `SKILL.md` | Playbooks / prompts |
| `script` | `scripts/` + manifest | Batch / subprocess |
| `extension` | `plugin.py` via PluginAPI | Tools, routes, WS, widgets, companions — **default product lane** |

Ownership tags: `native` · `self_authored` · `external` · `clawhub` · `ouroboroshub`. Product skills land as **external/** (or hub), not by editing launcher seeds.

### Minimal extension layout

```
skills/<your_skill>/
  SKILL.md          # name, type: extension, entry: plugin.py, plugin_api: "2.0", permissions, …
  plugin.py         # def register(api: PluginAPI): …
  lib/              # optional
  scripts/          # optional companions / workers
```

Declare `plugin_api: "2.0"` (absent field = legacy load only; new PASS refused — see `plugin_api.py` admission helpers).

### Lifecycle modules

| Concern | Path |
|---------|------|
| Discover / load | `ouroboros/skill_loader.py` — state `data/state/skills/<name>/` |
| Extension load / registry | `extension_loader.py`, `extension_registry_state.py` |
| Host PluginAPI impl | `extension_plugin_api.py` |
| Process isolation | `extension_process_runner.py`, `extension_companion.py`, `extension_isolated_deps.py`, `extension_import_staging.py` |
| Review | `skill_review*.py`, `tools/skill_preflight.py` |
| Exec scripts | `tools/skill_exec.py` |
| Marketplace | `ouroboros/marketplace/*` |
| Gateway UX | `gateway/extensions.py`, `marketplace.py`, `skill_publish.py`, `widgets.py` |

**GATE:** discover ≠ preflight ≠ review ≠ grant ≠ deps ≠ enable ≠ execute. See [05-decision-gates.md](05-decision-gates.md) **Skill may execute?**.

## Frozen ABI (D19)

Package: `ouroboros/contracts/` · doc: `architecture/11-frozen-contracts-v1.md` · gate: `tests/test_contracts.py`.

| File | Role |
|------|------|
| `plugin_api.py` | PluginAPI **2.0** — negotiate manifest before import; closed permissions |
| `skill_manifest.py` | Unified frontmatter (instruction/script/extension), presence, scheduled_tasks |
| `skill_payload_policy.py` | What a payload may contain / execute |
| `tool_abi.py` / `tool_context.py` | Tool entry + minimum tool context Protocol |
| `task_contract.py` / `task_constraint.py` | Task normalization, acceptance claims, pacing |
| `chat_id_policy.py` | Human vs synthetic chat ids (see also ch.12) |
| `schema_versions.py` | Opt-in `_schema_version` helpers |

**Browser envelopes (separate):** `ouroboros/gateway/contracts.py` ↔ `web/modules/api_types.js` — parity tests; additive fields only.

**Extend rule:** add fields keeping old consumers working; removals need versioned successor + release note. Pre-upgrade scan: `scripts/rc_audit.py`.

### PluginAPI registration surface (`plugin.py`)

| Method | Adds |
|--------|------|
| `register_tool` | Namespaced agent tool |
| `register_route` | HTTP under `/api/extensions/<skill>/…` |
| `register_ws_handler` | Namespaced WS handler |
| `register_ui_tab` | Widgets-page card |
| `register_settings_section` | Host-rendered Settings panel |
| `register_supervised_task` | In-process asyncio task (in-process mode) |
| `register_companion_process` | Manifest-declared companion subprocess |
| `subscribe_event` | Manifest-declared host events |
| `send_ws_message` | Best-effort namespaced WS broadcast |
| `get_settings` / `get_state_dir` / `skill_job_dir` / `get_runtime_info` / `log` | Runtime access |
| `on_unload` | Cleanup hook |

Execution modes: `IN_PROCESS` vs short-lived `OUT_OF_PROCESS` child — `subscribe_event` / `register_supervised_task` unavailable in OOP child; use `companion_process` for long-running work.

## Tool plane (D04/D05) — second path

Prefer PluginAPI `register_tool`. Core tools use registry ABI (`get_tools()`).

Anti-pattern: patching `loop_tool_execution.py` / `agent.py` to special-case a product tool.

## Host Service (privileged callbacks)

Loopback authenticated boundary for reviewed skills: `ouroboros/gateway/host_service.py` default port **8767**. Token: `skill_token.py` hash-bound. Frozen routes listed in `architecture/12-host-service-companions-and-chat-ids.md`.

## Catalogue ≠ Widgets (again)

- Extend catalogue chrome via skills + `gateway/extensions`
- Widgets GET is passive — register UI tabs; do not invent a second widget store

## Test / gate entry points

| Gate | Path |
|------|------|
| Frozen ABI | `tests/test_contracts.py` |
| Gateway ↔ web parity | `tests/test_gateway_parity.py` |
| Domain manifest | `tests/test_domain_manifest.py`, `scripts/check_domains.py` |
| Docs sync | `tests/test_docs_sync.py` |
| Extensions API | `tests/test_extensions_api.py` |
| OOP PluginAPI parity | `tests/test_oop_extension_parity.py` |
| Pre-upgrade ABI scan | `scripts/rc_audit.py` |

## Anti-patterns (reject in review)

| Don’t | Do instead |
|-------|------------|
| Edit `loop*.py` / `supervisor/queue*.py` for a feature | Skill tool + enqueue |
| Collapse skill review into a toggle | Keep ch.13 gate sequence |
| Silent ABI reshape | Additive field or versioned successor |
| Fork SPA for one screen | `register_ui_tab` |
| Second notifier daemon | Presence + transport skill |
| Bypass `extension_dispatch` / registry | Register through PluginAPI or `get_tools()` |
| Strip D06/D09/D13 to ship faster | Configure strictness; keep immune system |

## Related

- Domains D14/D19/D20 → [03-domain-map.md](03-domain-map.md)
- Skills pipeline → [04-pipelines.md](04-pipelines.md)
- Invariants → [07-invariants.md](07-invariants.md)
