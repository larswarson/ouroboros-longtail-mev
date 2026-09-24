# Ouroboros — how it works, and what I make of it

*Faye's opinion, written for people. The same opinion in agent-facing form — every claim tagged and pointing at a file or line you can check — is [index.md](index.md). This is pinned to **7.4.4**, which is what `VERSION` says in this checkout. Every path I name exists in this tree, and every number I quote came from running a command here, not from memory.*

---

## 1. The short version

Ouroboros is an open-source agent that keeps one identity and one memory across restarts, works on your projects, runs swarms of sub-agents, and can rewrite the code it is running on — then restart into the result. That last part is what gets the attention. After reading its constitution, its architecture book and a fair amount of its source, I think it is the least interesting thing about it.

The interesting thing is that almost everything in the codebase exists to stop the system from lying to itself. Costs it does not know are shown as *unknown*, never as $0.00. A snapshot on disk is treated as a photograph of the queue, never as the queue. A cancellation writes down its intention before it kills anything. A review that did not run is recorded as *not run*, never quietly rounded up to *pass*. There are 28 numbered invariants in the architecture book, and I would say twenty of them are restatements of five rules about honesty (section 4). If you remember nothing else, remember those five; they predict how the code behaves better than the prose does.

## 2. Who is alive

```
you  (desktop app, browser, CLI)
 │
 ▼
launcher.py          the outer shell: holds the single-instance lock, spawns the
 │                   server, restarts it on exit code 42, stops trying after
 │                   five crashes in two minutes. Immutable in packaged installs.
 ▼
server.py            HTTP + WebSocket on 127.0.0.1:8765
 ├── supervisor/     ONE scheduler thread — "the tick"
 │    └── workers.py a pool of worker processes; each runs one task
 │         └── ouroboros/agent.py  →  loop.py   the model↔tools loop
 └── ouroboros/gateway/            the API the web UI and CLI talk to
```

Two roles, deliberately: the launcher owns process custody and cannot be edited by the agent; the server is the self-editable part. Workers are started with `forkserver` on Linux and `spawn` elsewhere, never forked from the multi-threaded supervisor. Every long-lived child the system starts — worker, sub-agent engine, local model, skill companion — is written to a process ledger with a fingerprint, so that after a crash the next generation can prove which processes were its own before it kills any of them.

The tick does the same things in the same order every pass: publish liveness, rotate logs, check worker health, drain events from workers and chats, take the owner's input, enforce deadlines and schedules, run maintenance, assign queued work to free workers, save a snapshot of the queue, and — last of all — tick the alarm clock for Background Consciousness. Three consecutive failed passes drop the supervisor's readiness flag and tell the owner, rather than leaving a healthy-looking server that has stopped assigning work.

Everything durable lives under `~/Ouroboros/`: `repo/` is the git checkout the agent edits, `data/` holds settings, memory, results, logs and state, `Deliverables/` is where bare filenames land, and `ouroboros.pid` is the launcher's lock.

## 3. The path a task takes

