# Peer

You code. It watches, remembers, and helps.

Repo: `secondpeer`
CLI: `peer`
Language: Go
No persona, no role-play. Pure functionality.

## Problem

AI coding agents solved "the AI writes the code".
Nothing solves "the human writes the code and the AI keeps up".

When the human drives, today's tools offer two bad options:

1. **Autocomplete-class assistants** (Copilot, editor plugins): keystroke-level, editor-locked, no session memory, no git awareness, no judgment about the shape of the work.
2. **Chat-class agents** (claude-code, opencode, codex): powerful but blank. Every session starts cold. The agent knows nothing about the 40 minutes of editing you just did unless you paste it in. Asking for a commit message means re-explaining what you changed and why.

The human pays a context tax on every interaction: the assistant was not watching.

Peer removes that tax.
A zero-token Go daemon watches the filesystem and git state while the human codes.
It accumulates a structured session narrative.
On-demand tools (commit, review, explain, why, pr, ask) arrive pre-loaded with that context through MCP, callable from any harness or the CLI.
A bounded-budget analyzer periodically reviews the human's work and queues observations: potential bugs, missed edge cases, test gaps, uncommitted-work nudges.
Session briefings restore continuity: where you left off, what happened while you were away.

Peer is the inverse of secondhand.
Secondhand: AI crews write code, human approves.
Peer: human writes code, AI watches and assists.
The AI never writes code.

## Core principles

1. **The human drives.** Peer never edits files, never commits without being asked, never opens PRs on its own. Every mutation is an explicit human request (via harness or CLI).
2. **Watching is free.** The watcher layer (filesystem events, git state, session accounting) spends zero tokens. It must be able to run all day at negligible CPU cost.
3. **Token spend is deliberate and bounded.** AI analysis runs on triggers with a daily budget. When the budget is gone, mechanical observations continue; analytical ones pause. No surprise bills.
4. **MCP-first, CLI-always.** The primary interface is an MCP server any harness connects to. The CLI exposes the same operations for harness-free use. Adapters are thin; intelligence lives in the daemon.
5. **Harness-agnostic everywhere.** Works with claude-code, opencode, pi, codex, grok. The analyzer's LLM access is a configurable command template, not a hardwired vendor SDK.
6. **Quiet by default.** Observations queue; they do not interrupt. Only severity=critical pushes immediately. The human's flow state is the most expensive resource in the room.
7. **Local and private.** All state on disk, no telemetry, no cloud component. Sensitive files (secrets, keys) are never read, never included in narratives or analysis payloads.
8. **One binary.** Daemon, MCP server, CLI, hook shims: all `peer`. No shell scripts, no runtime dependencies beyond git (and optionally gh).

## Architecture overview

```
                 human (writes all code, any editor)
                    |
        edits files / runs git
                    |
                    v
   +--------------------------------------------+
   | peer daemon (one per user, always running) |
   |                                            |
   |  watcher      fs events + git state        |  0 tokens
   |  store        sqlite session log           |  0 tokens
   |  rules        mechanical observations      |  0 tokens
   |  analyzer     periodic AI review           |  budgeted tokens
   |  observations queue + delivery policy      |
   |  MCP server   streamable HTTP, localhost   |
   |  events feed  SSE for adapters             |
   +---+----------------+----------------+------+
       |                |                |
       v                v                v
  MCP tools        adapters          peer CLI
  (any harness)    (push/pull        (harness-free)
  commit, review,   injection:       same verbs
  explain, why,     claude hooks,
  pr, ask, brief,   opencode plugin,
  observe,          pi extension,
  activity          codex/grok hooks,
                    desktop notify)
```

Three modes of engagement:

