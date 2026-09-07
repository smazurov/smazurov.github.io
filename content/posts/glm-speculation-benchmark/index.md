---
title: "Speculative Decoding on GLM-5.3-Flash: DFlash vs Native MTP"
date: 2026-09-07T11:00:00-07:00
draft: false
description: "A four-arm benchmark of speculative decoding on GLM-5.3-Flash (NVFP4, TP2 over two GB10 nodes): DFlash BF16, DFlash FP8 draft KV, native MTP, and no speculation. Native MTP's bundled draft head costs about 20% of short-prompt throughput and leaves an 89% larger KV pool - a good trade for long-context serving."
tags:
  - sglang
  - llm
  - benchmark
  - speculative-decoding
  - self-hosting
takeaways:
  - The native MTP head drafts with the target's own layers, so its draft KV costs far less pool than an external drafter's: 982k profiled KV tokens vs 520k with DFlash BF16
  - DFlash BF16 fits four ~128k sessions with 12 tokens of headroom; MTP holds the same four with 462k spare
  - The throughput cost is moderate: about 20% on short prompts, 16% on two-session 128k cached decode
  - MTP accepted 2.5-2.7 of 3 draft tokens on average across workloads (80-100% peak acceptance rate)
  - No speculation has the biggest pool (1.7M tokens) but the slowest decode in every phase
tested_on:
  - SGLang (pinned serving image, fc3c6bf6)
  - GLM-5.3-Flash-NVFP4-Spark, revision 53e77dbb
  - 2x NVIDIA GB10 Spark nodes
  - incoai/GLM-5.3-Flash-DFlash2, revision bf582e4
---

Serving GLM-5.3-Flash on two GB10 Spark nodes means every speculation choice is a trade between single-stream speed and KV memory. A draft model accelerates decode, but the draft's KV cache eats pool that the target could use for context. We benchmarked four serving configurations against the same workloads to find out which one to run.

## The Four Arms

GLM-5.3-Flash ships with a native MTP head (NEXTN) bundled in the checkpoint, so serving has two ways to draft: an external drafter, or the model's own prediction head. Every drafter has a price - DFlash2's draft weights and draft KV eat pool the target could otherwise use for context, while the native head reuses the target's own layers, so its draft KV is smaller and there's no extra model to load. This benchmark measures what a drafter costs in speed and KV, and what it buys in capacity.

