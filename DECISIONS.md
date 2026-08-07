# Decisions

This document is the single durable home for design decisions on this project.
It lives alongside `SPECS.md` so forks in direction are recorded in the repo instead of an external task queue.
A decision is recorded here with its adopted outcome, the alternatives that were weighed, and a pointer to the `SPECS.md` section that specifies the resulting behavior normatively.

## llm-default

Status: decided.
Adopted: shell out to the locally installed `claude` binary through the single command-template runner, riding the user's existing Claude subscription.

This is a cost-model fork for how every AI call reaches a model by default.
It is not analyzer-only: one command-template runner and one per-purpose model map serve analysis, commit, review, explain, and ask, so this default applies to all of them.

Alternatives weighed:

- Shell-out through the command template (adopted): no key management at all, and a harness-agnostic template keeps the default from hard-coding one vendor's CLI.
- `api:anthropic` mode of the same runner, requiring `ANTHROPIC_API_KEY`: lower latency for users who already hold a key, at the cost of per-token billing instead of riding an existing subscription.

Shell-out won because it is zero-config for a first-run user and reuses auth the user already has.
The `api:anthropic` mode stays available as an explicit opt-in, expressed as a mode of the one runner rather than a separate code path, so the invariant that all AI calls route through the command-template runner holds either way.

Normative in `SPECS.md`, section `## LLM runner`: the default `llm.command` is `claude -p --output-format json --model {model} {prompt}`, with `api:anthropic` listed among the documented alternatives.

## delivery-default

Status: decided (revised).
Adopted: observations have human-level lifecycle state, proactive delivery chooses one best surface, and non-critical interruptions are bounded by an explicit attention budget.

The previous middle policy got severity mostly right but still treated delivery too much as a client concern: critical findings pushed on every available channel and per-client cursors could make one human experience the same finding as new in several windows.
Peer is one peer even when Claude, OpenCode, Pi, and a terminal are all attached.

Adopted policy:

- critical: immediate primary interruption on the best active interactive surface, desktop fallback when no interactive surface exists; other clients receive it as existing unresolved context, not another alarm;
- warning: proactive only while idle and while the global attention budget allows, otherwise queued; compatible warnings may batch;
- suggestion/info: queue only for the next natural turn, briefing, or explicit observation read;
- observation state is global to the human: pending, presented, acknowledged, resolved, dismissed, or stale;
- client cursors record synchronization only, not whether the human has seen the item;
- default non-critical attention budget: at most 4 proactive interruptions per local day, at least 30m apart, with a 10m batch window. Critical findings are cap-exempt but still obey one-observation/one-primary-interruption.

Alternatives weighed:

- Per-client delivery state: simple for adapters, but duplicates human attention when several harnesses attach to one project.
- Push critical everywhere: maximizes reach, but turns one peer into an alarm system and violates the quiet-by-default product promise.
- Queue everything except critical: very quiet, but warnings can become invisible and fail to earn trust.
- Human-level state + best-surface routing + attention budget (adopted): preserves visibility while making interruption a scarce user resource distinct from model quota.

Normative in `SPECS.md`, sections `## Adapter specification (proactive delivery)` and `### Observation persistence and presentation`.

## budget-model

Status: decided.
Adopted: bound background analysis by the human's remaining model quota, read through an optional command-template probe, with a hard reserve floor and a runs-per-day cap.

This follows from `llm-default`.
Once every AI call rides the user's existing Claude subscription, tokens stop being the scarce resource: a background run costs no marginal money, but it draws on the same rolling quota window the human's own coding sessions need.
The original `analysis.daily_tokens = 100000` therefore measured the wrong thing.
It was neither a bill nor a meaningful fraction of any real limit, and it offered no protection against the failure that actually hurts, which is peer being the run that locks the human out mid-task.

Alternatives weighed for what the budget defends against:

- Quota contention (adopted): the harm is being locked out at the wrong moment, not the cost.
- Money: real only in the opt-in `api:anthropic` mode, so it cannot be the default model.
- A pure runaway backstop: cheap, but `min_interval` plus the unchanged-diff hash already bound the loop, so it defends against a bug rather than against the human's actual complaint.

Alternatives weighed for how remaining runway is read:

- Command-template probe (adopted): mirrors `llm.command`, so no vendor is hardwired and the user points it at whatever reads their provider. Absent or failing, it degrades to the runs-per-day cap.
- A built-in reader for Claude's local state: accurate with no configuration, but couples the daemon to undocumented per-vendor file formats that shift under CLI updates.
- Self-accounting from the runner's own output: honest about what peer spent, blind to what the human spent in the same window, which is most of it.

Alternatives weighed for the gate rule:

