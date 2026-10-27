# natlang LoRA patch (on wllama master 3bb2a46, llama.cpp 5aeac37)

Two small changes let a page serve tuned LoRA stacks with per-request scales. Both are intended for upstreaming.

## wllama (TypeScript)

- `LoadModelParams.lora_adapters` entries take either `path` (inside the wasm FS) or `blob` (a GGUF Blob). Blobs are
  mounted as their own files (`models/lora-NNNNN.gguf`), never passed through `prepareBlobs`, which treats every blob
  as a model shard.
- `Wllama.capabilities.loraBlobs` for feature detection.

Usage: load with `lora_adapters: [{ blob, scale: 0 }, ...]` and `lora_init_without_apply: true`, then select a stack per
request with the server's existing field: `createCompletion({ ..., lora: [{ id, scale }] })` (also for chat).

## llama.cpp server (`tools/server/server-context.cpp`, submodule)

- A request without a `lora` field now goes through the same comparison as one with it, so returning to the default
  adapters after a request with other scales clears the slot's cached prompt. Before, the slot switched adapters but
  kept a prefix computed with the previous ones.
- The host prompt cache (`--cache-ram`, idle-slot caching) stores states without the adapters they were computed with
  and restores them by token prefix only. While LoRA adapters are loaded it is not consulted, and idle-slot caching is
  disabled. Recording adapters in `server_prompt` would let it be re-enabled.

Verified: natlang `ts-host/scripts/wllama-lora-smoke.mjs` (headless Chromium, LFM2.5-350M f16 + two LoRAs): each adapter
changes greedy output, returning to the base (explicitly or with no `lora` field) reproduces it exactly, and every
stack is reproducible after other stacks.

Base history: first written on wllama 3.6.1 (llama.cpp 83d855c5a; that submodule commit is kept as tag
`wllama-natlang-lora-b10664` in https://github.com/werg/llama.cpp-neuralese), then merged with upstream master 3bb2a46.
The llama.cpp change is cherry-picked onto upstream's pin 5aeac37 (`patches/llama.cpp-server-lora-cache.patch`); the
submodule points at https://github.com/werg/llama.cpp-neuralese, where that commit is tag `wllama-natlang-lora-5aeac37`.
The smoke above passes unchanged on the merged build (identical greedy outputs to the 3.6.1-based build).

# natlang native state spans and decision readout (llama.cpp-neuralese `wllama-natlang` 9ff259a)

The llama.cpp submodule moves to the fork's `wllama-natlang` branch: ee85d96 plus two natlang features, both also
served by the fork's native `llama-server`.

## Native state spans

- llama.cpp: `llama_state_span_register` imports exact per-layer state (attention keys before RoPE and values, LFM2
  short-convolution gated inputs) for virtual tokens; the server loads spans with `--native-state` and consumes one
  where a chat message has a `{"type": "native_state", "id"}` content part (cherry-pick of the fork's 6eff883, without
  `--lora-per-sequence`). Prompts submitted as CLI tasks (wllama's path) are tokenized the same way.
- wllama: `LoadModelParams.native_states` takes `{ path }` or `{ blob }` entries (blobs are mounted as their own files,
  like LoRA blobs); `Wllama.capabilities.nativeStates`.

## Decision readout

- llama.cpp server: `POST /v1/natlang/score` scores candidate replies (each as the closed final assistant message,
  summed over its tokens after the rendered prompt; the replies of an item share the prompt prefix). Task building and
  the response are shared functions (`server_score_tasks`, `server_score_response`). An explicit
  `"continue_final_message": false` now renders a final assistant message closed.
- wllama: `Wllama.createNatlangScore({ items })` (glue action `score`), `Wllama.capabilities.natlangScore`.

## Fixes found on the way

- `server_queue::defer` was a stub (`assert(false)`, compiled out), so a task that found no free slot was dropped and
  its reader waited forever (for example more scored replies, or systemone groups, than `n_parallel`). Deferred tasks
  now wait in `queue_tasks_deferred` and return to the queue when a slot is released, as in the native server.
- A failing action returned no message and the caller saw `Invalid magic number`; it now returns `glue_msg_error`,
  which the worker raises as `WllamaRuntimeError` with the original message.
- Chat content parts other than text were all treated as media; `native_state` parts now pass through.

Verified in headless Chromium against the fork's native `llama-server` (natlang `ts-host/scripts/wllama-fork-smoke.mjs`,
LFM2.5-350M F32, CPU): answers consuming native states match the native server token for token and within 0.05 in
log-probability, a state not loaded is refused with the server's message, and decision readout matches. The LoRA smoke
(`wllama-lora-smoke.mjs`, two random rank-8 LoRAs) passes 7/7.

Build: `scripts/docker-compose.yml` steps run with a local emsdk 4.0.20 (x86_64).

# natlang guided generation (llama.cpp-neuralese `wllama-natlang` b35efc8)

- llama.cpp server: a `"guidance"` request field (`true` or `{require_call, tools, repeat, run, syntax, retries}`)
  checks the reply while it is generated, as natlang's reference server: call names must be offered tools, and the
  code of `eval(code=...)` calls is checked line by line for repetition, redeclaration and TypeScript syntax
  (tree-sitter with the TypeScript grammar, vendored in `llama.cpp/vendor/tree-sitter`). A rejected point is rolled
  back (slot memory cut or a context checkpoint restored), its token banned and sampled again. Responses carry
  `x_natlang_guidance: {rejections}`; a streamed guided reply arrives with the final chunk.
  `POST /v1/neuralese/guidance/check` checks a fixed reply.
- wllama: the build adds `llama.cpp/vendor/tree-sitter` and `server-guidance.cpp`; `guidance` passes through
  `createChatCompletion` like other request fields.

Verified in headless Chromium against the fork's native `llama-server` (`wllama-fork-smoke.mjs`, LFM2.5-350M F32, CPU):
a guided tool call whose first attempt fails the syntax check is rolled back and equals the native server's reply
and rejections; the other checks still pass (14/14), and the LoRA smoke passes 7/7.