1. **Session lifecycle** (proactive, near-zero tokens): session-start briefing, away/return detection, checkpoint nudges.
2. **Passive analysis** (proactive, budgeted tokens): periodic review of accumulated diff, observations queued by severity.
3. **On-demand tools** (tokens on invoke): commit, review, explain, why, pr, rebase, ask.

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
    observe.go brief.go activity.go
    config.go update.go
  internal/
    watcher/                # fsnotify tree watch, debounce, gitignore filtering
    gitstate/               # git CLI shell-outs: status, diff, log, blame
    store/                  # sqlite (modernc.org/sqlite): events, activity, observations, sessions, analyses
    narrative/              # derive compact session narrative from store
    rules/                  # mechanical observation rules (zero token)
    analyzer/               # triggers, budget, prompt assembly, LLM runner
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
- claude: run `claude mcp add --transport http --scope user peer http://127.0.0.1:7433/mcp`; merge hook entries into `~/.claude/settings.json` (SessionStart -> `peer hook claude session-start`, UserPromptSubmit -> `peer hook claude user-prompt-submit`, Stop -> `peer hook claude stop`); optional statusline snippet.
- opencode: add MCP entry to `~/.config/opencode/opencode.json`; install `peer-observe.js` plugin into `~/.config/opencode/plugins/`.
- pi: install `peer-observe.ts` extension; add MCP entry if pi supports MCP, otherwise extension registers thin wrapper tools that shell out to `peer` CLI.
- codex/grok: add MCP entry to their config; merge hooks fragments.
- Idempotent; `--dry-run` prints what would change; never overwrites foreign entries.

---

### `peer status`

One-screen daemon and session overview.

```
daemon      running (pid 8412, 2 projects, up 6h)
mcp         http://127.0.0.1:7433/mcp (3 clients)
project     nsr        branch fix-auth   dirty 4 files   last edit 2m ago
project     yes2infra  branch main       clean           idle 3h
analyzer    standard | today: 14.2k/100k tokens | last run 12m ago
queue       2 observations (1 warning, 1 suggestion)
```

---

### `peer brief [--project <p>]`

Print the session briefing (same content the harness gets at SessionStart).

Sections, assembled zero-token except the optional narration pass:
1. Where you left off: last session summary + last edited files.
2. Git: branch, dirty files, last commit (relative age), ahead/behind upstream.
3. External (requires gh, config `brief.remote_checks`): open PRs authored/assigned, CI status of last pushed branch.
4. Queued observations (drains them as delivered).

---

### `peer observe [--drain] [--min-severity <s>] [--json]`

List queued observations. `--drain` marks them delivered.

```
warning     auth.go:88 new branch on error path returns nil err (rule: ai-analysis, 12m ago)
suggestion  auth_test.go not updated alongside auth.go (rule: test-sibling, 25m ago)
```

---

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

Context-aware commit: stage, generate message, commit.

```
peer commit
peer commit --staged          # only what is already staged
peer commit -m "override"     # skip generation
peer commit --amend
peer commit --dry-run         # print proposed staging + message, do nothing
```

Behavior:
1. Determine scope: staged changes if any, else all tracked modified files (`--staged` forces the former).
2. Refuse empty scope.
3. Generate message via LLM runner: diff + session narrative + last 10 commit subjects for style matching. One-line subject, body only when the why is not obvious.
4. `git add` scope (if needed), `git commit -m` with generated message. User git config applies (signing, hooks). Never `--no-verify`, never `--no-gpg-sign`.
5. Warn (do not stage) on sensitive-pattern files (`.env*`, `*.pem`, `*.key`, `credentials*`); require explicit `--include <file>`.
6. Print the commit hash and message.

Errors:
- Nothing to commit.
- Not a git repo / detached state warnings.
- LLM runner failure (falls back: print diffstat, ask human for a message via error text, never commit with a junk message).

---

### `peer review [flags]`

Review uncommitted work (tokens on demand, not budget-gated).

```
peer review                # unstaged + staged vs HEAD
peer review --staged
peer review --focus "error handling"
```

