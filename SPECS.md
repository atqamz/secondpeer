# Peer

You code. Peer keeps up.

Repo: `secondpeer`
CLI: `peer`
Language: Go
No persona, no role-play. Pure functionality.

## Problem

AI coding agents solved "the AI writes the code".
Nothing solves "the human writes the code and the AI keeps up".

When the human drives, today's tools offer two bad options:

1. **Autocomplete-class assistants** (Copilot, editor plugins): keystroke-level, editor-locked, weak continuity, and little judgment about the shape of a work session.
2. **Chat-class agents** (claude-code, opencode, codex): powerful but usually arrive cold. They know the current conversation, not necessarily the 40 minutes of editing, failed approaches, decisions, and git state that happened before the question.

The human pays a context tax on every interaction: the assistant was not there for the work.

Peer removes that tax.
A zero-token Go daemon watches filesystem, git, and harness signals while the human codes.
It stores structured evidence and work context in SQLite: activity, current goal, direction, decisions, open questions, attempts, and outcomes.
On-demand tools (commit, review, explain, why, pr, ask) arrive pre-loaded with that context through MCP or the CLI.
A quota-gated analyzer periodically reviews both the code and the shape of the work, then records observations: bugs, missed edge cases, test gaps, repeated failed approaches, scope drift, stale assumptions, and unfinished threads.
Session briefings restore continuity: what the human was trying to do, where the work stopped, and what still deserves attention.

Peer is the inverse of secondhand.
Secondhand: AI crews write code, human approves.
Peer: human writes code, AI watches, remembers, reasons, and assists.

Peer may suggest an implementation, including complete source code or a patch-shaped answer. It never writes those suggestions into implementation files. The human remains the author who types, pastes, or otherwise applies code changes.

## Core principles

These are product contracts. They should survive changes in protocols, models, and harnesses.

1. **You code.** Peer never edits implementation files or takes ownership of implementation. It may explain, critique, sketch, or provide complete code for the human to apply. Git mutations such as commit or PR creation happen only on explicit human request.
2. **Peer keeps up.** Context is accumulated continuously so the human does not have to re-explain the work every time a harness starts or a question is asked.
3. **Peer understands the work.** Activity alone is not enough. Peer tracks the current goal, direction, decisions, open questions, attempts, outcomes, and their provenance, alongside filesystem and git evidence.
4. **Peer speaks when useful.** Relevance beats frequency. Critical findings may interrupt; everything else respects an explicit attention budget and prefers idle moments, batching, or the next natural turn.
5. **Peer remembers for the human, not against them.** Memory is local, structured, inspectable, attributable, and deletable. Summaries are derived conveniences, never opaque canonical truth.

## Architecture principles

1. **Watching is free.** Filesystem events, git state, session accounting, deterministic rules, and normal context queries spend zero model tokens and should run all day at negligible CPU cost.
2. **Unasked work never competes with the human.** Background AI is quota-gated. On-demand tools are never gated because the human explicitly chose the spend.
3. **Context first, interfaces second.** The daemon and SQLite context store own the intelligence. MCP is the primary harness interface and the CLI always exposes the same core operations; adapters stay thin.
4. **Harness-agnostic everywhere.** Claude Code, OpenCode, Pi, Codex, Grok, or future clients attach to the same work context. The LLM runner is a configurable command template, not a vendor SDK baked into the design.
5. **Structured memory over vibe retrieval.** Canonical memory is typed relational data with provenance and relations. Common retrieval is deterministic SQL; LLMs may compose read-only SQL against stable views when the query is genuinely semantic. No vector store is required for v1.
6. **No Peer cloud service.** State is stored locally and Peer has no telemetry/backend. Model calls follow the privacy boundary of the configured runner and are sanitized before leaving the daemon.
7. **One binary.** Daemon, MCP server, CLI, hook shims, query guard, and adapters' installer entrypoints are all `peer`. Runtime dependencies are git and optionally gh plus the configured model runner.

## Architecture overview

Context is the product. Watchers, SQLite, analyzers, MCP, and CLI are mechanisms for keeping that context accurate and useful.

```
                    human
             writes / thinks / tests
                      |
        files + git + harness signals
                      |
                      v
   +------------------------------------------------+
   | peer daemon (one per user, always running)     |
   |                                                |
   | evidence                                       |
   |   watcher + gitstate + harness signals   0 tok |
   |                                                |
   | context                                        |
   |   SQLite typed memory                    0 tok |
   |   activity / goal / direction / decisions      |
   |   questions / attempts / outcomes / relations  |
   |                                                |
   | awareness                                      |
   |   rules                                  0 tok |
   |   analyzer                    quota-gated tok  |
   |   observations + attention policy              |
   |                                                |
   | assistance                                     |
   |   commit / review / explain / why / pr / ask   |
   |                                                |
   | interfaces                                     |
   |   MCP + CLI + SSE/hooks/adapters               |
   +------------------------+-----------------------+
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          harnesses      peer CLI      notifications
```

Three modes of engagement:

1. **Ambient continuity** (proactive, near-zero tokens): activity tracking, work-context retrieval, session/away continuity, deterministic rules, briefings.
2. **Passive awareness** (proactive, quota-gated tokens): periodic review of both code changes and work patterns, producing observations subject to the attention budget.
3. **On-demand assistance** (tokens on invoke where needed): commit, review, explain, why, pr, rebase, ask, plus read-only structured-memory queries.

## Directory layout

Source repo:

```
secondpeer/
  main.go
  cmd/                      # cobra commands
    root.go
    daemon.go               # daemon run/install/uninstall/stop
    status.go
    project.go              # project add/list/remove
    connect.go              # register MCP server + adapters with harnesses
    hook.go                 # hook shim subcommands (claude/codex/grok stdin-json)
    commit.go review.go explain.go why.go pr.go ask.go rebase.go
    observe.go brief.go activity.go focus.go context.go memory.go
    config.go update.go
  internal/
    watcher/                # fsnotify tree watch, debounce, gitignore filtering
    gitstate/               # git CLI shell-outs: status, diff, log, blame
    store/                  # sqlite canonical memory + migrations + stable query views
    workcontext/            # typed facts: goal, direction, decisions, questions, attempts, outcomes
    memory/                 # read-only SQL guard, retrieval, retention, forget/reset
    narrative/              # derive compact session narrative from store
    rules/                  # mechanical observation rules (zero token)
    analyzer/               # code + work analysis, triggers, prompt assembly
    attention/              # human-level presentation state, routing, batching, interruption budget
    privacy/                # exclusion, secret redaction, outbound context inspection
    budget/                 # quota probe, reserve floor, run cap, spend accounting
    llm/                    # command-template LLM invocation (claude -p et al)
    observations/           # queue, dedup, severity, delivery policy, staleness
    mcpserver/              # MCP streamable HTTP server + stdio proxy mode
    brief/                  # session briefing assembly
    events/                 # SSE feed for adapters
    project/                # project registry, root resolution
    config/                 # config.toml load/validate
    daemonctl/              # single-instance, signals, service files
  adapters/
    claude/                 # settings.json fragment (hooks -> peer hook claude-*)
    opencode/               # plugin: peer-observe.js
    pi/                     # extension: peer-observe.ts
    codex/                  # hooks.json fragment
    grok/                   # hooks fragments
  go.mod go.sum
  AGENTS.md CLAUDE.md README.md LICENSE .gitignore
  Makefile flake.nix
  release-please-config.json .release-please-manifest.json
```

Runtime state (XDG, no repo pollution):

```
~/.config/peer/config.toml       # user configuration
~/.local/state/peer/
  peer.db                        # sqlite context store
  daemon.log                     # rotating daemon log
  daemon.pid
$XDG_RUNTIME_DIR/peer/peer.sock  # unix socket (CLI <-> daemon control)
```

Optional per-project override: `.peer.toml` in a project root (gitignored by the user if they want it private).

## CLI specification

### `peer daemon [flags]`

Run the daemon in the foreground.

```
peer daemon
peer daemon install      # write + enable systemd user unit / launchd plist
peer daemon uninstall
peer daemon stop
```

Behavior (run):
1. Enforce single instance by claiming the control socket (`net.Listen("unix", ...)` fails when another daemon holds it; the tailscaled pattern, no separate lock file).
2. Load config, open store, start watchers for all registered projects.
3. Serve MCP (streamable HTTP) on `127.0.0.1:<port>` (default 7433) and the SSE events feed.
4. Start rules engine and analyzer scheduler.
5. Exit cleanly on SIGINT/SIGTERM; SIGHUP reloads config.

`install` detects the platform: writes `~/.config/systemd/user/peer.service` + `systemctl --user enable --now peer` on Linux, `~/Library/LaunchAgents/dev.peer.daemon.plist` + `launchctl load` on macOS.

Errors:
- Another instance already running (prints its pid).
- Port in use.

---

### `peer project add [path]` / `peer project list` / `peer project remove <name>`

Register a project directory for watching. `path` defaults to the git root of the cwd.

```
peer project add
peer project add ~/code/nsr --name nsr
```

Behavior (add):
1. Resolve git toplevel; refuse non-git directories.
2. Refuse duplicates (by resolved path).
3. Persist in store; signal daemon over control socket to start watching immediately.

Auto-registration: when an MCP tool call or hook shim arrives for an unregistered git root, the daemon registers it automatically (config `projects.auto_register = true`, default true).

---

### `peer connect [harness|all]`

Register peer with a harness: MCP server entry + proactive adapter.

```
peer connect claude
peer connect all
```

Behavior:
- claude: run `claude mcp add --transport http --scope user peer http://127.0.0.1:7433/mcp`; merge hook entries into `~/.claude/settings.json` (SessionStart -> `peer hook claude session-start`, UserPromptSubmit -> `peer hook claude user-prompt-submit`, Stop -> `peer hook claude stop`, SessionEnd -> `peer hook claude session-end`); optional statusline snippet.
- opencode: add MCP entry to `~/.config/opencode/opencode.json`; install `peer-observe.js` plugin into `~/.config/opencode/plugins/`.
- pi: install `peer-observe.ts` extension; add MCP entry if pi supports MCP, otherwise extension registers thin wrapper tools that shell out to `peer` CLI.
- codex/grok: add MCP entry to their config; merge hooks fragments.
- Idempotent; `--dry-run` prints what would change; never overwrites foreign entries.

---

### `peer status [--json]`

One-screen daemon and work overview.
`--json` emits the same state machine-readably, including per-purpose spend and attention totals.

```
daemon      running (pid 8412, 2 projects, up 6h)
mcp         http://127.0.0.1:7433/mcp (3 clients)
project     nsr        branch fix-auth   dirty 4 files   last edit 2m ago
focus       fix token refresh race | explicit | direction: move ownership to SessionManager
project     yes2infra  branch main       clean           idle 3h
analyzer    standard | runway 61% (resets 2h14m) | 6/24 runs today | ~38k tokens | last run 12m ago
attention   2/4 proactive interruptions today | last 48m ago
queue       2 unresolved observations (1 warning, 1 suggestion)
```

When the reserve floor is crossed the analyzer line names the reason and the recovery time:

```
analyzer    paused: runway 22% below floor 30% (resets 1h03m) | mechanical rules active
```

When no quota probe is configured the line says so, so the missing defense is visible rather than silent:

```
analyzer    standard | no quota probe configured (runs/day only) | 6/24 runs today | last run 12m ago
```

---

### `peer focus [<text>|clear]` / `peer context [flags]`

`peer focus` exposes the current goal anchor.

```
peer focus
peer focus "fix multiplayer reconnect after race"
peer focus clear
```

With no argument it prints the active goal, whether it is explicit or inferred, its confidence, provenance, current direction, and unresolved questions.
An explicit human-set focus outranks all inferred context and remains active until cleared, superseded, or the project is forgotten.
When no explicit focus exists, inference may use harness prompts, linked issue/PR metadata, branch names, commits, and recent work evidence in that order; inferred facts always carry source and confidence.

`peer context` prints the structured current work context: goal, direction, recent decisions, open questions, attempts, outcomes, and activity summary.
`peer context --purpose <analysis|commit|review|explain|ask>` prints the exact sanitized context payload that would be supplied to that model purpose, after exclusions and secret redaction. It never invokes a model.

---

### `peer memory [--project <p>]` / `peer memory query <sql>` / `peer forget ...`

Memory is inspectable and user-owned.

```
peer memory
peer memory query 'SELECT kind, text, source_type FROM memory_facts WHERE status = "active"'
peer forget session <id>
peer forget project <name>
peer reset --all
```