- Hard reserve floor (adopted): one knob, one explainable state, and mechanical rules keep running at zero tokens so a paused peer is degraded rather than dead.
- Pace-aware projection: smarter on slow days, but needs usage-history tracking and misjudges bursts.
- Graded intensity degrade: avoids a cliff, at the cost of more states to predict and test.

On-demand tools are deliberately exempt from the gate.
Refusing to write a commit message the human explicitly asked for would read as the tool malfunctioning, and it contradicts the first core principle.
Those tools instead append a runway line to their output when a reading is below the floor.

Token accounting survives as reporting rather than enforcement, so `peer status` can still answer what peer cost, and `analysis.daily_tokens` remains enforced in `api:anthropic` mode where it corresponds to a real bill.

The probe defaults to empty, which means a fresh install has no reading and falls back to the runs-per-day cap.
That is accepted rather than papered over: `peer status` and the first daemon run print a line naming the missing probe, on the view that a visible gap is better than a silent one or a hardwired vendor dependency.

Normative in `SPECS.md`, section `## Budget gate`, with the configuration surface under `## Configuration` and the reframed invariant as core principle 3.

## session-identity

Status: decided (revised).
Adopted: one watcher-owned work session per project; harness connections are clients; `away` is a state inside a session, not the session boundary.

The original decision correctly separated work sessions from MCP/harness sessions, but the normative lifecycle later contradicted itself: it both recorded away gaps inside a session and closed that session at the away threshold.

Adopted lifecycle:

- all Peer-visible signals feed one `last_signal_at` per project;
- after 15m default silence the active work session becomes away;
- returning before the 2h default close threshold records the gap and resumes the same work session;
- after 2h silence the work session closes at the threshold boundary, and the next signal starts a new one;
- daemon shutdown closes the active session;
- no harness event starts or closes a work session;
- harness/MCP clients carry observation synchronization cursors only. Human-level observation presentation/acknowledgement is global and independent of those cursors.

Alternatives weighed:

- Close at away threshold: simple, but lunch/reading/meetings fragment one coherent piece of work and make an "away gap" impossible by definition.
- No session closure until daemon shutdown: preserves continuity but collapses unrelated morning/evening work into one session.
- Two thresholds (adopted): `away` captures a meaningful pause; `close` captures a genuinely new work window.

Normative in `SPECS.md`, section `## Session continuity`.

## implementation-boundary

Status: decided.
Adopted: Peer may generate any amount of suggested code, including a complete file or patch-shaped answer, but never writes those suggestions into implementation files.

"The AI never writes code" was too strong and confused authorship with assistance.
A useful engineering peer should be able to show exactly how they would implement something. The product boundary is who owns and applies the implementation, not whether source code appears in the response.

Alternatives weighed:

- Never generate source code: preserves a hard distinction from coding agents, but unnecessarily weakens explanation and mentoring.
- Generate and auto-apply edits: useful, but crosses into delegated implementation and makes Peer compete with secondhand/agent harnesses.
- Generate freely, never apply (adopted): the human can type or paste the suggestion and remains the author of the working-tree change.

Git/workflow mutations such as staging, committing, pushing, or opening a PR remain allowed only when the human explicitly asks; those operate on human-authored changes rather than authoring implementation.

Normative in `SPECS.md`, `## Core principles`, MCP design rules, and `## What is NOT in scope`.

## work-context

Status: decided.
Adopted: intent is first-class structured context alongside activity.

A history of file saves answers what happened but not what the human is trying to accomplish.
Peer therefore tracks typed work facts such as goal, direction, decision, question, attempt, outcome, constraint, and convention, each with provenance and confidence.

Source precedence is explicit human context first, then linked task/PR metadata, harness extraction, branch/commit inference, and finally filesystem/git pattern inference.
Lower-confidence inference never overwrites stronger facts; it appends a new fact or supersession relation that remains inspectable.

`peer focus` is the minimal human override: it can set/clear the active goal without requiring the human to maintain a planning document.
`peer context` exposes the current structured state and the sanitized payload for a specific model purpose.

Alternatives weighed:

- Activity-only narrative: zero-token and reliable, but produces a smart watcher rather than a peer that understands the work.
- Freeform prose memory: expressive, but hard to query, merge, correct, attribute, or safely consolidate.
- Typed facts + provenance (adopted): enough semantics for useful continuity while staying inspectable and relational.

Normative in `SPECS.md`, sections `## Structured memory and work context`, `peer focus/context`, analyzer input, and briefings.

## structured-memory

Status: decided.
Adopted: SQLite is canonical structured memory; relational facts/evidence are source of truth, prose summaries are derived caches, and LLM/harness historical retrieval may compose guarded read-only SQL over stable views.

The memory problem should not be solved by asking a model to vibe-search or periodically rewrite a text file.
Models are good at composing SQL when given a small documented schema, and SQLite already exists in Peer as the local state substrate.

The schema separates:

- observed evidence (activity, git/harness/session signals, presentations);
- typed semantic facts with provenance/status/confidence;
- relations among facts;
- derived summaries/briefings.

Stable `memory_*` views are the public query surface. Internal tables may migrate without becoming an API contract.
`memory_query` runs `query_only`, accepts one SELECT/WITH-SELECT statement, and uses an authorizer/allowlist so generated SQL can see stable `memory_*` views but not internal tables, schema internals, file helpers, or extension-loading surfaces; rows and runtime are bounded.
Normal context assembly still uses deterministic prepared SQL in Go; LLM-generated SQL is reserved for open-ended historical questions where semantics actually help.

"Dreaming"/consolidation, if enabled, is taxonomy-constrained. A model may propose typed facts, outcomes, relations, or supersessions with evidence references; it may not silently delete canonical evidence/facts or decide that explicit human memory should disappear. Human `forget` is the deletion authority.

Alternatives weighed:

- Plain-text memory: simple and portable, but retrieval/correction becomes fuzzy and consolidation can destroy important detail.
- Vector-first memory: useful for fuzzy similarity, but hides structure Peer already knows and makes provenance/precise filtering harder.
- PostgreSQL: richer operational database, but unnecessary for a one-user local daemon and violates the one-binary/no-server shape.
- SQLite relational memory (adopted): local, zero-ops, queryable by Go and LLMs, and sufficient for the expected scale.

Normative in `SPECS.md`, `## Structured memory and work context` and `## State management`.

## privacy-boundary

Status: decided.
Adopted: Peer has no cloud service and keeps canonical state locally, but model privacy follows the configured runner; every outbound model payload is filtered and secret-redacted first.

Calling the default `claude -p` runner can send code/context to a provider, so the old phrase "local and private" was too absolute.
The accurate promise is that Peer itself has no telemetry/backend, while external model processing follows the user's selected runner and its terms.

Files matching built-in sensitive patterns or configured privacy excludes are never included in model context. A secret scanner/redactor also runs on otherwise normal source/diff text because credentials can be embedded in ordinary files. Project-scoped human harness prompts are sanitized before local persistence as evidence as well as before any later model use; raw model responses are not stored by default.
`peer context --purpose ...` lets the human inspect exactly what Peer would send after sanitization without invoking a model.

Normative in `SPECS.md`, architecture principle 6, configuration `[privacy]`, commit behavior, and privacy tests.

## attention-model

Status: decided.
Adopted: human attention is a first-class budget independent from model quota.

Quota gating protects the human's model allowance. It does not stop four technically valid warnings from interrupting four times in an afternoon.
Peer's promise is not merely "quiet by convention"; it needs enforceable limits.

The default non-critical budget is global across projects/surfaces, capped at four proactive interruptions per local day with a 30m minimum interval and a 10m batch window.
Critical findings bypass the count but not duplicate-channel suppression.
When the budget is exhausted, unresolved observations remain visible at natural turns/briefings rather than being discarded.

Normative in `SPECS.md`, `### Attention budget` and `[attention]` configuration.

## commit-scope

Status: decided.
Adopted: `peer commit` should use the context it observed to identify a coherent logical change rather than defaulting to every tracked modification.

A context-aware commit wrapper that blindly stages all tracked files leaves one of Peer's strongest advantages unused and misses untracked new source files.

Default behavior considers staged/tracked/untracked changes, excludes sensitive/ignored paths, and groups changes using diff relationships, work context, and chronology.
One high-confidence group may commit directly; several plausible unrelated groups cause a fail-closed proposal before staging. `--staged`, explicit include/staging, or `--all` express human intent when grouping is ambiguous.

Normative in `SPECS.md`, `### peer commit [flags]`.

## completion-boundary

Status: decided.
Adopted: keystroke-level/LSP autocomplete is not on the Peer roadmap.

Autocomplete has different latency, editor integration, and product gravity. Once Peer owns keystrokes it starts converging on Copilot-class coding assistance instead of owning continuity and awareness.
Peer may still return suggested code in an explicit interaction; it simply does not become the keystroke completion engine.

Likewise, Peer does not become a multi-agent orchestrator. Several harnesses or agents may query the same structured context, but conversational/orchestrator/worker topology belongs to those harnesses or to secondhand.

Normative in `SPECS.md`, `## What is NOT in scope`.

## principles-layering

Status: decided.
Adopted: separate durable product contracts from replaceable architecture principles.

"MCP-first, CLI-always" is an excellent current architecture decision but not a timeless definition of Peer. A future protocol could replace MCP without changing the product.
The durable contracts are: the human codes; Peer keeps up; Peer understands the work; Peer speaks when useful; Peer remembers for the human.
Protocol, daemon, SQL implementation details, and model-runner mechanics live below that layer.

Normative in `SPECS.md`, `## Core principles` and `## Architecture principles`.
