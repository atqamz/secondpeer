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

Status: decided.
Adopted: the middle policy, where critical pushes immediately, warnings push only when the session is idle and otherwise queue, and suggestions and info queue only.

This is a product-personality fork for how loudly the daemon surfaces observations by default.
It is one configuration value either way, but the default shapes a new user's first contact with the tool.

Alternatives weighed:

- Quieter: queue everything except critical, which risks the tool reading as invisible and never earning trust.
- Middle (adopted): severity decides the channel, so the only unprompted interruption is the one the user would want interrupted for.
- Louder: warnings always push, which turns routine findings into mid-thought interruptions and trains the user to ignore them.

The middle policy won because it avoids both failure modes: the tool stays visible without interrupting a live turn for anything short of critical.

Normative in `SPECS.md`, section `## Adapter specification (proactive delivery)`, under "Delivery policy": critical pushes immediately on every available channel, warning pushes only when the session is idle (harness idle event or >=60s no activity) and otherwise queues, and suggestion/info queue only.

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

Status: decided.
Adopted: one session type, the watcher-owned work session per project, with harness connections modelled as clients that attach to it and carry their own observation cursor.

The spec previously used the word "session" for two different things.
`## Session continuity` called a session a contiguous activity window per project, derived from the watcher.
`## MCP server specification` called a session an MCP connection, distinguished per MCP session id.
Those have different lifetimes and different cardinality, since one project can have several harness windows open at once, and the store had a single `sessions` table for both.

Alternatives weighed:

- Work session only (adopted): one entity, watcher-owned, existing whether or not a harness runs. Harness signals become evidence and requests, never authority.
- Two first-class types, work sessions and harness sessions, linked: can answer what happened inside one specific window, at the cost of two lifecycles and a per-feature choice about which one is meant.
- Harness-primary, where the session is the harness connection: matches perception while a harness is open, and has no answer for editor-only work with no harness running, which is peer's core case.
- No session entity at all, only time queries over events: smallest schema, but session summaries and away/return narratives lose their anchor.

Work-session-only won because peer's account of the day must be identical whether the human worked in an editor alone, in one claude window, or in three at once.

Three consequences follow from it.

Liveness is a single `last_signal_at` per project fed by every peer-visible signal, not by filesystem events alone.
The alternative, counting only saves and git operations, would close a session during a long design conversation and record an away gap that never happened.
Per-signal-class thresholds were rejected as three knobs buying a distinction the away narrative cannot explain.

Observation delivery moves from a global drain to a per-client monotonic cursor over an append-only log.
Under the old global drain, two windows on one project meant the first prompt won and the other window never saw the observation.
Dismissal and staleness stay global, so the cursor only records who has seen what.
A new client's cursor starts at the first observation of the work session in progress: starting at zero would dump days of backlog into a fresh window, and starting at the connection instant would silently drop the warning queued a minute earlier.

The claude hook mapping was wrong and is corrected.
Stop fires every time the main agent finishes responding, not once per session, so it becomes the harness idle signal that the delivery policy already needed, and `SessionEnd` takes over updating the client record.
Neither closes a work session, because a window closing is not the human stopping work.

Normative in `SPECS.md`, section `## Session continuity`, with the hook mapping under `### peer hook <harness> <event>` and the delivery consequences under `## Adapter specification (proactive delivery)`.
