# Benchmarking vLLM Deployments with GuideLLM: A Complete Guide to Inference Performance

Self-hosting LLMs on your own hardware gives you control over cost, privacy, and latency — but it also puts the burden of performance validation squarely on you. Without objective measurements, comparing two models or two vLLM configurations is guesswork. This guide walks through using **GuideLLM** as a systematic benchmarking framework: what metrics to collect and why, how to run a comprehensive sweep, how to generate realistic workloads from HuggingFace datasets, how to capture real AI-agent traffic as test input, and finally how to turn the resulting JSON into useful visualizations.

---

## 1. Why Benchmark vLLM Deployments?

Deploying an LLM to production means balancing three forces at once: throughput (how much work per second), latency (how quickly each user gets a response), and hardware cost. A model that looks acceptable in a one-user demo may collapse under concurrent load. A quantized variant that appears slower on a single request may actually serve more users simultaneously because it fits more into GPU memory.

Benchmarking answers questions like:

- Should we deploy Llama 3.1 8B or 70B on this H100 for our chatbot SLO of < 500 ms TTFT at peak?
- Does switching from FP16 to FP8 quantization meaningfully degrade latency, or does the throughput gain make up for it?
- How many replicas do we need to stay within our P99 latency budget at projected traffic?
- Does our current hardware saturate at 20 RPS, or can it handle 50?

GuideLLM is built specifically to answer these questions for OpenAI-compatible inference servers, and vLLM is its primary target.

---

## 2. Understanding the Key Metrics

GuideLLM measures LLM inference performance through a well-defined set of metrics. Understanding what each one measures — and which matters for your use case — is a prerequisite for making good decisions from benchmark results.

### 2.1 Time to First Token (TTFT)

**Definition:** The elapsed time from when a request is submitted to when the first output token is received.

**What drives it:** TTFT is dominated by the *prefill* phase — the GPU's work to process the entire input prompt and build the KV cache before generation starts. Longer prompts mean longer prefill. Under heavy concurrency, queuing time adds to TTFT even when the model itself is fast.

**Why it matters:** For interactive applications (chatbots, code completion, agentic loops), TTFT determines perceived responsiveness. A user or an agent waiting on a tool call feels every millisecond of TTFT directly. The typical threshold for a chatbot to feel "instant" is under 300–500 ms. Coding assistants require under 100 ms to feel seamless.

**How to read GuideLLM output:** GuideLLM reports TTFT as mean, median, P95, and P99. The P99 is the one to watch for SLO compliance — it captures the worst 1% of requests, which often corresponds to peak-load behavior.

```
TTFT (ms):  mean=142  p50=118  p95=381  p99=612
```

A large gap between P50 and P99 indicates that some requests are queuing or hitting a slow path. In this case investigate concurrency and scheduling.

### 2.2 Inter-Token Latency (ITL) / Time Per Output Token (TPOT)

**Definition:** The average time between consecutive output tokens after the first one. For a single request, ITL (per-gap measurement) and TPOT (average over all gaps) are equivalent. Across many requests, they differ in how they aggregate.

**What drives it:** ITL is dominated by the *decode* phase, where the model autoregressively generates one token at a time. GPU memory bandwidth, batch size, and speculative decoding all affect ITL. Unlike TTFT, ITL does not scale with prompt length — it scales with how many requests are being decoded simultaneously.

**Why it matters:** In a streaming interface, ITL determines how smoothly text appears on screen. At 30 ms ITL, a user sees roughly 33 tokens per second — comfortable reading speed. At 150 ms ITL, text visibly trickles. For agentic workloads that don't stream output but wait for complete responses, ITL × output_tokens adds directly to end-to-end latency.

**Formula:**
```
TPOT = (E2E_latency − TTFT) / (output_tokens − 1)
```

### 2.3 End-to-End (E2E) Latency

**Definition:** Total wall-clock time from request submission to receiving the final output token.

**Why it matters:** For batch-style workloads, background agents, or synchronous API calls (where your code blocks on the full response), E2E latency is the metric your users actually experience. It is the sum of TTFT and all subsequent decode steps:

```
E2E_latency = TTFT + TPOT × (output_tokens − 1)
```

GuideLLM reports E2E latency as a histogram with percentiles.

### 2.4 Requests Per Second (RPS) / Throughput

**Definition:** The number of requests the server completes per second at a given concurrency or request rate.

**Why it matters:** RPS determines capacity: how many simultaneous users can be served within acceptable latency. It is the primary metric for infrastructure cost calculations — higher RPS on the same hardware means lower cost per request.

**The saturation curve:** As request rate increases, RPS initially grows linearly. Eventually the server reaches saturation — its maximum RPS — and any additional load only increases queue depth and latency, with no further throughput gain. GuideLLM's sweep profile is designed specifically to map this curve.