`peer memory` shows retained structured facts and their provenance, not a prose blob.
`peer memory query` executes read-only SQL against documented stable memory views; see `## Structured memory and work context`.
`peer forget` deletes the selected retained memory and derived summaries. `peer reset --all` removes all Peer state after explicit confirmation (or `--yes`). Neither command touches the project working tree.

### `peer brief [--project <p>]`

Print the work briefing (same content a harness gets at SessionStart).

Sections, assembled from structured state except an optional narration pass:
1. Work context: current goal, direction, recent decisions, open questions, last outcome, likely next thread.
2. Where you left off: last work-session summary + last edited files and meaningful away gap.
3. Git: branch, dirty files, last commit (relative age), ahead/behind upstream.
4. External (requires gh, config `brief.remote_checks`): open PRs authored/assigned, CI status of last pushed branch.
5. Human-level unresolved observations. Items already presented elsewhere are labeled existing context rather than re-announced as new warnings.

### `peer observe [--min-severity <s>] [--json]` / `peer observe <id> --ack|--dismiss`

List human-level unresolved observations and their state.

```
42  warning     presented  auth.go:88 new branch on error path returns nil err
45  suggestion  pending    auth_test.go not updated alongside auth.go
peer observe 42 --ack
peer observe 45 --dismiss
```

Observation state belongs to the human, not to one harness window. Reading it from the CLI does not make another client announce it as new; clients separately track whether they have synchronized the current state.
Acknowledgement means "I saw this; keep it unresolved context." Dismissal suppresses it globally. Resolution and staleness may happen automatically when the underlying condition no longer holds.

### `peer activity [--since 40m] [--json]`

Zero-token session narrative: per-file aggregates from the store.

```
session started 09:12 (2h41m ago), away 27m (11:02-11:29)
auth.go        14 saves  +122/-31   09:14-11:47  (active now)
auth_test.go    2 saves  +18/-2     09:31-09:40
notes.md        1 save   +40/-0     10:05
git: 2 commits this session, last "handle token refresh" 52m ago, branch fix-auth
```

---

### `peer commit [flags]`

Context-aware commit: identify a coherent logical change, stage it, generate the message, and commit.

```
peer commit
peer commit --staged          # only what is already staged
peer commit --all             # explicitly treat every eligible change as one scope
peer commit -m "override"     # skip generation
peer commit --amend
peer commit --dry-run         # print proposed grouping + staging + message, do nothing
```

Behavior:
1. Inspect staged, tracked, and untracked changes, excluding ignored and sensitive paths.
2. `--staged` uses exactly the staged scope. Otherwise derive logical change groups from diff overlap, file activity, current goal/direction, recent decisions, and session chronology.
3. If there is one high-confidence group, use it. If several unrelated groups remain or grouping confidence is low, fail closed before staging and print the proposed groups; the human can stage explicitly, use `--include`, or choose `--all`.
4. Generate the message from the selected diff + structured work context + relevant attempts/outcomes + last 10 commit subjects for style matching. One-line subject, body only when the why is not obvious.
5. Stage only the selected files (including relevant untracked files), then run normal `git commit`. User git config applies; never `--no-verify`, never `--no-gpg-sign`.
6. Sensitive-pattern files (`.env*`, `*.pem`, `*.key`, `credentials*`) and secret-scanner hits are never auto-staged; require explicit `--include <file>` and still redact their content from model payloads.
7. Print the commit hash, message, and files included.

Errors:
- Nothing to commit.
- Ambiguous logical scope without an explicit human choice.
- Not a git repo / detached state warnings.
- LLM runner failure: print the proposed scope and diffstat and refuse the commit rather than manufacture a junk message.

### `peer review [flags]`

Review uncommitted work (tokens on demand, never gated).

```
peer review                # unstaged + staged vs HEAD
peer review --staged
peer review --focus "error handling"
```

Sends the selected diff + structured work context (goal, direction, decisions, attempts, outcomes) to the LLM runner with a review prompt.
Output: findings with file:line, severity, one-line rationale.
Findings are also queued as observations with `suppress_delivery` set, so the analyzer never re-flags them and no client is told twice.

---

### `peer explain <target> [--level beginner|normal|deep]`

Teach what code does, with session context.

`target`: `path`, `path:line`, `path:start-end`, or a symbol name (resolved via `git grep -n`).
The prompt includes: the code slice, surrounding context, whether/when the human edited it this session, the active goal/direction, relevant decisions, and recent related changes.

---

### `peer why <file:line>`

Explain why a line is the way it is: `git log -L`/`git blame` + commit messages + linked PR (via gh when available) + matching structured decisions/outcomes from Peer memory, narrated by the LLM runner.

---

### `peer pr [flags]`

Create a PR with a context-aware description.

```
peer pr
peer pr --draft
peer pr --base develop
```

Behavior:
1. Require clean tree or confirm `--allow-dirty`.
2. Push current branch (`--no-push` to skip).
3. Generate title + body from branch diff vs base + typed goals/decisions/outcomes relevant across the branch lifetime + factual session/activity evidence + commit list. Body format: Summary bullets, `Fixes #N` when a linked issue is inferable (never invented), Test plan.
4. `gh pr create`; print URL.

---

### `peer rebase`

Guidance, not execution.
Analyzes current branch vs upstream/base: what diverged, conflict predictions per commit, a suggested step-by-step plan (`git rebase -i` sequence, which commits to squash/fixup based on message and diff overlap).
Prints the plan; the human runs it.

---

### `peer ask "<question>"`

Freeform question answered with current work context, git state, and relevant code evidence injected.
For questions about older work, `ask` uses the structured-memory query path rather than stuffing prose history into the prompt: a model may compose read-only SQL against the stable memory views, Peer validates and executes it locally, then the answer is grounded in the returned rows. This extra planning call is on-demand and therefore never budget-gated.

---

### `peer hook <harness> <event>`

Internal shim: reads the harness hook payload on stdin, talks to the daemon over the control socket, emits the harness-specific response on stdout.
Sub-second, no LLM calls.

```
peer hook claude session-start      # -> {"hookSpecificOutput":{"additionalContext": "<brief>"}}
peer hook claude user-prompt-submit # -> synchronize unresolved context as additionalContext
peer hook claude stop               # -> harness idle signal; never blocks stop
peer hook claude session-end        # -> update client last_seen
peer hook codex ...                 # same pattern, codex payload shapes
```

Event mapping, given that a harness client is not a work session:

