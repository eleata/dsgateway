# Benchmarks

All numbers were measured on a two-DGX-Spark pair serving DeepSeek-V4.1-Flash-1017 (EXL3 2.9 bpw, TP=2 over
RoCE), on 2026-10-23. Each figure is the median of three runs, alternating the gateway and the engine, with
identical prompts. Nothing here is estimated.

Reproduce: `python3 medir.py` (in the measurement bundle) against `--gateway http://127.0.0.1:8888` and
`--zig http://127.0.0.1:8887`.

## Gateway overhead

### Decode, concurrency 1 (tok/s)

| Prompt | Gateway | Engine direct | Difference |
| --- | --- | --- | --- |
| Code | 78.4 (77.7-78.5) | 67.3 (66.8-67.9) | +16.5 % |
| Structured | 72.1 (62.8-72.6) | 63.5 (63.3-63.5) | +13.5 % |
| Prose | 66.1 (65.9-66.4) | 58.2 (46.9-58.8) | +13.6 % |

### Decode, 4 concurrent requests, aggregate (tok/s)

| Prompt | Gateway | Engine direct |
| --- | --- | --- |
| Code | 213.5 | 104.9 |
| Structured | 166.2 | 74.0 |
| Prose | 70.9 | 157.3 |

Prose is the one shape where the direct engine aggregates better: prose answers diverge on the API template
and the gateway serialises more per request. Single-request decode is still faster through the gateway.

### Prefill, long context (seconds)

| Target | Real prompt tokens | Gateway | Engine direct |
| --- | --- | --- | --- |
| 8k | 9,476 | 12.85 | 6.90 |
| 32k | 37,784 | 5.99 | 7.34 |
| 128k | 151,058 | 76.23 | 76.83 |

### Memory

| Moment | Gateway RSS | Host MemAvailable |
| --- | --- | --- |
| Idle | 235 MiB | 6,292 MiB |
| After a 188,795-token prompt | 306 MiB | ~6,600 MiB |

A single large request adds about 70 MiB. The container default is 1 GB; concurrent requests over 100k tokens
need 2 GB.

## Bit-exactness against the previous build

31 fixed prompts, recorded with the previous engine image and replayed against the new one: identical token ids,
text, finish reasons and cache hits in **31 of 31**, including a 100k-token prompt, four concurrent requests,
three interleaved conversations and resumable prefixes with 2,048-6,144 cached tokens.

## Parity battery

68 contract cases on the real pair: 61 comply; 25 diverge in ways the contract already expects (tool grammar 13,
`response_format` 3, prefix 2, spellings 1, system 1, other 5). The 7 that do not comply:

| Case | Class | What the engine does |
| --- | --- | --- |
| `chat-think-minimal`, `chat-think-medium`, `messages-thinking`, `responses-reasoning-medium` | reasoning effort | renders 25/62 instead of 50/75 |
| `chat-think-xhigh` | reasoning effort | renders 88 instead of 75 |
| `chat-asistente-final-sin-prefix` | system | trailing newline differs |
| `chat-tool-result-sin-id` | fallback | returns 400 instead of falling back to the engine |

These are engine-template gaps, not gateway defects; they are tracked for the engine's next round.

## What these numbers do not cover

- One machine, one model, one checkpoint. No cross-hardware comparison.
- TTFT is only measurable through the gateway: the engine does not emit the first token over SSE.
- Prose aggregates worse at concurrency 4; that is real and reported as is.
- No long soak test, no multi-hour run.
