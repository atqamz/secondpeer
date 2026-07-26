# Decisions

This document is the durable home for open design decisions on this project.
It lives alongside `SPECS.md` so forks in direction are recorded in the repo instead of an external task queue.
A decision is resolved by recording the outcome here, in place, once it is made.
Until then, every entry below is open and undecided.

## analyzer-llm-default

Status: open, undecided.

This is a cost-model fork for how the analyzer reaches a model by default.

- Option A: shell out to the locally installed `claude` binary, riding the user's existing Claude subscription.
This means no key management at all, and a harness-agnostic command template keeps it from hard-coding one vendor's CLI.
- Option B: require `ANTHROPIC_API_KEY` and call the API directly through `anthropic-sdk-go`.
This means per-token billing for the user instead of riding an existing subscription.

Standing recommendation: use Option A as the default, with Option B available as an explicit configuration option.
No decision has been made yet.

## delivery-default

Status: open, undecided.

This is a product-personality fork for how loudly the daemon surfaces observations by default.
It is one configuration value either way, but the default shapes a new user's first contact with the tool.

- Quieter: queue everything except critical.
- Middle: critical pushes immediately, warnings push only when the session is idle, and suggestions and info queue silently.
- Louder: warnings always push.

Standing recommendation: the middle policy.
No decision has been made yet.