| Event | Effect |
|---|---|
| SessionStart | return the briefing, register/update the client; does not start a work session |
| UserPromptSubmit | record a liveness signal; persist a sanitized user-prompt evidence row when available; synchronize unresolved observation/context state. The hook itself never waits for an LLM. |
| Stop | mark that harness client idle for delivery routing; no session accounting (Stop fires on every agent turn, not once per work session) |
| SessionEnd | update the client's `last_seen`; never closes the work session |

Raw model responses are not stored as memory by default. Project-scoped human prompt text is useful evidence for intent, so a sanitized prompt may be retained with the raw-event retention window; longer-lived intent is represented by typed facts extracted from it. Secret redaction runs before persistence as well as before outbound model calls.

---

### `peer config [get|set|edit]`, `peer update`, `peer version`

Standard. `peer update` follows the secondhand/no-mistakes/treehouse pattern (GitHub Releases, checksum verify, in-place replace, daily non-blocking version check notice).

## MCP server specification

Transport: streamable HTTP at `http://127.0.0.1:7433/mcp` (config `mcp.port`), built on the official `modelcontextprotocol/go-sdk`.
One daemon serves many concurrent harness clients. Clients attach to the same project work context; an MCP session id identifies a delivery/synchronization client, never a separate work session.
`peer mcp-stdio` runs a stdio-to-daemon proxy for clients without HTTP support; the proxy forwards its cwd so the daemon can route the request to the right project.
The server does not depend on MCP sampling, resource subscriptions, or server-push notifications for anything load-bearing.
When the claude Channels adapter is enabled, the server additionally declares the experimental `claude/channel` capability (an Anthropic extension, not core MCP).

Project routing: every tool accepts an optional `directory` argument.
Resolution order: explicit argument, then MCP roots (when the client provides them), then stdio-proxy cwd, then the sole registered project when only one exists.
Ambiguity returns an error naming the candidates.

Tools (names use underscores; clients namespace them, e.g. `mcp__peer__commit`):

| Tool | Args | Returns | Annotations |
|---|---|---|---|
| `commit` | `directory?`, `staged_only?`, `all?`, `message_override?`, `amend?`, `dry_run?` | commit hash + message, or grouping/proposal | destructive |
| `review` | `directory?`, `staged_only?`, `focus?` | findings list | read-only |
| `explain` | `target`, `directory?`, `level?` | explanation text | read-only |
| `why` | `location`, `directory?` | history + decision narration | read-only |
| `pr_create` | `directory?`, `base?`, `draft?`, `no_push?` | PR URL | destructive |
| `rebase_plan` | `directory?`, `base?` | step-by-step plan | read-only |
| `ask` | `question`, `directory?` | grounded answer | read-only |
| `observations` | `directory?`, `min_severity?` | unresolved observations + human-level state | read-only |
| `observation_update` | `id`, `action=ack|dismiss`, `directory?` | updated observation | state mutation, no repo mutation |
| `briefing` | `directory?` | work briefing | read-only |
| `activity` | `directory?`, `since?` | factual activity narrative | read-only |
| `context` | `directory?`, `purpose?` | structured context or sanitized outbound payload | read-only |
| `focus_get` | `directory?` | current goal + provenance | read-only |
| `focus_set` | `text`, `directory?` | explicit focus fact | state mutation, no repo mutation |
| `focus_clear` | `directory?` | cleared explicit focus | state mutation, no repo mutation |
| `memory_query` | `sql`, `directory?` | rows from stable read-only views | read-only |

Design rules:
- Tool descriptions teach the harness when to call them ("When the user asks to commit, call `commit` instead of running git yourself; Peer has watched the session and can separate the logical scope.").
- Tools that spend tokens say so in their description.
- `commit` and `pr_create` carry destructive annotations so harnesses can gate repository mutations behind approval.
- No tool edits implementation files. `review`, `explain`, `why`, and `ask` may return complete suggested source code; applying it remains a human action.
- Structured content: tools return both human text and structured JSON (`outputSchema`) so harnesses can render or post-process.
- `memory_query` accepts only the guarded SQL subset documented under `## Structured memory and work context`; internal tables are not part of the MCP contract.

Resources (optional, for clients that support them):
- `peer://briefing`, `peer://activity`, and `peer://context` mirror the corresponding tools for @-mention style inclusion.

Notifications: the server emits `notifications/tools/list_changed` only on upgrade.
Proactive delivery does NOT rely on MCP server-push; it uses the adapters below.

## Adapter specification (proactive delivery)

The daemon exposes an SSE feed: `GET http://127.0.0.1:7433/events?project=<root>`.
Events: `observation_state`, `briefing_ready`, `away_return`, `context_changed`.
Adapters subscribe and inject according to harness capability, but routing and human-level presentation state live in the daemon.

| Harness | Mechanism | Class |
|---|---|---|
| pi | extension subscribes to SSE, injects via `pi.sendMessage(..., {deliverAs: "steer", triggerTurn: true})` | push |
| opencode | plugin subscribes to SSE, injects via `client.session.promptAsync` | push |
| claude-code (stable) | hooks pull: SessionStart injects briefing, UserPromptSubmit synchronizes unresolved context; statusline shows queue depth ambiently | pull-at-turn |
| claude-code (preview) | Channels: MCP server declares experimental `claude/channel` capability; off by default until the feature exits research preview | push |
| codex | hooks.json pull (UserPromptSubmit stdout context); `notify` key for desktop-notification degrade | pull-at-turn |
| grok | hooks pull (event taxonomy mirrors claude; additionalContext support unconfirmed, verify before enabling) | pull-at-turn |
| any-in-herdr | optional: `herdr pane send-text` when pane agent state is idle; `herdr notification` | push (terminal) |
| none | desktop notification (`notify-send` / `osascript`) | fallback |

### Human-level observation state

A person may have several harness windows, but Peer is one peer. Observation lifecycle therefore belongs to the human:

`pending -> presented -> acknowledged -> resolved`

`dismissed` and `stale` are terminal alternatives.

- `presented` records when and where the human was first shown the finding as new.
- `acknowledged` means the human saw it but it remains unresolved context.
- `resolved` is set when the condition is verified gone; `dismissed` is an explicit human choice; `stale` means the code fingerprint changed enough that the finding can no longer be trusted.
- Client cursors synchronize state; they do not define whether the human has seen an observation.
- A different harness may receive an already-presented unresolved item as ambient context, but never re-announce it as a fresh interruption.

