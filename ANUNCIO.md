# dsgateway — the DeepSeek API in front of an engine that only speaks tokens

*A launch note, written to be published. Signed by eleata.*

---

## The problem

Local inference on DGX Sparks got fast enough to be the default answer for a lot of work. The engines that
run there are excellent at one thing: turning token ids into token ids. What they are not is an API.

Every client you already have — the OpenAI SDK, the Anthropic SDK, an agent framework, a chat UI — speaks one
of three shapes. An engine that speaks none of them means either rewriting the client or putting something in
the middle. Until now there was nothing in the middle that understood the actual contract.

## What we shipped

`dsgateway` is that middle layer, written in Rust, 13.9 MiB, no Python, no OpenCV, no GPU, running read-only
as an unprivileged user.

Point a client at `:8888` and it gets Chat Completions, Anthropic Messages, Responses, `count_tokens`,
streaming, tools and images. The prompt is converted with DeepSeek's own `deepseek-recipe` library, rendered
and encoded on the gateway, and handed to the engine as token ids. The engine never sees text, and it never
changes.

## What we measured

We did not ship a claim, we shipped a measurement. On a two-Spark pair serving DeepSeek-V4.1-Flash-1017:

- **No overhead.** Decode runs at 78.4 tok/s through the gateway against 67.3 direct, 66.1 against 58.2 on
  prose, and four concurrent requests aggregate 213.5 tok/s against 104.9 direct.
- **No drift.** 31 of 31 fixed prompts produce identical token ids, text and finish reasons against the
  previous engine build, including a 100k-token context and resumable conversations.
- **No surprises in memory.** 235 MiB idle, 306 MiB after a 188k-token prompt.
- **68 contract cases** that classify every divergence against the engine, by class and by the file
  responsible. 61 comply; the 7 that do not are engine template gaps, listed publicly.

The full tables, the method and what the numbers do not cover are in `BENCHMARKS.md`. The ones that do not
cover are stated as plainly as the ones that do.

## Why it is built this way

- **The engine stays the engine.** One optional patch adds the token route; nothing else in the engine moves.
- **The fallback is bounded.** Requests the conversion cannot represent go to the engine unchanged, with a
  header and one log line. A real error never becomes a successful response.
- **Images cannot leak your network.** Private, loopback, link-local, CGNAT, reserved, documentation and
  tunnelling destinations are rejected for IPv4 and IPv6, on every redirect and every resolved address.
- **It is small on purpose.** No Python, no OpenCV, no GPU: the image is 13.9 MiB and its runtime dependencies
  are the C library and the loader.

## What is next

The parity battery points at the engine now. Reasoning-effort table, tool grammar and `response_format`
rendering are template gaps on the engine side, and they are the next round of work, not a gateway problem.

## Try it

```sh
cargo build --release --locked --bin ds41-gateway --bin ds41-paridad
ds41-paridad casos --casos casos
```

It runs on CPU against a scripted engine with no model weights, so you can see the whole contract work before
you have a GPU.

---

*eleata* · [github.com/eleata](https://github.com/eleata)
