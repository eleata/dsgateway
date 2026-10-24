# dsgateway

**The DeepSeek API, in front of an engine that only speaks tokens.**

`dsgateway` is a Rust service that gives a token-in/token-out inference engine the public DeepSeek API
surface: Chat Completions, Anthropic Messages, Responses, `count_tokens`, streaming, tools and images.
You point a client at `:8888` and it behaves like a DeepSeek endpoint. The engine underneath never
changes and never sees text.

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.97.1-orange.svg)](rust-toolchain.toml)
[![Tests](https://img.shields.io/badge/tests-113%20passing-brightgreen.svg)](#parity-battery)
[![Image](https://img.shields.io/badge/image-13.9%20MiB%2C%20no%20Python%2C%20no%20GPU-informational.svg)](Dockerfile)

## Why it exists

Local inference engines on DGX Sparks are fast and cheap, but they speak their own protocol. Every client,
every agent framework and every SDK expects one of three API shapes, and rewriting the client is how a
week of work disappears.

`dsgateway` closes that gap without touching the engine: the prompt is converted with DeepSeek's own
`deepseek-recipe` library, rendered and encoded on the gateway, and sent to the engine as token ids. The
token stream comes back and is reassembled into the API's body or SSE. Requests the conversion cannot
represent faithfully fall back to the engine unchanged, with a header and one log line saying why.

## Measured, not claimed

Everything below was measured on a two-Spark pair (DGX Spark over RoCE) serving DeepSeek-V4.1-Flash-1017,
comparing the gateway against the engine directly, identical prompts, three runs each:

| | Gateway `:8888` | Engine direct `:8887` |
| --- | --- | --- |
| Decode, code prompt (tok/s) | **78.4** | 67.3 |
| Decode, prose prompt (tok/s) | **66.1** | 58.2 |
| Four concurrent requests, aggregate (tok/s) | **213.5** | 104.9 |
| Prefill, 151k-token prompt (s) | 76.2 | 76.8 |
| Gateway RSS, idle / 188k-token prompt | 235 MiB / 306 MiB | n/a |

**Bit-exact:** 31 of 31 fixed prompts produce identical token ids, text, finish reasons and cache hits
against the previous engine build, including 100k-token contexts and resumable conversations.

**Parity:** a 68-case contract battery classifies every divergence against the engine, by class and by the
file responsible. 61 cases comply; the 7 that do not are engine template gaps, listed in
[BENCHMARKS.md](BENCHMARKS.md), not gateway defects.

See [BENCHMARKS.md](BENCHMARKS.md) for the full tables, the method and what the numbers do not cover.

## What it serves

| Route | Protocol |
| --- | --- |
| `POST /v1/chat/completions` | OpenAI Chat Completions, with tools, streaming and `stream_options` |
| `POST /v1/messages` | Anthropic Messages, with `tool_use`, `stop_sequences` and streaming |
| `POST /v1/messages/count_tokens` | Anthropic token counting |
| `POST /v1/responses` | OpenAI Responses |
| `GET /health` | gateway and engine health |
| `POST /ds41/render/{chat,messages,responses}` | prompt rendering without generating |

Responses from the converted path carry `x-ds41-gateway: api`. Fallback responses carry
`x-ds41-gateway: fallback` and a log line with the reason. `/inference/*` is never exposed.

## Design constraints that hold

- **No Python, no OpenCV, no GPU** in the image: 13.9 MiB, `linux-vdso`, `libm`, `libc` and the loader, nothing else.
- **Read-only container**, unprivileged user, `tmpfs` at `/tmp`, both base images pinned by digest.
- **Images are filtered**: private, loopback, link-local, CGNAT, reserved, documentation and tunnelling
  destinations are rejected for IPv4 and IPv6, every redirect and every resolved address.
- **No silent errors**: a real error never becomes a successful response; the fallback is bounded and logged.

## Build

Rust 1.97.1 is pinned in `rust-toolchain.toml`. On a host with a C toolchain:

```sh
cargo build --release --locked --bin ds41-gateway --bin ds41-paridad
```

The tests that need a tokenizer read `DS41_TOKENIZER` (a `tokenizer.json` path); without it they fall back to
`../tok/tokenizer.json` next to this folder, so `cargo test --locked` needs either the variable or a `tok/` checkout.

The container image is built by the recipe's `scripts/prepare.sh` from `Dockerfile` (multi-stage
`rust:1.97-bookworm` to `debian:bookworm-slim`, unprivileged user, `ca-certificates`, both binaries) with the
tag `GATEWAY_IMAGE`, which carries a hash of this folder. The Dockerfile builds `ds41-gateway` and
`ds41-paridad` and copies `casos/`, so the parity battery runs from the image too.
## Run with the recipe (GPU)

Put `API_GATEWAY=1` in `scripts/local.sh` and run `./start.sh` (or `./start.sh restart`). The recipe builds the gateway
image if needed, starts the engine on `127.0.0.1:ENGINE_PORT` (default 8887) with `TENSORFOLD_TOKEN_ROUTE=1`, starts
`ds41-gateway` on `HOST:PORT`, waits for `/health`, and runs the smoke tests. `./stop.sh` removes the gateway first and
saves its log. `API_GATEWAY=0` is the recipe as before.

## Run without a GPU

The gateway only needs an engine that speaks `POST /inference/v1/generate`. For a CPU check, start a fake token-in
engine and then the gateway; the fake engine is the TensorFold checkout's `fake_serve` (`zig/tests/server/build.sh`,
then `zig-out/server/fake_serve serve zig/tests/server/fixtures`):

```sh
TF_FAKE_TOKENIZER=tok/ TENSORFOLD_TOKEN_ROUTE=1 TF_FAKE_CONTEXT=65536 zig-out/server/fake_serve serve zig/tests/server/fixtures &
./target/release/ds41-gateway --listen 127.0.0.1:8888 --engine http://127.0.0.1:8887 \
  --tokenizer tok/tokenizer.json --served-name DeepSeek-V4.1-Flash-1017
curl -s http://127.0.0.1:8888/health
```

Other engine routes are proxied; `/inference/*` is not exposed. `GATEWAY_STRICT=1` returns the API's 400 instead of
falling back.

## Settings

The defaults are in the recipe's `scripts/config.sh`; the flags are in `src/main.rs`. The important ones:

| Setting | Flag | Default |
| --- | --- | --- |
| `HOST`, `PORT` | `--listen` | `0.0.0.0:8888` |
| `ENGINE_PORT` | `--engine` | `http://127.0.0.1:8887` |
| `SERVED_NAME` | `--served-name` | `DeepSeek-V4.1-Flash-1017` |
| `THINKING`, `REASONING_EFFORT` | `--default-thinking`, `--default-effort` | 1, empty |
| `CONTEXT` | `--max-model-len` | auto from the engine, 65536 if it does not answer |
| `GATEWAY_IMAGE_TOKEN` | `--image-token` | `<｜deepseek_image｜>` |
| `GATEWAY_STRICT` | `--strict` | 0 |
| `GATEWAY_BLOCK_PRIVATE_URLS` | `--block-private-urls` | 1 |
| `GATEWAY_API_SAMPLING` | `--api-sampling` | 1 |
| `GATEWAY_MAX_IMAGES`, `GATEWAY_MAX_IMAGE_MB` | `--max-images`, `--max-image-mb` | 600, 64 |
| `GATEWAY_LOG` | `--log` | info |
| `API_KEY` | `--require-key` | 0 |
| `GATEWAY_MEMORY` | container `--memory` | 1g (2g for contexts over 100k in concurrency) |

With `API_KEY=1` the recipe mounts the key at `/run/secrets/key` and passes `--require-key`; the gateway forwards
`Authorization` / `x-api-key` to the engine and lets it decide.

## Images

`image_url`, `input_image` and Messages image blocks are fetched by the gateway and counted with DeepSeek's
`ImageTokenSpec::v41`; only their dimensions are decoded, never their pixels. With `GATEWAY_BLOCK_PRIVATE_URLS=1` (the
default) the destination filter rejects, for IPv4: `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10`, `127.0.0.0/8`,
`169.254.0.0/16`, `172.16.0.0/12`, `192.0.0.0/24`, `192.0.2.0/24`, `192.31.196.0/24`, `192.52.193.0/24`,
`192.88.99.0/24`, `192.168.0.0/16`, `192.175.48.0/24`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`,
`224.0.0.0/4`, `240.0.0.0/4` and the broadcast address; for IPv6: `::/128`, `::1/128`, `::ffff:0:0/96`,
`64:ff9b::/96`, `64:ff9b:1::/48`, `100::/64`, `2001::/23`, `2001:db8::/32`, `2002::/16`, `3fff::/20`, `fc00::/7`,
`fe80::/10`, `fec0::/10` and `ff00::/8`. IPv4-mapped and 6to4 addresses are checked through their embedded IPv4
address. Every redirect and resolved address is checked, so an image URL cannot make the gateway fetch a private or
non-routable destination. `GATEWAY_MAX_IMAGES` (600), `GATEWAY_MAX_IMAGE_MB` (64) and 8192-character URLs still apply.

## Parity battery

`ds41-paridad` compares the gateway's API rendering with TensorFold's token route and checks the API's shape against a
scripted engine. It needs no model weights:

```sh
ds41-paridad casos --casos casos
ds41-paridad render --casos casos --zig http://127.0.0.1:ENGINE_PORT \
  --gateway http://127.0.0.1:PORT --tokenizer tok/tokenizer.json --salida informes/cpu
ds41-paridad e2e --casos casos --gateway http://127.0.0.1:PORT \
  --zig http://127.0.0.1:ENGINE_PORT --tokenizer tok/tokenizer.json --salida informes/cpu
```

`casos` validates the case files; `render` compares the API path with the native `/tokenize` route and reports the
first differing index, the ids, the class and the responsible file; `e2e` checks the message, usage, stream, Messages,
Responses and context-error invariants and the token hashes. Each run writes `informes/cpu/informe.json` and
`informes/cpu/informe.html`. The exit code is 0 when everything passes, 1 for divergences and 2 for a setup error.
The contract is summarized in `docs/api-gateway.md`.

## Limits

Some fields are engine-only and go to the engine unchanged: `logprobs`, grammar/structured output,
`thinking_budget` and `previous_response_id`. The engine's own `top_k` and `min_p` still apply. The known
engine-side template gaps (reasoning-effort table, tool grammar, `response_format` rendering) are listed in
[BENCHMARKS.md](BENCHMARKS.md).

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

---

<sub>Built by <b>eleata</b> · <a href="https://github.com/eleata">github.com/eleata</a></sub>