### Delivery policy

- critical: bypass the non-critical attention cap and immediately route to the best currently active interactive surface. Use one primary interruption, not every channel. If no interactive surface is available, fall back to desktop notification. Other clients learn it as existing unresolved context.
- warning: proactively present only while the human is idle and while the attention budget permits; otherwise queue. Multiple compatible warnings may be batched into one interruption.
- suggestion/info: never interrupt; surface at the next natural turn, briefing, or explicit `observe`.
- Staleness: before any presentation, re-verify the observation's file/work fingerprint; stale findings are dropped from delivery and marked stale.
- Dedup: fingerprint(rule/profile, project, relevant files, normalized finding); once queued, the same finding is not re-created until the old condition resolves and later genuinely recurs.

### Attention budget

Human attention is independently budgeted from model quota.
Defaults apply only to proactive non-critical interruptions:

- at most `attention.max_proactive_per_day` (default 4) per local day across all projects and surfaces;
- at least `attention.min_interval` (default 30m) between proactive interruptions;
- warnings discovered within `attention.batch_window` (default 10m) are eligible to batch;
- critical findings are exempt from the cap, but still obey the one-observation/one-primary-interruption rule.

When the budget is exhausted, warnings remain unresolved and appear in the next natural turn/briefing. Nothing is discarded merely because the human chose quiet.

## Watcher specification

- fsnotify on each project tree; recursion is hand-rolled (fsnotify has no recursive mode: walk the tree and `Add` each directory at start, `Add` new directories on Create events); honors .gitignore (plus config `watch.ignore` globs and built-in excludes: `.git` internals except the state files below, `node_modules`, build dirs).
- Debounce 400ms per path; editor atomic-save patterns (write-temp-then-rename, vim backup dance) normalized to writes by watching the directory, not the file, and re-adding watches after renames.
- macOS note: fsnotify uses kqueue (one fd per watched item), so the gitignore filter matters even more there; inotify watch exhaustion on Linux degrades that project to polling with a warning observation.
- Git state files watched directly for instant signals: `.git/HEAD`, `.git/index`, `refs/heads/`, `MERGE_HEAD`, `REBASE_HEAD`, `FETCH_HEAD`.
- Reconciliation: `git status --porcelain=v2` at most every 30s and after git events; churn via `git diff --numstat` on quiescence, never per event.
- Away detection: no signal of any kind for `session.away_threshold` (default 15m) => away; next signal => return. The watcher is one signal source among several; see `## Session continuity`.
- Watch limits: on inotify exhaustion, log, fall back to polling that project, surface a warning observation.

## Analyzer specification

Triggers (all subject to `min_interval`, default 10m, and the budget gate):
- quiescence: an edit burst (>=3 saves) followed by >=2m of silence.
- volume: >=`analysis.file_threshold` files or >=`analysis.line_threshold` changed lines since last analysis.
- staged: the index grew (imminent commit; jump the queue).
- work-pattern: deterministic evidence indicates repeated reversals, unusually long churn on the same area, or goal/diff divergence worth a semantic check.
- Never analyzes mid-burst and never re-analyzes an unchanged input hash.

Input: current diff/evidence + structured work context + recent attempts/outcomes + unresolved/surfaced findings. Large diffs are capped file-by-file with an explicit truncation note. Secret/excluded content is sanitized before the runner sees it.
Output contract: JSON array of `{severity, category, title, detail, file?, line?, confidence, evidence_refs[]}`; findings below `analysis.min_confidence` (default 0.6) are dropped; at most `analysis.max_observations` (default 5) queued per run.

Two observation families are first-class:

- **Code observations:** potential bugs, missed edge cases, test gaps, style drift against surrounding code, dead/leftover debug code, contract mismatches.
- **Work observations:** repeated failed/reverted approaches, scope drift away from the stated goal, an assumption invalidated by later edits, a changed contract with no subsequent verification signal, an unfinished thread before a long away period, or a likely next check that has been forgotten.

The analyzer must ground work observations in stored evidence and context facts; it cannot invent intent from vibes. Low-confidence intent inference is stored as an inferred fact with provenance, not silently promoted to truth.

Every run is gated by the budget gate below.
Intensity presets set trigger thresholds and model tier:
- off: no analyzer, mechanical rules and structured context only.
- light: staged + high-signal work-pattern triggers, small model.
- standard (default): all triggers, small model.
- eager: all triggers, tighter intervals, mid model.

## Budget gate

The scarce resource under the default cost model is not money, it is the human's own rolling model quota.
Peer shells out to the locally installed `claude` binary, so a background analysis run consumes the same subscription window the human's own coding sessions draw on.
The gate exists so peer's unasked work is never the reason the human runs out mid-task.
Money remains a concern only in `api:anthropic` runner mode, where calls are metered per token.

`internal/budget` owns the probe, the gate, and spend accounting.
It shares the command-template execution helper with `internal/llm` rather than duplicating it.

### Quota probe

`budget.quota_command` is a command template in the same style as `llm.command`.
Peer does not know how to read any vendor's local quota state; it asks a command the user configures.
Expected stdout, extra fields ignored:

```json
{"remaining_percent": 42.0, "resets_at": "2026-08-02T18:00:00Z"}
```

The probe runs with a 2s timeout and its result is cached for `budget.probe_cache` (default 60s).
A missing command, non-zero exit, unparseable output, or an out-of-range percent all mean "no reading" and are logged at most once per hour.
Without `resets_at` the gate still applies; `peer status` shows the reset time as unknown.

The default is empty, so peer ships without assuming any vendor or any installed helper.
Because that leaves the gate with no reading out of the box, `peer status` and the first daemon run print one line naming the gap.

### Gate

The gate is evaluated once immediately before each background analyzer run, and nowhere else.

- Reading present and `remaining_percent` below `budget.reserve_floor` (default 30): skip the run. The analyzer stays paused until a later probe clears the floor.
- No reading: allow the run, bounded only by the run cap.
- `budget.max_runs_per_day` (default 24, counter resets at local midnight) applies in both cases, so it also serves as the runaway backstop against a trigger bug.

Probe failure fails open to the run cap, consistent with the rule that observation-side errors degrade rather than block.
Crossing the floor while a run is in flight does not abort it.

On-demand tools (`commit`, `review`, `explain`, `why`, `pr`, `ask`) never consult the gate.
When a reading is below the floor they append one runway line to their output.
They never prompt and never refuse.

