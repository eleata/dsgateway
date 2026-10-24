# dsgateway — launch note

`dsgateway` is a Rust front end that gives a token-in/token-out inference engine the DeepSeek API surface.
Binaries: `ds41-gateway` and `ds41-paridad`. License Apache-2.0. Author: eleata.

## What problem it addresses

TensorFold's engine serves token ids. Clients (OpenAI SDK, Anthropic SDK, agent frameworks) expect one of three
API shapes. Rewriting the client per engine is avoidable if the conversion happens in front of the engine,
against the engine's own contract.

## What it does

Converts requests with DeepSeek's `deepseek-recipe` 0.2.0, renders and encodes the prompt, calls
`POST /inference/v1/generate`, and reassembles the token stream into the API's body or SSE. Requests the
conversion cannot represent are forwarded unchanged with `x-ds41-gateway: fallback` and a log line.
`--strict` returns the API's 400 instead of falling back.

## Measurements

Two DGX Sparks, DeepSeek-V4.1-Flash-1017 EXL3 2.9 bpw, TP=2, 2026-10-23. Same prompts, three runs, alternated.

- Decode: 78.4 tok/s (code) and 66.1 tok/s (prose) through the gateway, against 67.3 and 58.2 directly.
- Four concurrent requests: 213.5 tok/s aggregate through the gateway, 104.9 directly.
- Prefill of a 151,058-token prompt: 76.2 s through the gateway, 76.8 s directly.
- Gateway RSS: 235 MiB idle, 306 MiB after a 188,795-token prompt.
- Bit-exactness: 31 of 31 fixed prompts identical to the previous engine build (ids, text, finish reason,
  cached tokens), including a 99,916-token prompt and resumable prefixes.

Ranges, method and what is not covered are in `BENCHMARKS.md`. One machine, one checkpoint, no multi-hour soak;
TTFT is only observable through the gateway.

## Engine-side gaps

Measured and reported as `ds41-paridad` cases, not fixed in this repository:

- Reasoning effort: the engine's native table is 25/50/62/75/88/100; the API's is 50/50/75/75/75/100.
  `API_GATEWAY=1` selects the API table on the engine (`TF_DS_API_EFFORTS=1`).
- Tool grammar and `response_format`: not applied by the engine's `/tokenize`; applied by the gateway's renderer.
- `chat-asistente-final-sin-prefix`: trailing newline differs between the two renderers.

## Operational constraints

- 13.9 MiB image, read-only, uid 10001, no Python, no OpenCV, no CUDA; base images pinned by digest.
- Image destination filter covers IPv4 and IPv6 private, loopback, link-local, CGNAT, reserved, documentation
  and tunnelling ranges, on every redirect and every resolved address.
- Parity battery of 68 contract cases runs without GPU against a scripted engine.

---

eleata
