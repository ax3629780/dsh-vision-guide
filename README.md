# dsh-vision-guide

Add image understanding to **text-only** [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) models — e.g. `deepseek-v4-pro` — without hacking core packages.

> 中文说明见 [README.zh.md](./README.zh.md)

> **Verified against** `@deepseek-ai/dsh@0.1.0-rc.6`. The extension seams used here (`ctx.llm.resolveModelInfo`, `llm/stream`, `inputModalities`) are pre-1.0 and may move in future releases.

## The problem

DeepSeek V4 Pro / Flash and many strong coding models are **text-only**. Dropping an image into the chat fails with:

> Model "...": does not accept image input

## The solution

Use the community plugin [`@dsh-plugin/dsh-auxiliary`](https://github.com/dsh-plugins/dsh-auxiliary). It layers vision (plus optional compaction / subagent / title / approval / image-generation routes) on the **official `ctx.llm` seam**:

1. A `resolveModelInfo` wrapper makes a text-only model *claim* image input, so the stock admission check passes — **no core-package edits**.
2. An `llm/stream` listener rewrites image blocks into lightweight `[image: {...}]` text references.
3. The main model calls the `describe_image` tool **on demand** to fetch the image content from a vision-capable model (`gpt-5.6-sol` in the example).

## Quick start

```sh
# 1. install (needs pnpm on PATH)
dsh plugin --profile web add @dsh-plugin/dsh-auxiliary

# 2. configure vision route in ~/.dsh/settings.yaml
```

```yaml
dsh-auxiliary:
  vision:
    provider: new-api          # any already-registered provider
    model: gpt-5.6-sol         # a vision-capable model on that provider
```

```sh
# 3. restart
dsh web
```

Then drop an image into the chat. The UI keeps the original image; the main model sees it via `describe_image`.

## Why this approach (advantages)

| Advantage | Detail |
|---|---|
| **Reuses your providers** | `vision.provider` / `vision.model` point at models already registered in `llm-pi-ai` — no base-URL guessing |
| **No core-package hack** | Uses official seams (`ctx.llm`, `resolveModelInfo` wrapper, `llm/stream` waterfall) — survives reinstall |
| **On-demand (efficient)** | Images become tiny text references; the vision model runs only when the main model actually calls `describe_image` |
| **One plugin, many routes** | Vision + compaction + subagent + title + approval + image generation |
| **CI + TypeScript** | `typecheck` + `build` on Linux/macOS/Windows before every publish |

## Referenced projects

This guide was written after surveying several community vision plugins. See [docs/alternatives.md](./docs/alternatives.md) for the full comparison and why `dsh-auxiliary` was selected.

## Troubleshooting

Known pitfalls (the crash and the "can't switch model" bug) are documented in [docs/troubleshooting.md](./docs/troubleshooting.md).

## Optimizing

Beyond vision, `dsh-auxiliary` can route compaction, title, subagent, approval and image-generation work to cheaper models. See [docs/optimizing.md](./docs/optimizing.md).

## License

[MIT](./LICENSE)
