# Troubleshooting

## 1. `prepared LLM call config changed before adapter dispatch`

**Symptom:** sending an image fails the whole turn with this `INVALID_PREPARED_CALL` error.

**Root cause (in a hand-rolled bridge):** the plugin called `prepareCall()` with one config, then dispatched
`prepared.stream(request)` with a *different* config. `callConfigEquals` compares
`provider / model / reasoningEffort / temperature / maxTokens / stop` field-by-field, so a
`temperature: 0` added only to the request (and not to `prepareCall`) made `undefined !== 0` → the frozen prepared
config no longer matched the dispatched request.

**Fix:** make the prepared config and the dispatched request identical — pass every override (`temperature`,
`maxTokens`, …) to `prepareCall` itself, and build the request from `prepared.config`.

**Why `dsh-auxiliary` avoids it:** its vision tool calls `ctx.llm.stream(...)` directly (no `prepareCall`/`stream`
hand-shake), so there is no frozen config to mismatch.

## 2. "Can't switch model" after sending an image

**Symptom:** once a session contains an image, switching to a text-only model is rejected
(`model-unavailable` / "does not accept image input").

**Root cause (in the hand-rolled approach):** the *stock* `dsh-host-apiproxy` already blocks this switch, and a
hand-rolled "escape hatch" was added only to the `prompt` path — the parallel check in `selectModel` was left out.

**Fix:** don't patch the core apiproxy at all. `dsh-auxiliary` solves it at the right layer — its
`resolveModelInfo` wrapper makes the text-only model *report* image input, so **both** the `prompt` and
`selectModel` admission checks pass naturally, and you can switch models freely.

## 3. Don't patch `dsh-host-apiproxy`

Modifying the globally-installed `@deepseek-ai/dsh-host-apiproxy/lib/index.js` is tempting but:

- it is **not durable** — a reinstall/upgrade of `@deepseek-ai/dsh` silently reverts it;
- it is **easy to leave half-patched** (see issue #2 above);
- it is unnecessary — the official `ctx.llm` seam (what `dsh-auxiliary` uses) is the supported extension point.

## 4. Install gotchas

- **`pnpm` is required** for `dsh plugin add` (`npm install -g pnpm`). `dsh plugin` forwards to pnpm via
  `spawnSync(..., { stdio: "inherit" })`.
- **Pin the plugin version.** `dsh-auxiliary` iterates fast and has no test suite — prefer an exact
  `"@dsh-plugin/dsh-auxiliary": "0.5.1"` over `^0.5.1`.
- **Peer deps resolve from `~/.dsh/profiles/node_modules`.** The `@deepseek-ai/*` peer dependencies are not
  installed into the profile's own `node_modules` (`autoInstallPeers: false`); they resolve from DSH's shared
  fallback directory at runtime. The pnpm "peer dependencies" warning is expected and harmless.
- **`allowBuilds` spelling** in `pnpm-workspace.yaml` (pnpm expects `allowBuilds`, not `allowedBuilds`).

## 5. Vision route times out ("Request timed out")

**Symptom:** `describe_image` / `inspect_image` fails with `vision model call failed (TIMEOUT): Request timed out`.

**Root cause:** a transport-level timeout from the provider/relay, not the plugin's own deadline. The plugin's
`tool.timeoutMs` deadline surfaces with code `AUX_VISION_TOOL_TIMEOUT`; a plain `TIMEOUT` means the vision
provider (often a shared `new-api`-style relay) is slow, rate-limited or down.

**Fix:** retry (transient rate-limits pass); check the relay endpoint is healthy; switch to a more reliable
vision provider; or raise `tool.timeoutMs` **only** if the model itself is slow — a longer deadline does not
fix a hard transport timeout.

## 6. `inspect_image` rejects files without an image extension

**Symptom:** `inspect_image` fails with `unsupported image extension "..." (supported: png/jpg/jpeg/webp/gif)`.

**Root cause:** `inspect_image` infers the media type from the file path's extension, so attachment object files
(hash-named, no `.png`/`.jpg` suffix) are rejected.

**Fix:** give the file a recognized extension first (e.g. copy to `foo.png`), or use `describe_image` with the
`[image: {...}]` reference for images already attached to the chat.