### Accounting

The runner extracts usage from its own JSON output where the mode reports it and records one row per call in the `spend` table, tagged with the purpose that requested it (analysis, commit, review, explain, ask).
Every call is recorded, on-demand ones included, so the accounting covers all of peer's spend rather than only the part it gates.
The analyzer line in `peer status` reports the analysis purpose alone; the totals are available through `peer status --json`.
On the subscription path this accounting gates nothing; it exists so `peer status` can answer what peer cost.
`analysis.daily_tokens` is enforced only in `api:anthropic` mode, where it is a real bill, and defaults to 0 meaning unlimited.

## LLM runner

All AI calls go through one command-template runner (config `llm.command`), default:

```
claude -p --output-format json --model {model} {prompt}
```

Template variables: `{model}`, `{prompt}` (or stdin).
Alternatives documented: `opencode run --format json`, `codex exec`, or `api:anthropic` mode using an API key.
Timeouts, JSON extraction, and retry-once semantics live in the runner.
The runner also extracts reported usage from the response and hands it to `internal/budget` for accounting.
Per-purpose model map: `llm.models.analysis`, `llm.models.commit`, `llm.models.review`, `llm.models.explain` (small/mid tiers by default).

## Mechanical rules (zero token)

| Rule | Fires when | Severity |
|---|---|---|
| uncommitted-nudge | dirty tracked files for > `rules.uncommitted_after` (40m) | suggestion |
| default-branch | edits while on main/master and remote exists | warning |
| conflict-markers | saved file contains `<<<<<<<` | critical |
| sensitive-file | tracked-but-sensitive pattern edited or staged | warning |
| test-sibling | source file heavily edited, matching test file untouched (lang-aware sibling map) | suggestion |
| debug-leftover | staged diff adds `console.log`/`fmt.Println`/`print(` in non-test code (heuristic list) | suggestion |
| big-stage | staged file > 1MB or > 5k lines | warning |
| rebase-in-progress | REBASE_HEAD present > 30m | info |

Each rule: one Go function over watcher/git state, individually toggleable in config.

## Structured memory and work context

SQLite is Peer's canonical memory, not a backing store for prose summaries.
The model should query structure when structure can answer the question.

### Memory layers

1. **Evidence:** append-oriented facts Peer can observe directly: file events/aggregates, git transitions, sanitized project-scoped human harness prompts/signals, tool invocations, session/away intervals, analysis runs, observation presentations. Raw model responses are not persisted by default.
2. **Work context:** typed semantic facts describing what the human is doing: `goal`, `direction`, `decision`, `question`, `attempt`, `outcome`, `constraint`, and `convention`.
3. **Derived views/summaries:** compact briefings and narration generated from the first two layers. They are caches and may be rebuilt; they are never the sole canonical copy of an important fact.

A work-context fact carries at least:

`id, project_id, session_id?, kind, text, status, confidence, source_type, source_ref?, explicit, created_at, superseded_by?`

Relations are stored separately so facts can form a small ontology without embedding prose blobs:

`(subject_fact_id, relation, object_fact_id)` where relation is one of `supports`, `blocks`, `replaces`, `answers`, `tests`, `derived_from`, or another schema-versioned relation.

Facts are append/supersede by default. An LLM-produced consolidation may add typed facts or mark a fact as superseded with provenance; it may not silently hard-delete evidence or canonical facts. Human `forget` commands are the deletion authority.

### Provenance and precedence

When sources disagree, current context prefers:

1. explicit human-set facts (`peer focus`, explicit harness statement captured as such);
2. explicit linked issue/PR/task metadata;
3. high-confidence harness-context extraction;
4. branch/commit inference;
5. filesystem/git-pattern inference.

Inference never overwrites stronger evidence. It creates a new fact, relation, or supersession edge with confidence and source so the human can inspect why Peer believes something.

### SQL retrieval contract

Common context assembly uses deterministic prepared SQL written in Go.
For open-ended historical questions, the harness or LLM may compose SQL rather than "vibe-search" memory.
Peer exposes stable read-only views, versioned as part of the public memory-query contract:

- `memory_facts` — typed facts with project/session names and provenance;
- `memory_relations` — semantic links between facts;
- `memory_sessions` — work sessions and away intervals;
- `memory_activity` — per-file/session activity aggregates;
- `memory_observations` — current and historical human-level observation state;
- `memory_decisions` — active/superseded decision facts plus provenance.

`memory_query` is guarded:

- execute on a read-only connection with `PRAGMA query_only = ON`;
- accept one `SELECT` or `WITH ... SELECT` statement only; no PRAGMA, ATTACH, virtual-table creation, extension loading, or multiple statements;
- expose stable views, not writable/internal tables; a SQLite authorizer/allowlist denies internal tables, `sqlite_schema`, extension/file helpers, and unsafe functions even for SELECT;
- default row limit 200 and hard maximum 1000;
- query timeout default 500ms;
- return typed rows plus the schema version used.

For `peer ask` historical recall, Peer may use a two-step on-demand flow: give the model the stable view schema and question, require SQL-only output, validate/execute it, then answer from the rows. SQL generation failure degrades to deterministic context retrieval; it never broadens filesystem access.

### Consolidation ("dreaming")

Optional consolidation is taxonomy-constrained, never freeform memory gardening.
At session close or during idle budget, a model may propose structured facts such as a decision, outcome, supersession, or unanswered question from stored evidence.
Every proposal must include `kind`, `text`, `confidence`, and `evidence_refs`; invalid taxonomy or missing provenance is rejected.
Consolidation never deletes raw evidence and never decides by itself that an explicit human fact is obsolete.

### V1 memory scope

V1 prioritizes activity memory plus short-lived/project-scoped work context. Durable project conventions are supported by the schema but should only be promoted from explicit statements or repeated high-confidence evidence; there is no attempt to build a personality profile.
Embeddings/vector search are intentionally absent until SQL + git/grep are proven insufficient.

## Session continuity

Peer has exactly one kind of session.
A **work session** is a contiguous work window per project, owned by Peer, and it exists whether or not any harness is running.
Harness connections are **clients**, never sessions: several clients can attach to the same work session and structured context.
This keeps Peer's account of the day identical whether the human worked in an editor alone, in one Claude window, or in three harnesses at once.

### Work sessions

