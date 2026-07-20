---
title: Local LLM Landscape for a Homelab Server
date: 2026-07-19
tags: [ai, models, local-llm, homelab, hardware, self-hosting]
source: https://github.com/ggml-org/llama.cpp, https://github.com/ollama/ollama, https://docs.vllm.ai, https://huggingface.co
verified: 2026-07-19
---

# Local LLM Landscape for a Homelab Server

Practical quick-start survey: which open-weight models are worth running, which serving stack to pick for a headless server, how quantization works, and what hardware it takes. All claims verified against primary sources (official blogs, GitHub repos, Hugging Face model cards) on **2026-07-19** unless flagged otherwise. This space moves fast — treat every version number here as dated.

## Quick-start recommendation (homelab server, July 2026)

- **Runtime:** Run **llama.cpp `llama-server`** or **Ollama** on the headless box; both expose an OpenAI-compatible HTTP API that any client on the LAN can hit. Pick Ollama for convenience (model pulls, auto memory management), raw llama.cpp for control. If you have a proper GPU and multiple concurrent users/apps, step up to **vLLM**.
- **Models to start with** (as of 2026-07-19):
  - 24 GB VRAM (used RTX 3090 class): **Qwen3.6-35B-A3B** (Apache 2.0, MoE — fast because only ~3B active) or **gpt-oss-20b** (Apache 2.0, fits in 16 GB).
  - 12–16 GB: **Ministral 3 14B** or **Gemma 4 E4B / 12B** (both Apache 2.0), **Phi-4 14B** (MIT).
  - 8 GB or CPU-only: **Ministral 3 3B/8B**, **Gemma 4 E2B**, small Qwen3 variants.
- **Quant level:** default to **Q4_K_M** GGUF — the commonly recommended size/quality balance in llama.cpp's own tooling.
- **License check:** Qwen (open releases), Mistral 3, Gemma 4, and gpt-oss are Apache 2.0; DeepSeek is MIT; Phi is MIT. Llama 4 is *not* OSI-open (community license with conditions). For a homelab none of this bites, but it matters if the lab graduates into a product.

---

## 1. Open-weight model families worth running (state: 2026-07-19)

Observation first (what the sources say), conclusions at the end of the section.

