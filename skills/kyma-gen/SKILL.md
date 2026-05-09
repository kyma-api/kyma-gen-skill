---
name: kyma-gen
description: >
  Run a Kyma API model to generate images, video, or audio with the kyma-gen
  CLI. Use this when the user asks to create an image, generate a video, make
  audio, restyle media, run a model, generate a logo, design a thumbnail,
  produce a clip, or "use kyma-gen". Vietnamese intents also covered:
  "tạo ảnh", "vẽ", "sinh ảnh", "làm clip", "tạo video", "lồng tiếng",
  "thiết kế", "render". Guides discovery, schema inspection, input prep,
  execution, and result handling.
version: 0.1.0
tags:
  - generative-ai
  - image
  - video
  - audio
  - cli
  - kyma
---

# kyma-gen workflow

Use this skill when the user wants to execute a Kyma API model to generate
media. The CLI is `kyma-gen` (or the short alias `kg` if installed). It is
agent-first: every command supports `--json` for structured output and emits
canonical error envelopes you can parse without scraping.

## Steps

1. **Discover models** — search the catalog by free-text query and filter by
   modality:

   ```sh
   kyma-gen models "logo design" --json
   kyma-gen models --category text-to-image --limit 5 --json
   ```

   Each result has `endpoint_id` (use this to invoke), `name`, `category`
   (`text-to-image`, `text-to-video`, etc.), and `description`.

2. **Inspect the input shape** — before constructing a request, check the
   model's schema. Two formats:

   ```sh
   kyma-gen schema recraft-v4 --json              # compact: flag list
   kyma-gen schema recraft-v4 --format openapi    # raw OpenAPI 3.0
   ```

   The compact form returns `{ endpoint_id, name, category, input: [...flags], output: [...] }`.
   Each flag has `flag`, `type`, `required`, optional `enum`, optional `default`.

3. **Check pricing** — for models with a known price (most generative SKUs):

   ```sh
   kyma-gen pricing recraft-v4 --json
   ```

4. **Run the model** — pass each schema flag as `--<key> <value>`. Coercion
   happens client-side: `true`/`false`/`null`, integers, decimals, and JSON
   `[...]`/`{...}` are detected automatically; everything else stays a string.

   ```sh
   kyma-gen run recraft-v4 --prompt "kyma wave logo" --aspect_ratio 1:1 --json
   ```

   To download every URL in the result automatically, add `--download` (with
   an optional template):

   ```sh
   kyma-gen run recraft-v4 --prompt "logo" --download "./out/{index}.{ext}" --json
   ```

5. **Async submit** — for long-running jobs (video, audio), pass `--async`
   to receive a `request_id` immediately, then poll:

   ```sh
   kyma-gen run kling-v1.6 --prompt "..." --async --json
   # → {"status":"submitted","request_id":"kg_01XYZ",...}
   kyma-gen status kling-v1.6 kg_01XYZ --result --json
   kyma-gen status kling-v1.6 kg_01XYZ --cancel --json    # to abort
   ```

6. **Upload reference media** — when a model needs an image/video/audio
   input (image-to-image, video-to-video), upload the local file first or
   re-upload an existing URL so the resulting `cdn_url` is owned by the
   user:

   ```sh
   kyma-gen upload ./reference.png --json     # multipart
   kyma-gen upload https://x.com/y.png --json # server re-fetches + re-stores
   # → use the returned cdn_url as input.image_url on the next run
   ```

7. **Discover dynamically per-model** — `kyma-gen run <id> --help`
   introspects the model's schema and renders the per-model flag list:

   ```sh
   kyma-gen run recraft-v4 --help
   # → shows --prompt (required), --aspect_ratio (enum), --negative_prompt, --seed
   ```

## Handling errors

Every command emits a canonical error envelope on stderr (JSON mode) per the
error-shape contract:

```json
{
  "error": "<short message>",
  "details": {
    "error_type": "<one-of-15>",
    "cli_version": "0.1.0",
    "status": <http-status>,
    "endpoint_id": "<if-applicable>",
    "request_id": "<if-server-issued>",
    "validation_errors": [...],
    "topup_url": "<for 402>",
    "hint": "<recovery hint>"
  }
}
```

Recovery rules by `error_type`:

- `AuthError` (401) — set `KYMA_API_KEY` or run `kyma-gen setup
  --non-interactive --api-key <ky-...>`. Stop and ask the user for a key.
- `InsufficientBalanceError` (402) — `details.topup_url` points at the
  billing page. Stop and tell the user to top up.
- `ValidationError` (422) — `details.validation_errors[]` lists each
  problem field. Re-read the schema (`kyma-gen schema <id>`), fix the
  inputs, re-run.
- `NotFoundError` (404) — `endpoint_id` is wrong. Re-search via
  `kyma-gen models`.
- `RateLimitError` (429) — `details.retry_after_seconds` is the cooldown.
  Wait then retry once.
- `UpstreamError` (5xx) — Kyma already has 4-layer fallback server-side;
  if it returns 5xx all providers failed. Wait + retry once max.
- `NetworkError` / `TimeoutError` — retry once.
- `ChecksumError` / `SecurityError` — skill-bundle integrity / path-
  traversal. Stop; do not retry.

Errors go to **stderr**, success payloads go to **stdout**. So
`kyma-gen run … --json | jq …` works cleanly.

## Notes

- Always pass `--json` in agent contexts. The pretty/auto modes are
  TTY-only and will mix progress chatter with the payload.
- Use `kyma-gen schema <id> --format openapi` to embed full input/output
  contracts in tool definitions for downstream agents.
- The CLI honors `KYMA_API_URL` env override for staging deployments.
- Setup state lives at `~/.kyma-gen/config.json` (mode 0600, apiKey
  encrypted with AES-256-GCM keyed on hostname+username).
- Skills install into `.kyma-gen/.installed.json` manifest. Three
  adapters write per-tool: `.agents/skills/<name>/` (or
  `.claude/skills/<name>/` fallback), `.cursor/rules/<name>.mdc`, and
  a fenced block in `AGENTS.md`.