- Liveness is one `last_signal_at` per project, fed by every peer-visible signal: debounced file saves, git state changes, harness hook events, MCP tool calls, and CLI invocations.
- After `session.away_threshold` (default 15m) with no signal, the session enters `away`; it does **not** close. A return before the close threshold records the away gap and resumes the same work session.
- After `session.close_threshold` (default 2h) with no signal, the work session closes at the threshold boundary. The next signal opens a new work session. Daemon shutdown also closes the active session.
- No harness event closes a work session: a window closing is not the human stopping work.
- At close: persist factual aggregates immediately. An optional quota-gated consolidation/narration pass may add typed outcome/question facts and a derived summary, but failure leaves the factual session complete.
- "Where you left off" is the last closed work session plus the current project context; a long lunch does not invent a new session, while an afternoon-to-evening break does.

### Clients

- Table `clients`: `(id, harness, project, first_seen, last_seen, observation_cursor)`, keyed per client-and-project pair.
- Identity is the MCP session id for MCP clients, the harness payload's session id for hook shims, and fixed pseudo-id `cli` for the CLI.
- The cursor means "this client has synchronized observation state through id N". It is not the human presentation/acknowledgement state.
- A new client starts from the first unresolved observation relevant to the current work session, plus any older acknowledged-but-unresolved item that the context composer selects as relevant. Those arrive as existing context unless never presented to the human.
- Clients unseen for 7 days are pruned; a returning id reinitializes like a new client without changing human-level observation state.

### Observation persistence and presentation

- Observations are append-oriented with monotonic ids and a global lifecycle: pending, presented, acknowledged, resolved, dismissed, or stale.
- `observation_presentations` records each surface synchronization/presentation, including whether it was the primary interruption or ambient context.
- Exactly one presentation may claim `primary_interrupt = true` for a newly surfaced observation unless an explicit escalation policy later retries an unseen critical item.
- Dismissal, acknowledgement, resolution, staleness, and dedup are global human-level state.
- Analyzer suppression keys on the fingerprint of any observation ever queued until that condition is resolved; a genuinely recurring condition may create a new occurrence after resolution.
- `peer review` findings are stored with `suppress_proactive = true`: they remain queryable context and suppress analyzer duplicates, but are not proactively announced because the human just asked for the review.

### Persistence

- Persisted: work sessions/away intervals, clients, evidence/activity, typed context facts/relations, observations/presentations, analysis history, run-cap/spend counters, attention counters, and projects.
- Re-derived live, never persisted as canonical truth: current git status/diff and remote PR/CI state. Snapshots may be retained only as evidence references where needed to explain a historical fact.
- Retention defaults: raw high-volume events 14 days; sessions, facts, observations, presentations, analyses, and spend 90 days unless explicitly promoted/project-scoped; clients 7 days unseen. Vacuum weekly.
- Human forget/reset commands override retention immediately and cascade through derived summaries/relations.

## Configuration (`~/.config/peer/config.toml`)

```toml
[daemon]
port = 7433

[llm]
command = "claude -p --output-format json --model {model}"
[llm.models]
analysis = "haiku"
commit   = "haiku"
review   = "sonnet"
explain  = "sonnet"

[analysis]
intensity = "standard"     # off | light | standard | eager
daily_tokens = 0           # api:anthropic mode only; 0 = unlimited
min_interval = "10m"
min_confidence = 0.6
max_observations = 5

[budget]
quota_command = ""         # e.g. "quota-axi claude --json"; empty = no probe
reserve_floor = 30         # percent remaining; below this, background analysis pauses
max_runs_per_day = 24      # bound when no probe; also the runaway backstop
probe_cache = "60s"

[attention]
max_proactive_per_day = 4  # non-critical primary interruptions, global across projects
min_interval = "30m"
batch_window = "10m"

[session]
away_threshold = "15m"
close_threshold = "2h"

[memory]
query_row_limit = 200
query_max_rows = 1000
query_timeout = "500ms"

[privacy]
redact_secrets = true
exclude = []                # extra globs never included in model payloads

[rules]
uncommitted_after = "40m"
disabled = []

[brief]
remote_checks = true       # gh: PRs, CI

[watch]
ignore = []

[projects]
auto_register = true
```

Per-project `.peer.toml` may override `[analysis]`, `[rules]`, `[watch]`, `[brief]`, `[privacy].exclude`, and project-scoped context behavior.
`[budget]` and `[attention]` are user-global and not overridable per project: model runway and human attention are shared resources.

## State management

- Single SQLite database (`modernc.org/sqlite`, pure Go, WAL mode). Canonical tables include: `projects`, `sessions`, `away_intervals`, `events`, `file_activity`, `context_facts`, `context_relations`, `observations`, `observation_presentations`, `clients`, `analyses`, `spend`, `attention_events`, and `kv`/schema metadata.
- SQLite is intentionally the structured memory substrate. Prose summaries are derived artifacts; deleting a summary cannot delete the facts it summarized.
- Stable `memory_*` read-only views form the query contract for harness/LLM SQL. Internal table names and migrations are not exposed as API promises.
- The daemon is the only writer. CLI and hook shims request mutations over the control socket; they never open the DB for writes.
- Read-only status/activity/context/memory queries may fall back to a read-only DB connection when the daemon is down. SQL memory queries always enable `query_only` and the statement guard.
- Migrations are transactional and monotonic. Structured fact taxonomy/relation changes carry a schema version so old provenance remains interpretable.
- Atomic config edits (temp + rename). The daemon control socket is the single-instance authority; no independent application-level DB lock protocol.

## Error handling