### 2.5 Token Throughput

**Definition:** Output tokens generated per second, aggregated across all concurrent requests.

**Why it matters:** Token throughput is the GPU-level measure of efficiency. It is distinct from per-user RPS because a batch of ten 100-token requests and one 1000-token request both generate 1000 tokens, but serve users very differently. For infrastructure planning, token throughput tells you the raw generation capacity of your deployment.

### 2.6 Concurrency

**Definition:** The number of requests the server is actively processing at any moment.

**Why it matters:** Concurrency is the key lever that trades off latency and throughput. At low concurrency, each request gets maximum GPU attention (low latency, low throughput). At high concurrency, batching improves throughput but increases per-request wait time. vLLM's continuous batching means concurrency determines batch size dynamically, which is why understanding concurrency behavior matters for tuning `--max-num-seqs` and similar parameters.

---

## 3. Setting Up vLLM and GuideLLM

### 3.1 Start vLLM

GuideLLM requires an OpenAI-compatible server. vLLM provides this out of the box:

```bash
# Start vLLM with a model of your choice
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 \
  --dtype auto \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90
```

### 3.2 Install GuideLLM

```bash
pip install guidellm
```

Verify it can reach your server:

```bash
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=emulated,prompt_tokens=128,generated_tokens=128" \
  --profile kind=synchronous \
  --max-seconds 10
```

---

## 4. Running a Sweep Benchmark

The **sweep profile** is the most informative starting point for understanding a deployment. Rather than testing at a single fixed rate, a sweep runs a series of benchmarks that progressively stress the system:

1. **Synchronous** — one request at a time, representing a single-user baseline.
2. **Throughput** — all requests sent in parallel, identifying the server's maximum RPS ceiling.
3. **Intermediate rates** — a series of constant-rate benchmarks between the two extremes, capturing the full latency-throughput tradeoff curve.

This produces a complete picture of how your deployment behaves from idle to saturation in a single run.

### 4.1 Basic Sweep with Synthetic Data

```bash
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=emulated,prompt_tokens=512,generated_tokens=256" \
  --profile kind=sweep \
  --max-seconds 60 \
  --output-dir ./results/synthetic-sweep
```

The `emulated` data source generates synthetic prompts with the specified token counts, giving you reproducible results that are not confounded by dataset variability.

**Common data profiles to test:**

| Workload Type | Prompt Tokens | Output Tokens | Rationale |
|---|---|---|---|
| Chatbot | 512 | 256 | Short context, medium response |
| RAG | 2048 | 256 | Long context, concise answer |
| Code generation | 256 | 512 | Short prompt, long output |
| Summarization | 4096 | 128 | Very long input, short output |
| Document Q&A | 8192 | 512 | Max-context workloads |

### 4.2 Controlling Sweep Granularity

By default, GuideLLM interpolates intermediate rates automatically. You can control the duration and depth of each stage:

```bash
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=emulated,prompt_tokens=512,generated_tokens=256" \
  --profile kind=sweep \
  --max-seconds 120 \           # 2 minutes per stage
  --max-requests 500 \          # or cap by request count
  --output-dir ./results/deep-sweep
```

The `--max-seconds` and `--max-requests` flags apply **per strategy** within the sweep, so a `--max-seconds 60` sweep will run the synchronous stage for 60 seconds, the throughput stage for 60 seconds, and each intermediate rate for 60 seconds.

### 4.3 Reading the Sweep Output

After the run completes, GuideLLM prints a three-section summary:

**Benchmarks Metadata** — the run configuration: server URL, model, data profile, sweep parameters.

**Benchmarks Info** — per-stage overview: how many requests were sent, how many succeeded, total tokens processed.

**Benchmarks Stats** — the metrics you care about, per stage:

```
Stage: constant @ 5.2 req/s
  Concurrency:      8.3
  RPS:              5.18
  E2E Latency:      mean=1621ms  p95=2814ms  p99=3102ms
  TTFT:             mean=189ms   p95=412ms   p99=538ms
  ITL:              mean=42ms    p95=71ms    p99=89ms
  Token Throughput: 1324 tok/s
```

The full data, including every individual request, is written to `benchmarks.json` in your output directory.

---

## 5. Using HuggingFace Datasets for Realistic Workloads

Synthetic token counts tell you how the system performs under controlled conditions, but real production traffic has natural variation in prompt length, vocabulary, and structure that can meaningfully affect caching behavior, tokenization overhead, and KV cache utilization. HuggingFace datasets let you benchmark on representative content.

### 5.1 Direct HuggingFace Dataset Loading

GuideLLM's `kind=huggingface` data source loads any dataset from the Hub:

```bash
# General Q&A workload
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=huggingface,source=garage-bAInd/Open-Platypus" \
  --profile kind=sweep \
  --max-seconds 60 \
  --output-dir ./results/open-platypus-sweep

# Code-heavy workload
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=huggingface,source=bigcode/the-stack-smol,subset=python" \
  --profile kind=sweep \
  --max-seconds 60 \
  --output-dir ./results/code-sweep

# Instruction-following workload
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=huggingface,source=HuggingFaceH4/ultrachat_200k,split=train_sft" \
  --profile kind=sweep \
  --max-seconds 60 \
  --output-dir ./results/chat-sweep
```

### 5.2 Specifying Column Mappings

Datasets vary in their schema. Use `--data-args` to tell GuideLLM which column contains the prompt text:

```bash
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=huggingface,source=allenai/dolma,subset=v1_7" \
  --data-args '{"prompt_column": "text"}' \
  --profile kind=sweep \
  --max-seconds 60
```

### 5.3 Sampling and Shuffling

To ensure you don't always use the same examples in the same order:

```bash
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=huggingface,source=garage-bAInd/Open-Platypus" \
  --data-sampler random \
  --profile kind=sweep \
  --max-seconds 60
```

---

## 6. Recording AI Agent Sessions as Benchmark Input

Synthetic datasets and public HuggingFace corpora give you good coverage of the token distribution space, but they may miss the specific patterns of your actual users. One powerful technique is to **capture real AI agent sessions** and replay them as benchmark inputs. This produces test data that exactly mirrors the prompt shapes, lengths, and multi-turn structures your deployment will see in production.

### 6.1 Claude Code: Capturing Session Transcripts

Claude Code stores complete session transcripts locally in `~/.claude/projects/`. Each session is persisted as a JSONL file containing every message, tool call, and response.

**Exporting a session:**

Within a Claude Code session, use the `/export` command to dump the full conversation to a markdown file:

```
/export
```

This writes a timestamped file to your working directory containing every prompt, response, and tool invocation with full content.

To access raw JSONL transcripts programmatically:

```bash
# List all sessions
ls ~/.claude/projects/

# Find recent sessions
find ~/.claude/projects/ -name "*.jsonl" -ls

# Extract user messages (the prompts your LLM received)
cat ~/.claude/projects/<session-id>.jsonl \
  | jq -r 'select(.role == "user") | .content' \
  > claude-code-prompts.txt
```

**Formatting for GuideLLM:**

Convert extracted prompts to the JSON format GuideLLM's `kind=json_file` source expects:

```python
import json

with open("claude-code-prompts.txt") as f:
    raw_prompts = [line.strip() for line in f if line.strip()]

dataset = [{"prompt": p} for p in raw_prompts if len(p) > 20]

with open("agent-session-dataset.json", "w") as f:
    json.dump(dataset, f, indent=2)
```

### 6.2 Building and Running an Agent-Realistic Benchmark

Once you have your agent session dataset, run a sweep that reflects how these agents actually use the model:

```bash
guidellm benchmark \
  --target "http://localhost:8000" \
  --data "kind=json_file,path=./agent-session-dataset.json" \
  --data-sampler random \
  --profile kind=sweep \
  --max-seconds 90 \
  --output-dir ./results/agent-realistic
```

This benchmark now uses real prompt lengths and real vocabulary from production sessions, making it the most representative test of deployment readiness.

---

## 7. Interpreting Results and Making Decisions

A complete sweep benchmark gives you enough data to answer most deployment questions directly.

**Finding the saturation point:** Plot TTFT vs. RPS. The saturation point is where the TTFT curve inflects sharply upward — additional load produces no more throughput but significantly more latency. This is your maximum sustainable RPS for a given TTFT budget.

**Setting SLOs:** Identify the RPS at which your target percentile (typically P99 TTFT and P95 E2E latency) stays within acceptable bounds. That is your capacity ceiling for this deployment.

**Comparing configurations:** Overlay latency-throughput curves from two benchmarks. The better configuration is the one that achieves the same RPS at lower latency, or the same latency at higher RPS. The comparison is often non-obvious: FP8 quantization may improve throughput at high load while adding a small fixed latency penalty at low load, which may or may not matter for your SLO.

**Capacity planning:** Divide your projected peak RPS by the measured sustainable RPS per server to get the number of replicas needed. Add a safety margin for the SLO percentile you want to protect.

---

## Summary

GuideLLM gives you a rigorous, repeatable way to measure the performance of any vLLM deployment. The sweep profile maps the full latency-throughput curve in a single run. HuggingFace datasets let you benchmark against realistic content. Capturing AI agent sessions produces the most production-representative test data possible. And the resulting `benchmarks.json` can be loaded into Python for any visualization or analysis your workflow requires.

The combination of these tools — vLLM for serving, GuideLLM for benchmarking, HuggingFace for data, and your own agent sessions for realism — gives you everything needed to make confident, data-driven decisions about model selection, quantization, hardware sizing, and deployment configuration.