Sends diff + session narrative to the LLM runner with a review prompt.
Output: findings with file:line, severity, one-line rationale.
Findings are also recorded as delivered observations (so the analyzer never re-flags them).

---

### `peer explain <target> [--level beginner|normal|deep]`

Teach what code does, with session context.

`target`: `path`, `path:line`, `path:start-end`, or a symbol name (resolved via `git grep -n`).
The prompt includes: the code slice, surrounding context, whether/when the human edited it this session, and recent related changes.

---

### `peer why <file:line>`

Explain why a line is the way it is: `git log -L`/`git blame` + commit messages + linked PR (via gh when available), narrated by the LLM runner.

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
3. Generate title + body from branch diff vs base + session narratives covering the branch's lifetime + commit list. Body format: Summary bullets, `Fixes #N` when a linked issue is inferable (never invented), Test plan.
4. `gh pr create`; print URL.

---

### `peer rebase`

Guidance, not execution.
Analyzes current branch vs upstream/base: what diverged, conflict predictions per commit, a suggested step-by-step plan (`git rebase -i` sequence, which commits to squash/fixup based on message and diff overlap).
Prints the plan; the human runs it.

---

### `peer ask "<question>"`

Freeform question answered with project + session context injected (narrative, git state, recent diff). One-shot LLM runner call.

---

### `peer hook <harness> <event>`

Internal shim: reads the harness hook payload on stdin, talks to the daemon over the control socket, emits the harness-specific response on stdout.
Sub-second, no LLM calls.

```
peer hook claude session-start     # -> {"hookSpecificOutput":{"additionalContext": "<brief>"}}
peer hook claude user-prompt-submit# -> queued observations as additionalContext (drains queue)
peer hook claude stop              # -> session accounting; never blocks stop
peer hook codex ...                # same pattern, codex payload shapes
```

---

### `peer config [get|set|edit]`, `peer update`, `peer version`

Standard. `peer update` follows the secondhand/no-mistakes/treehouse pattern (GitHub Releases, checksum verify, in-place replace, daily non-blocking version check notice).

## MCP server specification

Transport: streamable HTTP at `http://127.0.0.1:7433/mcp` (config `mcp.port`), built on the official `modelcontextprotocol/go-sdk`.
One daemon serves many concurrent harness sessions; sessions are distinguished per MCP session id.
`peer mcp-stdio` runs a stdio-to-daemon proxy for clients without HTTP support; the proxy forwards its own cwd so the daemon can route the session to the right project.
The server does not depend on MCP sampling, resource subscriptions, or server-push notifications for anything load-bearing: no target harness implements the client side, and the 2026-07-28 spec revision deprecates sampling/roots/logging outright.
When the claude Channels adapter is enabled, the server additionally declares the experimental `claude/channel` capability (an Anthropic extension, not core MCP).

Project routing: every tool accepts an optional `directory` argument.
Resolution order: explicit argument, then MCP roots (when the client provides them), then stdio-proxy cwd, then the sole registered project when only one exists.
Ambiguity returns an error naming the candidates.

Tools (names use underscores; clients namespace them, e.g. `mcp__peer__commit`):

| Tool | Args | Returns | Annotations |
|---|---|---|---|
| `commit` | `directory?`, `staged_only?`, `message_override?`, `amend?`, `dry_run?` | commit hash + message, or proposal when dry_run | destructive |
| `review` | `directory?`, `staged_only?`, `focus?` | findings list | read-only |
| `explain` | `target`, `directory?`, `level?` | explanation text | read-only |
| `why` | `location`, `directory?` | history narration | read-only |
| `pr_create` | `directory?`, `base?`, `draft?`, `no_push?` | PR URL | destructive |
| `rebase_plan` | `directory?`, `base?` | step-by-step plan | read-only |
| `ask` | `question`, `directory?` | answer | read-only |
| `observations` | `directory?`, `min_severity?`, `drain?` | queued observations | read-only |
| `briefing` | `directory?` | session briefing | read-only |
| `activity` | `directory?`, `since?` | session narrative | read-only |