- Fail closed on mutations: commit refuses empty scope and junk messages; pr refuses unpushed surprises; nothing destructive without an explicit human-initiated call.
- Fail open on awareness: watcher errors degrade to polling; analyzer/consolidation errors skip that pass and log; a broken adapter never blocks the daemon. Canonical evidence collection continues whenever possible.
- Hook shims always exit 0 with valid harness JSON (a broken peer must never break the human's harness); errors go to the daemon log.
- Exit codes: 0 success, 1 error, 2 usage, 3 precondition failed.
- Errors to stderr, structured output to stdout.

## Testing strategy

- Unit: debounce/coalesce, gitignore filtering, mechanical rules, work-context precedence/supersession, relation validation, narrative/context derivation, prompt assembly, commit grouping, secret redaction, config validation, observation dedup/staleness.
- Memory/SQL: migration tests; stable-view golden schemas; query guard rejects writes/PRAGMA/ATTACH/multiple statements; row/timeout bounds; typed row encoding; forget/reset cascades; derived summaries rebuild from canonical facts.
- Budget: gate table tests across reading-present, reading-absent, and floor-boundary equality; run-cap rollover at local midnight; usage extraction from recorded runner output.
- Attention/delivery: global non-critical cap, batching, min interval, critical best-surface routing, exactly one primary interruption, second client receives existing context without duplicate announcement, acknowledge/dismiss/resolve/stale transitions.
- Sessions: away does not close; return before close threshold resumes same session; crossing close threshold creates a new session on next signal; daemon shutdown closes active session; harness SessionEnd never closes work.
- Privacy: excluded files never enter outbound payloads; inline secret fixtures are redacted; `peer context --purpose` exactly matches the sanitized payload handed to the runner.
- Integration: temp git repos with scripted edit/revert/scope-drift sequences -> expected activity, context, grouping, rules, and work observations; daemon lifecycle; MCP list/call each tool; hook shims with recorded harness payloads.
- LLM calls mocked via the command template. Historical `ask` tests mock SQL-planner output including valid SELECT, invalid mutation, malformed SQL, timeout, and fallback retrieval.
- e2e (tagged): real `claude -p` smoke test for commit/context-aware answer generation, real gh mocked.

## What is NOT in scope

| Cut | Why |
|---|---|
| Editing implementation files | Definitional. Peer may return snippets, complete files, or patch-shaped suggestions, but the human applies them. |
| Autonomous task execution | That is secondhand/harness-agent territory. Peer maintains context and assists the human doing the work. |
| Keystroke-level completion / LSP autocomplete | Copilot/editor completion owns keystrokes. This is intentionally not on the Peer roadmap; adding it would blur the product boundary. |
| Multi-agent orchestration | Multiple agents/harnesses may share Peer context, but conversational/orchestrator/worker topology belongs to the harness or secondhand, not Peer. |
| Editor-specific plugins (VS Code, JetBrains) | Terminal/harness-first. Editors reach Peer through MCP/harness integrations; reconsider only for capabilities impossible through those surfaces. |
| PR review bot / CI integration | Peer reviews local work and preserves its why; CodeRabbit-class tools own the remote PR-bot lane. Remote CI status is read-only briefing evidence. |
| Embeddings / vector memory | Typed SQLite memory + SQL + git/grep are the default retrieval stack. Add embeddings only after a measured retrieval failure that relational structure cannot solve. |
| Multi-user / remote server | One daemon per user per machine, localhost only. |
| Windows | Linux + macOS first. |
| Telemetry | None, ever. |

## Dependencies

| Dependency | Purpose | Why this one |
|---|---|---|
| `spf13/cobra` | CLI subcommands | de facto standard for multi-command Go CLIs; matches secondhand |
| `fsnotify/fsnotify` | filesystem events | the primitive every mature Go watcher uses; recursion hand-rolled (upstream has none) |
| `modernc.org/sqlite` | canonical structured memory + query views | pure Go (no cgo), trivial cross-compilation; relational querying/provenance is a product feature, not incidental persistence |
| `pelletier/go-toml/v2` | config | active, fast; `gopkg.in/yaml.v3` is archived and TOML fits a hand-edited dotfile better |
| `adrg/xdg` | state/config/runtime paths | correct XDG handling across Linux/macOS |
| `modelcontextprotocol/go-sdk` | MCP server | official SDK, first-class Streamable HTTP, already tracks the 2026-07-28 spec revision; github-mcp-server is migrating to it |
| `go-git/go-git/v5/plumbing/format/gitignore` | ignore filtering for the watcher | pattern matching without shelling `git check-ignore` per event; only this subpackage, not go-git's worktree ops |
| git CLI (runtime) | all repo state queries | `git status --porcelain=v2`, `diff --numstat`, `log`; go-git's `Status()` is known-slow (open issue #181) and lazygit/gh both shell out |
| gh CLI (runtime, optional) | PRs, CI status | brief remote checks and `pr_create`; degrade gracefully when absent |

No scheduler dependency: `time.Ticker` plus a last-event timestamp covers periodic analysis and idle detection.

## Distribution

Same pipeline as secondhand/no-mistakes/treehouse: release-please, conventional commits, GitHub Releases with linux/darwin amd64+arm64 tarballs and checksums, `go install github.com/atqamz/secondpeer@latest`, nix flake, install.sh, `peer update` self-update with daily version-check notice.

## Implementation plan

### Phase 1: watch, remember, and expose context (zero tokens end-to-end)
1. `peer daemon` foreground: watcher, git state, SQLite migrations/store, work-session away/close lifecycle.
2. Structured evidence + work-context schema, stable `memory_*` views, query guard, `peer focus`, `peer context`, `peer memory`, forget/reset.
3. `peer project add/list/remove`, `peer status`, `peer activity`.
4. Mechanical rules + human-level observation lifecycle + `peer observe`.
Deliverable: run the daemon for a day; `peer activity` tells the factual story, and `peer context` tells what the human is trying to do without a prose memory file.

### Phase 2: on-demand assistance
5. Privacy sanitizer + LLM runner + context composer.
6. `peer commit` with logical grouping, `peer review`, `peer ask` including guarded SQL historical recall.
7. MCP server exposing context/focus/memory plus phase-2 verbs; `peer connect claude` and hook shims.
Deliverable: from inside a fresh Claude Code window, "commit this" uses the work Peer already observed, while a historical question can query structured memory instead of requiring a pasted recap.

### Phase 3: proactive awareness
8. Work-context inference/extraction with taxonomy + provenance; optional constrained session consolidation.
9. Analyzer: code observations + work observations, triggers, quota gate, accounting.
10. Briefing assembly + human-level delivery state + attention budget + SSE adapters + desktop fallback.
Deliverable: Peer notices both code risks and work-shape risks, taps the human usefully at most a few non-critical times a day, and never repeats the same warning merely because another harness is open.

### Phase 4: polish
11. `peer explain`, `peer why`, `peer pr`, `peer rebase` using structured decisions/outcomes where relevant.
12. OpenCode/Pi/Codex/Grok adapters, stdio proxy, statusline.
13. Docs, e2e, release pipeline.

## Design decisions

Design decisions for this project, including the LLM default and the proactive delivery default, are recorded in `DECISIONS.md`.
