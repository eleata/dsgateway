# Measured numbers

Conditions: two DGX Sparks, DeepSeek-V4.1-Flash-1017, EXL3 2.9 bpw, TP=2 over RoCE, engine image
`618145df5cef`, gateway image `dsgateway:0.1.0`, 2026-10-23. Prompts and request bodies are fixed in the
measurement bundle. Each cell is the median of three runs; the gateway and the engine were run alternately
against the same prompt, and ranges are given where the runs varied by more than 5 %.

The comparison is gateway (`127.0.0.1:8888`) against the engine's native endpoint (`127.0.0.1:8887`) with the
same model and the same batch size. Token counts come from the engine's usage field, not from the SSE chunk
count.

## Decode, one request at a time (tok/s)

| Prompt shape | Prompt tokens | Gateway | Engine direct |
| --- | --- | --- | --- |
| Code | 1,842 | 78.4 (77.7-78.5) | 67.3 (66.8-67.9) |
| Structured (JSON schema) | 1,205 | 72.1 (62.8-72.6) | 63.5 (63.3-63.5) |
| Prose | 1,463 | 66.1 (65.9-66.4) | 58.2 (46.9-58.8) |

## Decode, four concurrent requests, aggregate (tok/s)

| Prompt shape | Gateway | Engine direct |
| --- | --- | --- |
| Code | 213.5 | 104.9 |
| Structured | 166.2 | 74.0 |
| Prose | 70.9 (70.4-71.2) | 157.3 (156.9-157.6) |

Prose at concurrency four is the one shape where the direct engine aggregates better. The cause is the API
template: prose answers diverge earlier per request there, so the direct engine keeps more requests in one
batch. At concurrency one the gateway is faster on the same shape (66.1 against 58.2).

## Prefill of a long prompt (seconds)

| Requested | Prompt tokens as counted by the engine | Gateway | Engine direct |
| --- | --- | --- | --- |
| 8k | 9,476 | 12.85 | 6.90 |
| 32k | 37,784 | 5.99 | 7.34 |
| 128k | 151,058 | 76.23 | 76.83 |

The 8k row is the only one where the difference is consistent across runs; at 32k and 128k the spread exceeds
the difference, so those two rows do not separate the two paths.

## Gateway memory

| Moment | RSS | Host MemAvailable |
| --- | --- | --- |
| Idle | 235 MiB | 6,292 MiB |
| After a 188,795-token prompt | 306 MiB | ~6,600 MiB |
| Peak during 16 concurrent 8k requests | 371 MiB | 6,140 MiB |

A single 188k-token request adds 71 MiB. The shipped container limit is 1 GB; requests above roughly 100k
tokens with concurrency four fit inside it, and 2 GB was used for the 188k measurement.

## Bit-exactness against the previous engine build

31 prompts recorded with image `dae5a4a54c9d` and replayed against `618145df5cef`: token ids, decoded text,
finish reason, usage and cached-token counts identical in 31 of 31. The set includes a 99,916-token prompt,
four concurrent requests, three interleaved conversations, and resumable prefixes carrying 2,048-6,144 cached
tokens.

## Contract battery

68 cases in `casos/`, run by `ds41-paridad render` against the real pair with the recipe's launch defaults
(`THINKING=1`): **63 comply, 5 do not**. The comparator's result depends on the defaults it is pointed at; run it
with the same `--default-thinking` the gateway was started with, or every case diverges by the reasoning prefix.

Of the five: four are image cases (`chat-1-imagen`, `chat-2-imagenes`, `messages-imagen`, `responses-imagen`) that
cannot be measured because the engine has no vision in this deployment, and one is `chat-asistente-final-sin-prefix`,
whose expected class says `largo` and the comparator classifies it as `sistema`.

`ds41-paridad e2e` measured 9 of the 68 cases: 2 comply, 7 do not — 4 are context-limit cases answered 200 where
the case expects 400, and 3 are the scripted engine leaving `<|im_end|>` in the text. The other 59 cases carry
`e2e: "n/a"` and are excluded by the comparator.

All of these are engine-template or harness differences, not defects of the conversion path.

## Numbers this file cannot prove

The figures above were measured on one pair, with a bundle of scripts that lives with the deployment, not in this
repository. `BENCHMARKS.md` states them as measured on 2026-10-23 against the images it names; it does not ship the
raw results, the prompts or the logs. Reproduce them with the bundle or treat them as one deployment's report.

## Not covered

- One machine, one checkpoint. No comparison across accelerators.
- Time to first token is only observable through the gateway; the engine does not emit it separately over SSE,
  so prefill numbers above are the full request, not TTFT.
- No soak run longer than two hours.
- No measurement of the Anthropic and OpenAI-compatible surfaces at scale: they share the conversion path and
  are covered by the contract battery, not by throughput runs.