Design rules:
- Tool descriptions teach the harness when to call them ("When the user asks to commit, call `commit` instead of running git yourself; peer has watched the session and writes better messages.").
- Tools that spend tokens say so in their description.
- `commit` and `pr_create` carry destructive annotations so harnesses can gate them behind approval.
- Structured content: tools return both human text and structured JSON (outputSchema) so harnesses can render or post-process.

Resources (optional, for clients that support them):
- `peer://briefing` and `peer://activity` mirror the corresponding tools for @-mention style inclusion.

Notifications: the server emits `notifications/tools/list_changed` only on upgrade.
Proactive delivery does NOT rely on MCP server-push (client support for subscriptions/sampling is not dependable); it uses the adapters below.

## Adapter specification (proactive delivery)

The daemon exposes an SSE feed: `GET http://127.0.0.1:7433/events?project=<root>`.
Events: `observation` (with severity), `briefing_ready`, `away_return`.
Adapters subscribe and inject according to harness capability.

| Harness | Mechanism | Class |
|---|---|---|
| pi | extension subscribes to SSE, injects via `pi.sendMessage(..., {deliverAs: "steer", triggerTurn: true})` | push |
| opencode | plugin subscribes to SSE, injects via `client.session.promptAsync` | push |
| claude-code (stable) | hooks pull: SessionStart injects briefing, UserPromptSubmit drains queue into additionalContext; statusline shows queue depth ambiently | pull-at-turn |
| claude-code (preview) | Channels: MCP server declares experimental `claude/channel` capability; off by default until the feature exits research preview | push |
| codex | hooks.json pull (UserPromptSubmit stdout context); `notify` key for desktop-notification degrade | pull-at-turn |
| grok | hooks pull (event taxonomy mirrors claude; additionalContext support unconfirmed, verify before enabling) | pull-at-turn |
| any-in-herdr | optional: `herdr pane send-text` when pane agent state is idle; `herdr notification` | push (terminal) |
| none | desktop notification (`notify-send` / `osascript`), critical severity only | fallback |

Delivery policy (daemon-side, adapters stay dumb):
- critical: push immediately on every available channel.
- warning: push only when the session is idle (harness idle event or >=60s no activity); otherwise queue.
- suggestion/info: queue only; drained at next turn, next brief, or explicit `observe`.
- Staleness: before delivery, re-verify the observation's file fingerprint; drop if the code changed.
- Dedup: fingerprint(rule, file, normalized finding); once delivered or dismissed, never repeat.

## Watcher specification

- fsnotify on each project tree; recursion is hand-rolled (fsnotify has no recursive mode: walk the tree and `Add` each directory at start, `Add` new directories on Create events); honors .gitignore (plus config `watch.ignore` globs and built-in excludes: `.git` internals except the state files below, `node_modules`, build dirs).
- Debounce 400ms per path; editor atomic-save patterns (write-temp-then-rename, vim backup dance) normalized to writes by watching the directory, not the file, and re-adding watches after renames.
- macOS note: fsnotify uses kqueue (one fd per watched item), so the gitignore filter matters even more there; inotify watch exhaustion on Linux degrades that project to polling with a warning observation.
- Git state files watched directly for instant signals: `.git/HEAD`, `.git/index`, `refs/heads/`, `MERGE_HEAD`, `REBASE_HEAD`, `FETCH_HEAD`.
- Reconciliation: `git status --porcelain=v2` at most every 30s and after git events; churn via `git diff --numstat` on quiescence, never per event.
- Away detection: no events for `session.away_threshold` (default 15m) => away; next event => return.
- Watch limits: on inotify exhaustion, log, fall back to polling that project, surface a warning observation.

## Analyzer specification

