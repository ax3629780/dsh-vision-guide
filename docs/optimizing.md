# Optimizing cost & latency

`dsh-auxiliary` ships several auxiliary routes beyond vision. All are **off by default** — enable the ones you
use to move non-core work off the main (expensive) model.

## Route compaction + title generation to a cheaper model

Two routes pay off quickly because they fire on every session or every long session:

```yaml
dsh-auxiliary:
  vision:
    provider: new-api        # your existing vision route
    model: gpt-5.6-sol
  title:                     # session-title generation (fires once per session)
    enabled: true
    provider: <cheap-provider>
    model: <cheap-model>
  compact:                   # context-overflow compaction (fires on long sessions)
    enabled: true
    provider: <cheap-provider>
    model: <cheap-model>
```

- **`title`** — the session title is generated once per session; a cheap model here is pure savings.
- **`compact`** — long sessions (hundreds of thousands of input tokens) compact through this model instead of the
  main one, which is where most of the savings sit.

## Optional: subagents, approvals, image generation

The same pattern applies to the other routes:

```yaml
dsh-auxiliary:
  subagent:                 # every delegated child agent
    enabled: true
    provider: <cheap-provider>
    model: <cheap-model>
  approve:                  # requires @dsh-plugin/dsh-approve-for-me
    enabled: true
    provider: <provider>
    model: <model>
  imagegen:                 # auxiliary image-generation work
    enabled: true
    provider: <provider>
    model: <model>
```

## Tune the vision call

```yaml
dsh-auxiliary:
  vision:
    maxTokens: 4096         # default 4096
  tool:
    timeoutMs: 180000       # default 120000 — raise only if the vision model is slow
    maxImageBytes: 10485760 # default 10 MiB
```

See [troubleshooting.md](./troubleshooting.md) for the vision-timeout and `inspect_image` extension gotchas.

> **Placeholders:** every `provider` / `model` value above is a placeholder — point them at models already
> registered in your `llm-pi-ai` (or equivalent) provider namespace. No base URLs or API keys belong here.