The target throughout is [GLM-5.3-Flash-NVFP4-Spark][1], [Inco's][7] weight-only NVFP4 quantization of [GLM-5.3-Flash][2] (320B total, 18B active, MIT license), published day-0 with GLM 5.3 for DGX Spark-class Blackwell hardware, at revision `53e77dbb`. All arms ran on the same hardware: two GB10 nodes, TP2, memory fraction 0.90, FP8 target KV, 262,144 per-request context, 4,096-token chunked prefill, four maximum running requests. What varies is the speculation mode:

| Arm | Draft | Draft KV |
|---|---|---|
| DFlash BF16 | [DFlash2][3], 5 tokens, BF16 weights | BF16 |
| DFlash FP8 | [DFlash2][3], 5 tokens | FP8 E4M3 |
| Native MTP | checkpoint's bundled NEXTN head, 3 tokens | FP8 E4M3 |
| No speculation | none | none |

The DFlash2 drafter is a separate block-diffusion draft model from [Inco AI][4]; native MTP uses the checkpoint's own bundled NEXTN head, so it carries no draft weights of its own. All arms got a 2,097,152 configured pool ceiling so the memory profiler could reveal what each mode actually fits.

The Inco checkpoint needed small Python image overlays (no base-image replacement) to load at all: mixed-quantization routing fixes, MXFP8 scale loading, narrow projection padding, and MLA absorption corrections. That engineering is a separate story - the numbers below are what it bought.

## What Was Measured

Throughput is **end-to-end output tokens / phase wall time**, not steady-state decode rate. Per arm:

1. Correctness probes - arithmetic, exact JSON, tool-call parsing, a generated image. Plus embedded-secret recall in the long prompts.
2. Short prompts - one warmup plus three measured 512-output-token requests each for code and prose.
3. Cold waves of distinct ~128,000-token synthetic documents at 1, 2, and 4 concurrent requests, 512 output tokens each.
4. Cached repeats of the same waves with 2,048 output tokens each - long enough to measure resident long-session decode, not just the prefill.

The harness reads cache-hit counts from the server for every request. That matters because an undersized pool can evict documents during the preceding cold wave, and a warm-cache assumption would hide it.

## Results

| Configuration | Actual KV pool (tokens) | Short code tok/s | Short prose tok/s | 128k repeat C1 tok/s | Repeat C2 aggregate tok/s |
|---|---:|---:|---:|---:|---:|
| DFlash BF16 | 520,192 | 32.35 | 26.65 | 32.75 | 46.06 |
| DFlash FP8 | 738,624 | 30.60 | 27.49 | 33.01 | 49.78 |
| Native MTP | 982,464 | 25.75 | 23.51 | 24.70 | 38.73 |
| No speculation | 1,703,232 | 14.74 | 14.66 | 14.60 | 24.57 |

Two things jump out. First, DFlash BF16 was the fastest arm on short prompts - no speculation mode beats plain DFlash for single-stream speed here. Second, native MTP is 25.75 tok/s on short code, about 20% lower than DFlash BF16. The same ordering holds on cached 128k repeats: the DFlash arms decode 32-33 tok/s single-stream and 46-50 aggregate at two concurrent sessions, MTP does 24.7 and 38.7. Every arm completed these waves with full cache hits and no queuing, so the comparison is clean.

## MTP Acceptance

The native MTP head's acceptance behavior was consistent across workloads:

| Workload | Avg accepted length (of 3) | Max accept rate |
|---|---:|---:|
| Short code | 2.68 | 0.90 |
| Short prose | 2.48 | 0.80 |
| 128k cold C4 | 2.62 | 0.91 |
| 128k cached C4 | 2.58 | 1.00 |

Acceptance held up at 128k context and under concurrent load. It doesn't degrade where it matters.

## What the Pool Buys

The pool is the budget for resident context. A session occupies its input plus the 2,048-token answer, so the pool divides into nominal session counts:

| Configuration | KV pool (tokens) | 128k sessions | 256k sessions | 512k sessions |
|---|---:|---:|---:|---:|
| DFlash BF16 | 520,192 | 4.0 | 2.0 | 1.0 |
| DFlash FP8 | 738,624 | 5.7 | 2.9 | 1.4 |
| Native MTP | 982,464 | 7.6 | 3.8 | 1.9 |
| No speculation | 1,703,232 | 13.1 | 6.6 | 3.3 |

The DFlash BF16 numbers have no slack in them: four 128k sessions need 520,180 tokens against a 520,192 pool, and two 256k sessions need 516,096 against the same pool. Either way there's essentially nothing left for a conversation to grow or a document to stay cached while another loads. MTP holds the same four 128k sessions with 462k left over, or three 256k sessions with 208k to spare.

One config note: the tested serving setup caps per-request context at 262,144, so 256k sessions fit as-is while 512k sessions would need the context limit raised. The 512k column is pool arithmetic showing the ceiling, not a configuration that was served.

Measured at the small end, where the data is clean, the cost side: with warm cache at one and two concurrent 128k sessions, MTP decodes 24.70 tok/s single-stream and 38.73 aggregate at two, against DFlash BF16's 32.75 and 46.06. Call it 16% at two sessions. The pool bought 89% more context budget. That's the trade: a moderate, predictable throughput cost for headroom DFlash BF16 structurally can't offer.

## What We Deployed

Native MTP. The cost is moderate - about 20% on short prompts, 16% on two-session 128k decode - and the return is an 89% larger KV pool, which converts directly into room for longer conversations or more resident contexts. DFlash BF16 is the pick when single-stream speed dominates and sessions stay pinned at 128k; MTP is the pick when they grow.

## Limits

- Throughput is end-to-end per phase, not steady-state decode rate. Cold and cached waves are reported separately; the reported cached waves ran with full cache hits and no queuing in every arm.
- Each arm ran once, with no repeat runs. The four-concurrent cached wave produced scheduler-dependent cache-miss behavior we can't fully explain, so this post reports only the clean 1-2 session waves and treats the session counts above as nominal arithmetic, not measured admission.
- Synthetic archival documents aren't real agent-history workloads. No contexts beyond ~128k were tested, no client-side compaction, no single-node execution, and the pool arithmetic doesn't imply working 240k sessions.

## FAQ

**Why does MTP have a bigger KV pool than DFlash?**
The MTP head reuses the target model's layers for drafting, so its draft KV is smaller and cheaper than a separate DFlash draft model's. FP8 draft KV helps too, but DFlash FP8 shows that alone isn't the difference.

**Is native MTP the same as EAGLE?**
SGLang resolves the bundled NEXTN/MTP head to its EAGLE implementation internally. Functionally it's the checkpoint's own prediction head, not an external draft model like DFlash2.

**Where's the model?** GLM-5.3-Flash weights are on [Hugging Face][2] under MIT, with the [Z.ai announcement][5] and the [GLM-5 technical report][6]. The [NVFP4 checkpoint][1] and the [DFlash2 drafter][3] are also on Hugging Face. The `-Spark` suffix in the repo name refers to DGX Spark-class hardware packaging, not a different model.

**Why two ways to draft?** Because the checkpoint ships with a native MTP head, the comparison is between using the model's own head versus loading a dedicated external drafter. DFlash2 drafts 5 tokens per pass and accepts more per step on paper; native MTP drafts 3 and carries no extra weights. The benchmark measures what those differences actually do to serving capacity.

**Would results change with more nodes or different parallelism?**
Almost certainly. TP2 across two nodes is the tested configuration; nothing here extrapolates to TP1 or TP4.

[1]: https://huggingface.co/local-inference-lab/GLM-5.3-Flash-NVFP4-Spark
[2]: https://huggingface.co/zai-org/GLM-5.3-Flash
[3]: https://huggingface.co/incoai/GLM-5.3-Flash-DFlash2
[4]: https://inco.ai/blog/dflash2/
[5]: https://z.ai/blog/glm-5.3-flash
[6]: https://arxiv.org/abs/2602.15763
[7]: https://inco.ai/blog/glm-5-3/