Triggers (all subject to `min_interval`, default 10m, and the daily budget):
- quiescence: an edit burst (>=3 saves) followed by >=2m of silence.
- volume: >=`analysis.file_threshold` files or >=`analysis.line_threshold` changed lines since last analysis.
- staged: the index grew (imminent commit; jump the queue).
- Never analyzes mid-burst, never re-analyzes an unchanged diff (input hash).

Input: unified diff since last analyzed state (capped, large diffs truncated file-by-file with a note) + session narrative + list of previously surfaced findings (to forbid repeats).
Output contract: JSON array of `{severity, title, detail, file, line?, confidence}`; findings below `analysis.min_confidence` (default 0.6) are dropped; at most `analysis.max_observations` (default 5) queued per run.

What it looks for (prompt profile, config-selectable): potential bugs, missed edge cases, test gaps, style drift vs the surrounding file, dead/leftover debug code.

Budget: `analysis.daily_tokens` (default 100k), spend recorded per run in the store; intensity presets set trigger thresholds + model:
- off: no analyzer, mechanical rules only.
- light: staged trigger only, small model.
- standard (default): all triggers, small model.
- eager: all triggers, tighter intervals, mid model.

## LLM runner

All AI calls go through one command-template runner (config `llm.command`), default:

```
claude -p --output-format json --model {model} {prompt}
```

Template variables: `{model}`, `{prompt}` (or stdin).
Alternatives documented: `opencode run --format json`, `codex exec`, or `api:anthropic` mode using an API key.
Timeouts, JSON extraction, and retry-once semantics live in the runner.
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

## Session continuity

- A session = contiguous activity window per project (bounded by away gaps or harness SessionStart/Stop signals when available).
- At session end (away > threshold or hook signal): write a compact summary row (files touched, churn, commits made, observations delivered, one-line LLM narration when budget allows).
- Persisted: sessions, summaries, undelivered + delivered observations, analysis findings history, budget counters.
- Re-derived live, never persisted: git status, branch, diff, PR/CI state.
- Retention: raw events pruned after `store.retain_days` (default 14); sessions/observations kept 90 days; store vacuumed weekly.

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
daily_tokens = 100000
min_interval = "10m"
min_confidence = 0.6
max_observations = 5

[session]
away_threshold = "15m"

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

Per-project `.peer.toml` may override `[analysis]`, `[rules]`, `[watch]`, `[brief]`.

## State management

- Single sqlite database (modernc.org/sqlite, pure Go, WAL mode). Tables: events, file_activity, observations, sessions, analyses, projects, kv.
- The daemon is the only writer. CLI and hook shims talk to the daemon over the control socket; they never open the db for writes (read-only fallback when the daemon is down: status/activity still work).
- Atomic config edits (temp + rename). No lock files beyond the daemon pidfile flock.

## Error handling

