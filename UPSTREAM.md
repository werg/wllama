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
