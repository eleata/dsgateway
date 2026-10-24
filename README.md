# dsgateway

DeepSeek API front end for TensorFold's token-in/token-out route.

| | |
| --- | --- |
| Language | Rust 1.97.1 (`rust-toolchain.toml`) |
| Image | `DS41` gateway + `ds41-paridad`, Debian bookworm-slim, uid 10001 |
| Runtime deps | `libc`, loader, `ca-certificates`; no Python, no OpenCV, no CUDA |
| API surface | `/v1/chat/completions`, `/v1/messages`, `/v1/messages/count_tokens`, `/v1/responses`, `/health` |
| Engine contract | `POST /inference/v1/generate` (token ids in, token ids out) |
| License | Apache-2.0 (`LICENSE`, `NOTICE`) |
| Author | eleata |

## What it is

The engine serves token ids. `dsgateway` accepts requests in the DeepSeek API's shapes, converts them with
DeepSeek's `deepseek-recipe` 0.2.0 (`=0.2.0` in `Cargo.lock`), renders and encodes the prompt, sends the ids to
`POST /inference/v1/generate`, and reassembles the returned token stream into the API's body or SSE.

The conversion is lossy for a fixed set of fields that only the engine represents (see Limits). Those requests
are forwarded to the engine unchanged, carrying `x-ds41-gateway: fallback` and one stderr line with the reason.
`--strict` returns the API's 400 instead of falling back.

## Measured

Two DGX Sparks (GB10, 128 GB each, ConnectX-7), DeepSeek-V4.1-Flash-1017 EXL3 2.9 bpw, TP=2, 2026-10-23.
Same prompts, three runs, gateway and engine alternated. Medians; ranges in `BENCHMARKS.md`.

| | `:8888` gateway | `:8887` engine |
| --- | --- | --- |
| Decode, code prompt | 78.4 tok/s | 67.3 tok/s |
| Decode, prose prompt | 66.1 tok/s | 58.2 tok/s |
| Decode, 4 concurrent, aggregate | 213.5 tok/s | 104.9 tok/s |
| Prefill, 151,058-token prompt | 76.2 s | 76.8 s |
| RSS, idle / after a 188,795-token prompt | 235 / 306 MiB | n/a |

Bit-exactness: 31 fixed prompts recorded against the previous engine build; token ids, text, finish reason and
cached tokens identical in 31 of 31, including a 99,916-token prompt, four concurrent requests, three
interleaved conversations and resumable prefixes with cached tokens.

Not covered: one machine, one checkpoint, no multi-hour soak; TTFT is only observable through the gateway
because the engine does not emit the first token over SSE.

## Known limits

Engine-side, tracked as `ds41-paridad` cases rather than gateway defects:

- Reasoning effort: the engine's native table renders 25/50/62/75/88/100 for minimal/low/medium/high/xhigh/max;
  the DeepSeek API table is 50/50/75/75/75/100. The engine selects the API table when it starts with
  `TF_DS_API_EFFORTS=1`, which is how the gateway deployment starts it; the native table is what the engine
  renders alone.
- Tool grammar and `response_format`: the engine's `/tokenize` does not apply either; the gateway's renderer does.
  Cases `chat-tools-*`, `chat-json-*` report class `herramientas` / `formato`.
- `chat-asistente-final-sin-prefix`: trailing newline differs between the two renderers.

Gateway-side: fields the engine alone represents go to the engine unchanged (`logprobs` while the engine reports
`features.logprobs: false`, `guided_grammar`, `thinking_budget`, `previous_response_id`). The engine's `top_k`
and `min_p` apply.

## Routes

| Route | Protocol |
| --- | --- |
| `POST /v1/chat/completions` | Chat Completions; tools, streaming, `stream_options` |
| `POST /v1/messages` | Messages; `tool_use`, `stop_sequences`, streaming |
| `POST /v1/messages/count_tokens` | token counting |
| `POST /v1/responses` | Responses |
| `GET /health` | gateway and engine status |
| `POST /ds41/render/{chat,messages,responses}` | render without generating |

## Build and run

```sh
cargo build --release --locked --bin ds41-gateway --bin ds41-paridad
```

The repository ships `tok/tokenizer.json`, so the test battery runs with no environment variable and no
checkout outside this directory:

```sh
cargo test --locked
```

`ds41-paridad` needs no model weights. `casos` validates the 68 case files; `render` and `e2e` compare against a
running pair and take the engine and gateway URLs:

```sh
ds41-paridad casos
ds41-paridad render --casos casos --zig http://127.0.0.1:8887 \
  --gateway http://127.0.0.1:8888 --tokenizer tok/tokenizer.json --salida informes/cpu
ds41-paridad e2e --casos casos --gateway http://127.0.0.1:8888 \
  --zig http://127.0.0.1:8887 --tokenizer tok/tokenizer.json --salida informes/cpu
```

`render` and `e2e` need both sides answering: the engine's `POST /inference/v1/generate` and the gateway in front
of it. Without a GPU, a scripted token-in server that implements that route and `/tokenize` is enough.

## Security

- Image: uid 10001, both base images pinned by digest, `LICENSE` and `NOTICE` at `/usr/share/doc/dsgateway/`.
- The container expects to run read-only with `tmpfs` at `/tmp`, a memory cap and a CPU cap; `docker run` in
  `docs/api-gateway.md` shows the flags. The tokenizer is read from a read-only mount.
- Image destinations: IPv4 and IPv6 loopback, private, link-local, CGNAT, reserved, documentation, benchmarking,
  discard, translation and tunnelling ranges rejected; every redirect and every resolved address checked.
- Only image dimensions are decoded, never pixels; 600 images and 64 MiB per request.

## Configuration

Flags in `src/main.rs`; defaults are the API's. `--strict`, `--block-private-urls`, `--api-sampling`,
`--max-images`, `--max-image-mb`, `--log`, `--require-key`. `--tokenizer` defaults to the copy inside the image at
`/served/tokenizer.json`; a bind mount, or `--tokenizer` with another path, replaces it. Memory and CPU caps are
docker flags, not process flags: 1 GB fits one 188k-token request, 2 GB with concurrency over long contexts.

## License

Apache-2.0. `LICENSE` and `NOTICE`; the image carries both at `/usr/share/doc/dsgateway/`.

---

<sub>eleata</sub>