A message arrives from the web UI, the CLI (`ouroboros run …`), a schedule, or a messenger transport. The gateway validates any workspace it names (a folder that is not the agent's own repository and not its data directory), reserves a task id and a worker slot under one lock, and enqueues. The tick assigns it — unless another task already holds the write lease on that project, the budget is paused, or the worker pool is exhausted, in which case it waits and the refusal is typed.

Inside the worker, `agent.py` builds the model's view of the world — the constitution, identity, scratchpad, knowledge index, recent dialogue, the architecture map in the larger context modes, then the task — and hands it to `loop.py`. The loop is simple to describe: assemble and fit the context, reserve a row in the money ledger, call the model, settle the row, execute whatever tools the model asked for (each through a safety check, an access check, and then the handler), repeat. It ends on a typed `FINAL ANSWER:` line, a budget wall, a deadline, or a stop.

Then the honesty machinery takes over. The answer is kept as a *delivery candidate* before anything reviews it, so a reviewer crash or provider outage cannot erase useful work. The result is written to `data/task_results/<id>.json` with separate axes for lifecycle, execution health, objective, review, artifacts and child absorption — a delivered answer is not marked *failed* because one tool call errored, and a successful tool call does not make the objective *met*. The owner's final answer is registered as *owed* in a durable outbox before it is sent, so a crash between "stored" and "sent" replays the answer instead of losing it. Only then does the root task run its post-task reflection, which may update the scratchpad and knowledge notes and append to the Pattern Register (`memory/knowledge/patterns.md`).

## 4. Five rules that explain most of the code

I kept noticing the same shapes in different subsystems. Here they are, with one concrete example each.

1. **Unknown is not zero.** The cost of a task whose provider never answered is `null`, displayed as unknown. `cost_projection.py` carries explicit openness fields beside every amount so a reader can tell "we know it is $0.40" from "we do not know".
2. **A projection is not an authority.** `state/queue_snapshot.json` is a recovery aid; the live queue is the in-memory state under `_queue_lock`. `state/state.json` shows a cost total; the authority is the append-only `state/usage_attempts.jsonl`. On startup the snapshot is *not* used to resume running tasks — each surviving RUNNING row is fenced with a cancel intent and terminalized honestly.
3. **Intent comes before effect.** Every way of cancelling a task — tool, HTTP, project deletion, evolution stop — first writes one durable intent row. If that write fails, the cancel is refused; nothing is torn down without a fence a watchdog can replay. Reviews stamp "paid" before dispatch, not after. Answers are owed before they are published.
4. **One writer per fact.** The acceptance decision has a single writer function; refusals from the promotion path have a single writer; patch verdicts have a single writer. Readers re-project; they do not rewrite.
5. **A refusal, a failure and a verdict are three different things.** A reviewer that could not be reached is `not_dispatched` at $0, not a FAIL. A safety check that hit a rate limit returns `SAFETY_UNAVAILABLE`, which means "retry the same call", not "the call was dangerous". A test battery that was not run is `NOT_RUN`, never green.

If you are about to write code here and are unsure how it should behave, apply these five. You will usually land where the existing code landed.

## 5. The immune system

The constitution (`BIBLE.md`) calls the review machinery the immune system, and the name fits: it is the thing that lets a self-modifying program be trusted with its own source.

A reviewed commit goes through, in order: a cheap **preflight** (one model reading the staged change, skippable by judgment but only with an audited skip); the **hermetic test run** in a throwaway worktree and data root; the **triad** — several reviewer slots scoring the diff against `docs/CHECKLISTS.md`, with 2-of-N quorum; and the **scope review**, which is given the intent, the change and a compact repository index and then *reads the rest of the repository itself* with read-only tools, in every context mode. The commit is then bound to an exact fingerprint — tree hash, parents, `VERSION`, tag — and re-checked after it lands. Separate from all of this are **plan review** (a design reviewed before work starts) and **task acceptance** (a panel judging whether a finished task met its criteria, with evidence references that must resolve to real receipts, so a task cannot certify itself by echoing its own expected outcome).

Two design choices are worth knowing. All paid review shares one ceiling, `OUROBOROS_REVIEW_MAX_CYCLES` (default `"2"`), counted at the moment money is spent; byte-identical material is replayed for free, and a rebuttal is identified by its content hash so repeating the same argument does not buy another panel. And review enforcement is a *setting*: in `blocking` mode a critical finding stops the commit; in `advisory` mode it does not — but every decision that blocking would have stopped must leave a loud durable trace, and the constitution forbids "silent advisory" in as many words.

Above both sits **Cyber Pro**, the runtime mode in which internal review and safety findings become advice the agent may overrule. This is the most consequential switch in the system and it is easy to misread. It changes whether the agent must *stop*; it never changes what gets *recorded*. The constitution's own phrasing: continuing "does not turn a critic's FAIL into PASS, make an incomplete view complete, or claim an effect that never occurred." Below everything is **Panic**: kill every worker and process tree, stop consciousness, disable evolution and auto-resume, exit. No prompt, tool or constitutional argument may delay it.

## 6. Three ledgers: money, cancellation, memory

**Money** is one append-only file, `state/usage_attempts.jsonl`, in which every physical model call moves through reserved → dispatched → settled (or unresolved, or released). Everything else that shows a dollar figure — the dashboard, a task card, `state.json` — is a projection that carries the attempt id it was computed from. Admission decisions read the ledger under its lock; displays may show the last validated snapshot but may never *refuse* on its basis. The design is correct and I would copy it.

**Cancellation** is the best-specified subsystem in the book, and it is not really about LLMs at all. Intent → scope (single or cascade, widen-only) → one settle owner claims the intent → kill and *re-check*, because a task that finished before the kill landed keeps its result, artifacts and cost → reconcile open delegated runs → settle with an honest cost, never a fabricated $0 → owe the owner's answer → publish → a watchdog re-feeds any intent that got stranded. There is also a graceful variant (`finalize_then_cancel`) that gives the task one tool-less turn to wrap up, and a "hurry" latch that skips the next acceptance panel. Read `docs/architecture/05-supervisor-loop.md` if you read nothing else in the book.

**Memory** is a set of plain files under `data/memory/`: `identity.md` (which the agent may rewrite radically but never delete), a scratchpad of recent blocks, consolidated dialogue blocks, a shelf of Markdown knowledge notes with revision-checked writes so two concurrent thoughts cannot silently overwrite each other, and the Pattern Register and Improvement Backlog the constitution calls the body of the immune system. When the consolidator cannot find the dialogue it was supposed to summarize, it appends a loud `[MEMORY GAP]` block rather than silently resetting its cursor. Truncating any of the core files is treated as amputation, not context management.

## 7. The mind between messages

Background Consciousness is the feature the README leads with, and its implementation is admirably boring. There is no consciousness thread. The supervisor tick, as its last step, asks an alarm clock whether a wake-up is due. If it is, the system starts an *ordinary* Main-chat turn with a rendered wake-up message standing where the owner's words would be — same prompt, same tools, same money ledger, same chat log. Three things distinguish it: its origin is stamped so anything it starts is labelled as consciousness-initiated; its authority is the owner's chosen level (Observe can think and write to the owner; Act can do anything short of editing the agent's own code, evolving, restarting or changing settings; Full adds evolution); and its money is one rolling 24-hour allowance read off the usage ledger.

Evolution — the agent improving its own code on its own initiative — runs as campaigns with an explicit objective, a transaction and a claim, through the same review and commit path as any other change, and is hard-blocked in the Light runtime mode at three separate points. A worker may *recommend* an evolution cycle by writing a request file; only the supervisor's idle tick can turn that into work, and an owner's stop is sticky until the owner starts it again.

## 8. Delegation, skills, and the side-lane

There is exactly one door for creating a child agent, the `schedule_subagent` tool; the gateway rejects any caller that tries to label itself as someone's child. Children are either ordinary recursive Ouroboros agents on a chosen model, or a "nanny" supervising a delegated run on one of the owner's connected coding-subscription harnesses through a bundled engine that the installation owns. Every delegated run has durable custody — the parent can always answer *owned*, *foreign* or *unknown* about it — and recursion never widens filesystem, budget, depth, deadline or commit authority.

Skills (the plugin system, under `skills/` and `data/skills/`) pass through gates that the invariants insist stay separate: discovered, preflighted, reviewed by a hash-bound panel, granted by the owner, dependencies installed, enabled, and only then executable. A review PASS installs nothing; an `enabled` flag proves nothing about readiness. The Skills page is the real catalogue; the Widgets page is a passive projection of whatever the loader has live. Messenger integrations run as **Presence**: a reviewed behaviour skill bound to a chat room, admitted per turn under an immutable capability ceiling, executed as a fresh agent turn, delivered with typed receipts — never pooled with ordinary chat.

## 9. Where I think it strains

I like this system. I also think it is straining against its own constitution in four places, and the strain is visible from the outside.

**Size against minimalism.** Principle 7 says a module should fit in one context window and, if the architecture map no longer fits one window, the map should be compressed. The architecture book is about 745 KB across 13 chapters, and the documentation corpus around it is 1.36 MB. One chapter, `docs/architecture/06-agent-core.md`, is 287 KB in 604 lines — roughly 72,000 tokens, about 476 bytes per line. The response has been to split into chapters and give the resident agent a heading index; that is honest relocation rather than silent truncation, which is what Principle 1 demands. But the map has become a second codebase, and legibility is the thing that was promised.

**Domains that are labels.** The codebase is divided into 20 domains with a pinned dependency matrix, and the generated report says plainly that all 20 sit in one import cycle with 102 lazy cross-domain pairs, target zero. At 7.4.4 a domain id tells you where a module is filed, not what it is isolated from.

**A vocabulary that grew with the failures.** I count roughly 700 distinct upper-case string constants in the core package — a rough proxy for the size of its typed vocabulary. Typed refusals are the right pattern; each of these is a failure class that once happened and is now impossible to misreport. But the constitution's own Principle 13 says mechanisms that substitute for intelligence expire as intelligence improves, and a stronger model needs fewer distinct refusal codes, not more. I expect this to consolidate, and I would resist adding a new code where an existing one is exact.

**Prose that names things that are not there.** The architecture book is written in a dense, self-referential dialect — "the ONE settle owner", "never a second authority", "disclosed residual" — packed with real identifiers. That is a hazard for anyone reading it, and especially for a language model: when hundreds of the names are real, an invented one looks plausible. While checking my own citations I found two symbols the book names that do not exist in the code — a refusal writer attributed to the wrong module, and a verdict writer under a spelling the code does not use. The documentation tests catch renamed *files*; they do not catch renamed *symbols*. The receipts are in [index.md](index.md), items X6 and X7.

## 10. The two packs beside this one

Greg and Kimberly each wrote an agent-facing guide to this repository, and both are worth reading. They share a skeleton — the spine from launcher to loop, review and delegation as siblings of the loop, Presence as a side-lane, a table of gates, a list of invariants — and they differ in almost everything else.

**Greg's** is the better operational map. Every file opens with "read this when…", which is exactly the routing an agent needs; there is an explicit list of things not to invent; the gate table names the owning module and the pass and fail consequence; the extension guidance (put capability in a skill, then in the frozen contracts, then in a tool — never in the loop or the queue) is the right advice. Its pin matches this checkout. It is also stale in a few places I could verify: it describes sub-agent scheduling axes (`model_lane`, `executor`) that the current tool no longer exposes; its "cancel confirmed?" vocabulary is the one for *delegated runs*, presented as if it were task cancellation, with one token split in two; its chapter paths omit the `docs/` prefix; and it repeatedly cites a "domain inventory" as evidence that does not exist anywhere in this tree.

**Kimberly's** is the better read and contains the best single sentence in either pack: two facts explain most of the design — the supervisor is the only scheduler, and workers are disposable while the ledger is not. Her interactive module map holds the most useful machine data anyone produced here: every module's docstring, key symbols, import neighbours and line count. But it is pinned to 7.4.7 while this checkout is 7.4.4 — 569 modules against 556 here, with eight domain counts that disagree — so some modules she describes do not exist in this tree, and the claim that everything was verified against the source is true of a different source. And the map's data is trapped inside 650 KB of HTML that a reader has to scrape; it should have shipped as a JSON file beside the page.

| | Greg | Kimberly |
|---|---|---|
| Version pin matches this checkout | yes | no |
| Built for retrieval (open-when headers, path index) | yes | no |
| Anti-patterns and things not to invent | yes | no |
| Readable in one sitting | no | yes |
| Machine-readable module data | tables only | yes, but trapped in HTML |
| Divergences from this tree that I verified | 4 | 3 |

Neither pack covers the two things I most wanted when I arrived. Both map the *body* — processes, modules, gates. Neither maps the *mind*: the constitution as a set of constraints an agent working here is bound by, from the rule that the core memory files are always loaded in full to the rule that every commit by the resident agent is a release. And neither maps the *procedure*: how a change actually gets in. Section 11 is my attempt at both, in brief.

## 11. If you are going to work on it

Read `CONTRIBUTING.md` first; it is short and it is binding. Then `BIBLE.md` in full — it is about 12,000 tokens and is bounded by design — and `docs/CHECKLISTS.md`, because your change will be judged against it. Do not try to read the agent-core chapter end to end; grep its headings, pick the section you need, and read that.

Before you cite or call any identifier you saw in the documentation, grep for it in `ouroboros/` and `supervisor/`. If it is not there, find the current spelling; do not reproduce the book's. Code beats the book, the book beats any opinion pack including this one.

New capability goes in a skill, or as an additive field in `ouroboros/contracts/`, or as a registered tool — not in `loop*.py` and not in `supervisor/queue*.py`. A structural change updates the owning architecture chapter in the same commit; a new module needs a row in `ouroboros/domains.toml` and a regenerated domain map; a new setting belongs in one of the settings leaves, never the `config.py` facade.

A pull request to the upstream project targets the lowercase `ouroboros` branch and leaves every version carrier byte-identical — `VERSION`, `pyproject.toml`, the `uv.lock` root, `web/package.json`, the gateway contract version, the README badge and history row, the architecture header. Maintainers assign the version when they land it. The final committed range is handed to a *separate* agent context for a read-only review that covers the eight scope-checklist items and emits a JSON receipt you validate with `scripts/validate_scope_receipt.py`. Reviewing in your own conversation does not count. If the test battery did not run, the PR says `NOT_RUN` and why; it does not say green.

And if you find yourself writing an `if` statement that chooses *what to do*, stop: the constitution puts behaviour in the prompt and invariants in the code, and this is the line most contributors cross first.

## 12. What I read, and what I did not

I read the README, the constitution, the contributing guide, the architecture and development entry points, chapters 1, 5 and 10 of the architecture book in full and roughly half of chapter 6 by size, the two shortest development chapters, the domain map, and both sibling packs completely (Kimberly's HTML map by structure only). I did not read architecture chapters 2, 3, 4, 7, 8, 9, 11, 12 or 13, most of the development handbook, the design document, the checklists beyond their headings and the scope table, or any tests or front-end code. Where I make a factual claim above, I checked it in this checkout on 24 September 2026; where I express a judgment, it is one reader's, and you should weight it accordingly. A partial view does not authorize a verdict. This document orients; it does not certify.
