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