### Qwen (Alibaba) — current open flagship for small hardware
- Current open-weight generation: **Qwen3.6-35B-A3B** (MoE, 35B total / 3B active), released April 2026, **Apache 2.0**, native 262,144-token context (up to ~1M with YaRN). Supports thinking and non-thinking modes; pitched at agentic coding. Verified from the [model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) and [official blog](https://qwen.ai/blog?id=qwen3.6-35b-a3b).
- The card references earlier **Qwen3.5** open variants (27B dense, 35B-A3B); secondary sources date Qwen3.5 to Feb 2026 (⚠ exact 3.5 lineup/dates not verified from a primary source).
- ⚠ Secondary sources report Alibaba's newest flagships (Qwen3.7 Max/Plus, May–June 2026) are **API-only, no open weights** — the open line continues at 3.6. Not verified from an Alibaba primary source.
- Good at: broad size ladder, strong coding/agentic tuning, most permissive license among the big families.

### DeepSeek — best open weights, but too big for most homelabs
- **DeepSeek-V4** family, released April 2026, **MIT license**, 1M-token context. Variants on [Hugging Face](https://huggingface.co/deepseek-ai): **V4-Flash** (284B total / 13B active, [model card](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash)) and **V4-Pro** (1.6T total base). Three reasoning modes (Non-think / Think High / Think Max).
- Good at: frontier-class reasoning/agentic work under a truly permissive license. Conclusion: even Flash at 284B total needs ~150 GB+ at 4-bit — this is multi-GPU or big-RAM Mac/EPYC territory, not a single-3090 model.

### Llama (Meta) — current gen is April 2025's Llama 4
- **Llama 4 Scout** (17B active / 109B total, 16 experts, 10M context, multimodal) and **Maverick** (400B total); released 2025-04-05. Verified from [Meta's announcement](https://ai.meta.com/blog/llama-4-multimodal-intelligence/) and the [Scout model card](https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct). No newer open Llama generation found as of 2026-07-19 (Behemoth still unreleased).
- License: **Llama 4 Community License** — commercial use allowed, but requires a separate Meta license above 700M MAU, plus "Built with Llama" attribution and "Llama" naming for derivatives ([model card](https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct)). Not OSI-open.
- Good at: huge context, mature Western ecosystem/tooling. Conclusion: for homelab, Scout needs ~55–60 GB at 4-bit; there is no current-gen small dense Llama, which has pushed homelab users toward Qwen/Gemma/Mistral.

### Mistral — best current dense small models, Apache 2.0
- **Mistral 3** family, released 2025-12-02: **Ministral 3 3B / 8B / 14B** (dense, multimodal image understanding, reasoning variants) and **Mistral Large 3** (MoE, 675B total / 41B active). "All models are released under the Apache 2.0 license." Verified from [mistral.ai/news/mistral-3](https://mistral.ai/news/mistral-3/).
- ⚠ A new open-weight family was in early access July 2026 (secondary press reports); nothing released as of 2026-07-19.
- Good at: efficient dense models sized exactly for consumer GPUs; the 14B reasoning variant claims 85% on AIME '25.

### Gemma (Google) — newly Apache 2.0, strong multimodal small models
- **Gemma 4**, released 2026-04-02 (releases page says initial launch 2026-03-31): sizes **E2B, E4B, 26B-A4B (MoE), 31B dense**, plus **Gemma 4 12B Unified** (text+image+audio in, 2026-06-03). 128K context on edge models, 256K on the larger ones; images/video input on all, audio on E2B/E4B. **Apache 2.0** — a change from earlier Gemma custom terms. Verified from [Google's announcement](https://blog.google/innovation-and-ai/technology/developers-tools/gemma-4/) and the [Gemma releases page](https://ai.google.dev/gemma/docs/releases).
- Good at: multimodal input at small sizes, on-device/edge; unquantized 31B fits a single 80 GB H100, quantized fits consumer GPUs (per Google's announcement).

### Phi (Microsoft) — small MIT-licensed reasoning models
- Current family: **Phi-4** (14B dense, **MIT**, [model card](https://huggingface.co/microsoft/phi-4)) plus Phi-4-mini / Phi-4-multimodal / Phi-4-reasoning variants ([Microsoft announcement](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/introducing-phi-4-microsoft%E2%80%99s-newest-small-language-model-specializing-in-comple/4357090)).
- ⚠ A "Phi-4-reasoning-vision-15B" (March 2026) and pre-release "Phi-5" appear only in secondary sources — not verified from Microsoft primary sources.
- Good at: math/structured reasoning per parameter; clean MIT license.

### gpt-oss (OpenAI) — the surprise Apache 2.0 entry
- **gpt-oss-120b** (117B total / 5.1B active) and **gpt-oss-20b** (21B total / 3.6B active), released 2025-08-05, **Apache 2.0**, MoE weights shipped in **MXFP4** so 120b fits a single 80 GB GPU and 20b runs within 16 GB memory. Configurable reasoning effort, tool use; requires the "harmony" response format. Verified from the [gpt-oss-120b model card](https://huggingface.co/openai/gpt-oss-120b).
- Good at: reasoning + agentic use in a small memory footprint; 20b is one of the best fits for a 16 GB GPU.

### GLM (Z.ai / Zhipu) and Kimi (Moonshot) — frontier open weights, datacenter-sized
- **GLM-5.2**: 753B MoE, MIT, 1M context, June 2026 ([zai-org on HF](https://huggingface.co/zai-org/GLM-5.2)). Smaller **GLM-4.7-Flash** (31B, Jan 2026) exists for consumer hardware.
- **Kimi K2.6 / K2.7-Code**: 1T–1.1T total / 32B active, "Modified MIT License" (MIT plus conditions — read the license file before commercial use), 2026 ([moonshotai on HF](https://huggingface.co/moonshotai/Kimi-K2.6)).
- Conclusion: top-tier open weights, but 0.75–1.1T params ≫ homelab; relevant mainly via cloud inference providers.

**Section conclusion:** for a single-GPU homelab in July 2026 the practical shortlist is **Qwen3.6-35B-A3B, gpt-oss-20b, Ministral 3 (3/8/14B), Gemma 4 (E2B–31B), Phi-4** — all Apache 2.0 or MIT. The frontier open-weight action (DeepSeek V4, GLM-5.2, Kimi K2.x) has moved to sizes that need server-class memory.

## 2. Runtimes / serving stacks

| Stack | What it is | OpenAI-compatible API | Concurrency | Homelab fit |
|---|---|---|---|---|
| llama.cpp | C/C++ inference engine, GGUF | Yes (`llama-server`, `/v1/chat/completions`) | Parallel decoding, multi-user | Headless server, any hardware |
| Ollama | Model manager + server built on llama.cpp | Yes | Modest (sequential-ish, fine for a household) | Easiest headless setup |
| vLLM | Python GPU serving engine (PagedAttention) | Yes | Excellent — continuous batching | GPU server, many clients |
| LM Studio | Desktop app (+ `llmster` headless, `lms` CLI) | Yes | Desktop-grade | Workstation first, server secondary |

- **[llama.cpp](https://github.com/ggml-org/llama.cpp)** (release b10068, 2026-07-18): minimal-dependency C/C++ inference for GGUF models. Backends: CUDA, Metal/Accelerate (Apple Silicon), AMD HIP, Vulkan, SYCL, WebGPU, plain CPU (AVX/AVX2/AVX-512). `llama-server` is a lightweight HTTP server with an OpenAI-compatible chat endpoint, web UI, multi-user parallel decoding, embeddings/reranking, speculative decoding. **Use when:** you want maximum control, oddball hardware, or minimum overhead on a headless box.
- **[Ollama](https://github.com/ollama/ollama)** (v0.32.1, 2026-07-16): model pull/run/manage layer built on llama.cpp, REST API on `localhost:11434`, official Docker image, Python/JS clients. **Use when:** you want `ollama pull gemma4` convenience and automatic model lifecycle on a server; the default choice for "one box, a few clients."
- **[vLLM](https://docs.vllm.ai/en/latest/)**: high-throughput GPU serving — PagedAttention KV-cache management, continuous batching, prefix caching, OpenAI-compatible server; quantization support includes FP8, INT8/INT4, GPTQ, AWQ, MXFP4, and even GGUF; runs on NVIDIA/AMD GPUs, CPUs, TPUs. **Use when:** you have real GPU(s) and multiple concurrent users/agents hammering the API — this is the "small production" tier.
- **[LM Studio](https://lmstudio.ai/docs/app)**: desktop app (macOS Apple Silicon, Windows, Linux) running GGUF via llama.cpp and MLX on Apple Silicon; OpenAI-compatible local server; now ships a headless variant (`llmster`) and `lms` CLI. **Use when:** the "server" is also someone's desktop, or on a Mac where MLX is faster; for a pure headless Linux box, prefer Ollama/llama.cpp.
- Also notable: **SGLang** (vLLM-class serving, listed as a supported framework on Qwen model cards), **HuggingFace TGI**, and **text-generation-webui** (multi-backend web UI; hobbyist tinkering rather than serving).

**Conclusion:** headless server + several clients → Ollama (easy) or `llama-server` (lean); GPU + genuine concurrency → vLLM; desktop/Mac → LM Studio.

## 3. Quantization basics

- **GGUF** is the single-file model format for llama.cpp/GGML executors: typed key-value metadata + tensors in one mmap-able file, successor to GGML/GGMF/GGJT, currently spec v3. [Spec](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md). One file = model + tokenizer + hyperparameters, which is why distribution is so simple.
- **Quant levels** (llama.cpp [quantize tool](https://github.com/ggml-org/llama.cpp/tree/master/tools/quantize), figures for Llama-3.1-8B):
  - **Q4_K_M** — 4.89 bits/weight, 4.58 GiB for 8B; the common default, ~7× smaller than F16 with modest quality loss.
  - **Q5_K_M** (5.70 bpw, 5.33 GiB) and **Q6_K** (6.56 bpw, 6.14 GiB) — closer to full quality, ~15–35% bigger.
  - **Q8_0** (8.5 bpw, 7.95 GiB) — near-lossless; use when RAM is plentiful.
  - **Q3_K_M / IQ3 / IQ2 / IQ1** (≤4 bpw) — degrade noticeably; for squeezing a model class into a too-small GPU. Rule of thumb: a bigger model at Q4 usually beats a smaller model at Q8.
- **Other formats:** **GPTQ** and **AWQ** are GPU-oriented 4-bit weight quantizations used by vLLM/SGLang; **FP8** (and MXFP4/NVFP4) are hardware-native low-precision formats on recent GPUs — all listed in [vLLM's supported quantizations](https://docs.vllm.ai/en/latest/). Publishers increasingly ship low-precision natively: gpt-oss ships MoE weights in MXFP4, GLM ships FP8 repos.
- **When each matters:** CPU / Apple Silicon / mixed CPU+GPU → GGUF via llama.cpp/Ollama/LM Studio. Pure-GPU multi-user serving → AWQ/GPTQ/FP8 checkpoints via vLLM.

## 4. Hardware sizing for a homelab server

Anchor numbers from primary sources: 8B at Q4_K_M = **4.58 GiB** file ([llama.cpp](https://github.com/ggml-org/llama.cpp/tree/master/tools/quantize)); gpt-oss-20b runs **within 16 GB**, gpt-oss-120b on one **80 GB** GPU ([model card](https://huggingface.co/openai/gpt-oss-120b)); unquantized Gemma 4 31B fits one 80 GB H100 ([Google](https://blog.google/innovation-and-ai/technology/developers-tools/gemma-4/)). The table below extrapolates from the ~0.6 bytes/param Q4_K_M ratio plus KV-cache/runtime overhead — *derived estimates, not vendor claims*:

| Model class (dense) | Q4_K_M file | Comfortable VRAM (or RAM for CPU) |
|---|---|---|
| 7–8B | ~4.5–5 GB | 8 GB |
| 13–14B | ~8–9 GB | 12 GB |
| 30–33B | ~18–20 GB | 24 GB |
| 70B | ~40–43 GB | 48 GB (2×24 GB) |

- **MoE changes the math:** memory scales with *total* params, speed with *active* params. Qwen3.6-35B-A3B needs ~20 GB at Q4 but generates like a 3B model — MoE models are unusually tolerant of partial CPU offload, making them the sweet spot for modest GPUs.
- **VRAM vs system RAM:** llama.cpp can split layers between GPU and CPU; anything that spills to system RAM runs at RAM bandwidth, so expect a hard speed cliff when a model doesn't fully fit VRAM. Long contexts add KV-cache on top of weights — leave headroom beyond the file size.
- **Popular homelab picks (observation from hardware support lists, conclusion mine):**
  - **Used RTX 3090 (24 GB)** — still the price/VRAM benchmark; runs 30B-class dense at Q4 or MoE like Qwen3.6-35B/gpt-oss-20b with room; two of them reach 70B Q4. Works with every stack including vLLM.
  - **Apple Silicon as server** — unified memory means a 64–128 GB Mac holds models no consumer GPU can; llama.cpp Metal and LM Studio MLX are first-class ([llama.cpp](https://github.com/ggml-org/llama.cpp), [LM Studio](https://lmstudio.ai/docs/app)). Lower token throughput than a dGPU, excellent perf/watt for an always-on box.
  - **CPU-only** — feasible for ≤8B dense or small-active MoE at Q4 (llama.cpp is heavily SIMD-optimized); interactive but slow, fine for background/batch jobs, frustrating for chat above ~14B dense.

## 5. Local vs cloud APIs (brief)

- **Cost:** local wins only if the hardware already exists or utilization is high; frontier-model API prices (especially Chinese-lab models) have fallen far enough (2026) that light usage is cheaper via API. Electricity for an always-on GPU box is the hidden line item.
- **Privacy/control:** the decisive local advantage — data never leaves the LAN, no retention policy to trust, no model deprecation on someone else's schedule. (Some cheap APIs explicitly retain data for training.)
- **Capability gap:** narrowed but real. Open weights at the frontier exist (DeepSeek V4, GLM-5.2 — MIT), but what fits in 24 GB VRAM is roughly "very good mid-tier, 1–2 years behind frontier." For summarization, RAG, coding assist, home automation: locally sufficient. For hardest reasoning/agent tasks: cloud still wins.
- **Maintenance:** local means driver/CUDA upkeep, runtime updates (llama.cpp ships multiple releases per day), model curation, and doing your own eval. Budget real time for it; Ollama minimizes but doesn't eliminate this.
- **Practical pattern:** OpenAI-compatible local endpoint as default, cloud API as escalation path — most clients let you swap `base_url`, so the two coexist trivially.

---

## Claims NOT verified from primary sources (flagged)

- Qwen3.5 exact lineup and Feb 2026 date; Qwen3.7 being API-only (secondary reports only).
- DeepSeek V4-Pro active-parameter count (~49B per secondary sources; only the 1.6T total is on HF).
- Phi-4-reasoning-vision-15B (Mar 2026) and "Phi-5" (secondary sources only).
- Kimi K2.6 exact release date (model-card fetch ambiguous; 2026 per HF activity).
- Mistral's "new open-weight family in July 2026 early access" (press only, nothing released).