- Fail closed on mutations: commit refuses empty scope and junk messages; pr refuses unpushed surprises; nothing destructive without an explicit human-initiated call.
- Fail open on observation: watcher errors degrade to polling; analyzer errors skip the run and log; a broken adapter never block the daemon.
- Hook shims always exit 0 with valid harness JSON (a broken peer must never break the human's harness); errors go to the daemon log.
- Exit codes: 0 success, 1 error, 2 usage, 3 precondition failed.
- Errors to stderr, structured output to stdout.

## Testing strategy

- Unit: debounce/coalesce, gitignore filtering, rule engine table tests, narrative derivation, budget accounting, prompt assembly (golden files), message generation JSON parsing, config validation, observation dedup/staleness.
- Integration: temp git repos with scripted edit sequences -> expected events/aggregates/rules; daemon lifecycle (start, signal handling, single-instance); MCP server exercised with an MCP client library (list/call each tool); hook shims with recorded harness payloads.
- LLM calls mocked via the command template (point `llm.command` at a fixture script).
- e2e (tagged): real `claude -p` smoke test for commit message generation, real `gh` mocked.

## What is NOT in scope

| Cut | Why |
|---|---|
| Writing or editing code | Definitional. Peer assists; the human codes. |
| Autonomous task execution | That is secondhand's job. |
| Editor plugins (VS Code, JetBrains) | Terminal-first; editors reach peer through their harness's MCP. Maybe later. |
| PR review bot / CI integration | Peer reviews uncommitted local work; CodeRabbit et al own the PR lane. |
| Keystroke-level completion | Save-granularity only. Copilot owns keystrokes. Future: LSP server for editor-agnostic completions (post-v1, see below). |
| Embeddings / vector search of the codebase | The narrative is small and structured; grep and git cover retrieval. Revisit only with proven need. |
| Multi-user / remote server | One daemon per user per machine, localhost only. |
| Windows | Linux + macOS first. |
| Telemetry | None, ever. |

## Dependencies

| Dependency | Purpose | Why this one |
|---|---|---|
| `spf13/cobra` | CLI subcommands | de facto standard for multi-command Go CLIs; matches secondhand |
| `fsnotify/fsnotify` | filesystem events | the primitive every mature Go watcher uses; recursion hand-rolled (upstream has none) |
| `modernc.org/sqlite` | context store | pure Go (no cgo), trivial cross-compilation; perf gap vs mattn irrelevant at this write rate |
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

### Phase 1: watch and remember (zero tokens end-to-end)
1. `peer daemon` foreground: watcher, git state, store, away detection.
2. `peer project add/list/remove`, `peer status`, `peer activity`.
3. Mechanical rules + observation queue + `peer observe`.
Deliverable: run the daemon for a day, `peer activity` tells the true story of the session.

### Phase 2: on-demand tools
4. LLM runner + `peer commit` + `peer review` + `peer ask`.
5. MCP server (streamable HTTP) exposing the phase-2 verbs + `activity`/`observations`/`briefing`.
6. `peer connect claude` (MCP registration + hooks) and `peer hook claude *` shims.
Deliverable: from inside claude-code, "commit this" produces a context-aware commit through peer.

### Phase 3: proactive loop
7. Analyzer: triggers, budget, prompt, observation output.
8. Briefing assembly + away/return summaries (+ gh remote checks).
9. Delivery policy + SSE feed + opencode plugin + pi extension + desktop fallback.
Deliverable: peer taps your shoulder, usefully, at most a few times a day.

### Phase 4: polish
10. `peer explain`, `peer why`, `peer pr`, `peer rebase`.
11. codex/grok adapters, stdio proxy, statusline.
12. Docs, e2e, release pipeline.

## Open decisions

1. **Analyzer LLM default**: shell-out (`claude -p`) vs API mode. Recommendation: shell-out default (zero config, uses existing auth), with `api:anthropic` as opt-in for users who want lower latency and have an API key.
2. **Proactive delivery aggressiveness**: how eagerly peer pushes observations. Recommendation: middle policy - push warnings only when idle (>=60s no activity), queue suggestions/info for drain at next turn, push critical immediately. Avoids both "invisible tool" and "noisy interruption".

## Future considerations (post-v1)

### LSP auto-complete server

Peer's watcher and context store position it to provide editor-agnostic code completions via LSP (Language Server Protocol).
Every modern editor speaks LSP natively, so no per-editor plugin is needed.

Architecture fit:
- The watcher already has file context, edit history, and git state.
- Mechanical rules could power instant pattern-based completions (<50ms).
- Cached analytical observations from background analysis could fuel richer suggestions.
- `textDocument/completion` is the standard LSP hook; users configure their editor to point at peer's LSP server.

This would be a fourth interface alongside CLI, MCP, and harness adapters.
Adds an LSP server to the daemon (same process, separate port or stdio), reusing the store and narrative.
Not in v1 because the MCP+CLI surface already covers the high-value tools; completions are a separable concern that benefits from a stable watcher and analyzer first.
