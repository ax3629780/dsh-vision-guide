# Referenced projects & why `dsh-auxiliary`

This guide surveyed the DeepSeek Harness vision-plugin ecosystem before settling on
[`@dsh-plugin/dsh-auxiliary`](https://github.com/dsh-plugins/dsh-auxiliary). The projects below were
evaluated; the official-harness Discussions [#456](https://github.com/deepseek-ai/deepseek-harness/discussions/456),
[#495](https://github.com/deepseek-ai/deepseek-harness/discussions/495) and
[#733](https://github.com/deepseek-ai/deepseek-harness/discussions/733) confirm this is an official "not built-in,
community-provided" area.

## Comparison

| Project | Mechanism | Tests | Cache / persistence | Core hack? | Reuses your provider? |
|---|---|---|---|---|---|
| **[`@dsh-plugin/dsh-auxiliary`](https://github.com/dsh-plugins/dsh-auxiliary)** ⭐ | `resolveModelInfo` wrapper + `llm/stream` + `describe_image` tool | none (CI typecheck) | none (on-demand) | **no** | **yes** (`vision.provider`/`model`) |
| [`dsh-vision-fallback`](https://github.com/1HelloMan1/dsh-vision-fallback) | `agent/pre-step` + model-only surface replacement | ~22 | `observations.json` (survives restart/compaction) | no | no (raw `baseURL`) |
| [`dsh-vision-router`](https://github.com/ysr666/dsh-vision-router) | tool-based (~14 tools) + stealth mode | ~657 | — | no (optional) | yes |
| [`dsh-llm-image-routing`](https://github.com/CuzWeAre/dsh-llm-image-routing) | image routing | — | — | — | — |
| [`dsh-auto-vision`](https://github.com/NormanFxxkingRockwell/dsh-auto-vision) | auto-discovers a multimodal model | — | — | — | yes |
| [`dsh-image-pathify`](https://github.com/dami9527/dsh-image-pathify) | built-in `识图` tool | — | — | — | — |
| [`visual-review`](https://github.com/wang-bool/visual-review) | image upload + recognition | — | — | — | — |
| [`dsh-llm-auto-route`](https://www.npmjs.com/package/dsh-llm-auto-route) | automatic model routing | — | — | — | — |

> Test counts, cache filenames and URL conventions for third-party projects were spot-checked against the
> linked repos at the time of writing and may drift as those projects change. Re-verify before citing them.

## Why `dsh-auxiliary` won

1. **Reuses an already-registered provider.** `vision.provider: new-api` + `vision.model: gpt-5.6-sol` point
   straight at a model already configured under `llm-pi-ai` (including its API key via `apiKeyEnv` and its exact
   endpoint). `dsh-vision-fallback` instead needs a raw `baseURL` and appends a hardcoded `/chat/completions`,
   which may not match a provider configured for `openai-responses`.

2. **No core-package hack.** It uses the official `ctx.llm` seam — a `resolveModelInfo` wrapper makes text-only
   models claim image input, so the *stock* admission check passes. This survives `npm i -g @deepseek-ai/dsh`
   reinstall and avoids the "forgot to patch `selectModel`" class of bug (see troubleshooting).

3. **On-demand, not per-turn.** Images become tiny `[image: {...}]` references; the vision model runs only when the
   main model actually calls `describe_image`. This is cheaper than a silent bridge that re-describes every turn.

4. **One plugin, many auxiliary routes.** Vision is the headline, but the same plugin also routes compaction, subagent,
   title, approval and image-generation work to dedicated models — one install, several cost savings.

5. **Engineering hygiene.** TypeScript with `typecheck` + `build` enforced on Linux/macOS/Windows in CI before every
   publish, signed commits, active maintenance.

## Known trade-offs

- **No unit tests** (unlike `dsh-vision-fallback`'s ~22 and `dsh-vision-router`'s ~657).
- **No result cache** — `describe_image` re-calls the vision model each time; mitigated by it being on-demand.
- **Compaction doesn't preserve image descriptions** — with a text-only compaction model, images degrade to text
  references in the summary (`dsh-vision-fallback` handles this explicitly).
- **LGPL-3.0-only** license — fine for personal use; mind the copyleft terms if you redistribute.
- **Single maintainer** — bus factor 1.

If your top priority is *persistent caching across restart/compaction* rather than *reusing your provider config*,
[`dsh-vision-fallback`](https://github.com/1HelloMan1/dsh-vision-fallback) is the stronger single-purpose pick.
