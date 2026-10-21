# Running vLLM with CPU Backend

GPUs are the default assumption for LLM serving, but not every machine has one, and not every use case needs one. For local prototyping, offline batch jobs, or just poking at a new model release without spinning up cloud infrastructure, vLLM's CPU backend is enough to get an OpenAI-compatible server running on a laptop or a plain Linux box. This guide walks through installing vLLM's CPU backend and serving **Qwen/Qwen3.5-0.8B** — the smallest model in Alibaba's Qwen3.5 family — entirely on CPU.

---

## 1. Why Run vLLM on CPU?

vLLM is best known as a GPU inference engine, but it ships a dedicated CPU backend for x86, ARM, and (experimentally) Apple Silicon. Throughput on CPU is far below what a GPU delivers, but that trade-off is acceptable for a specific set of situations: testing a model before committing GPU budget to it, running inference on infrastructure that has no accelerator available, or keeping a small assistant model warm on a machine that is otherwise idle. The same `vllm serve` command and OpenAI-compatible API you'd use on a GPU box work unchanged on CPU — only the installation and a handful of tuning environment variables differ.

## 2. Picking a Model: Qwen3.5-0.8B

Qwen3.5 is Alibaba's latest model family, released in early 2026, spanning dense models from 0.8B to 27B parameters and mixture-of-experts variants up into the hundreds of billions. All of them share a hybrid gated-delta-network architecture and a 262K-token context window. **Qwen/Qwen3.5-0.8B** is the smallest dense member of the family — small enough that CPU inference stays responsive, and still capable enough for real instruction-following and tool-use testing rather than toy demos.

Qwen3.5 models are natively multimodal (the model card lists an image-text-to-text pipeline), but they work perfectly well as text-only chat models — you simply never send an image in the request. The model also supports both "thinking" and "non-thinking" response modes, which vLLM exposes through its `qwen3` reasoning parser.

## 3. Checking Your Hardware

vLLM's CPU backend targets Linux on x86 or ARM, with experimental macOS support. Before installing, confirm what instruction sets your CPU offers:

```bash
lscpu | grep -E "avx2|avx512f"
nproc
```

`avx512f` gives the best performance; `avx2` works but with reduced throughput. If you see neither flag, the backend will still build on most modern x86 CPUs, but expect it to be slow. `nproc` tells you how many cores you have to allocate between inference threads and the rest of the system.

## 4. Installing vLLM's CPU Backend

The simplest path is a pre-built CPU wheel — no compiler toolchain required. `uv` is the fastest way to manage this:

```bash
uv venv --python 3.12 --seed
source .venv/bin/activate

export VLLM_VERSION=$(curl -s https://api.github.com/repos/vllm-project/vllm/releases/latest | jq -r .tag_name | sed 's/^v//')
uv pip install "https://github.com/vllm-project/vllm/releases/download/v${VLLM_VERSION}/vllm-${VLLM_VERSION}+cpu-cp38-abi3-manylinux_2_39_x86_64.whl" --torch-backend cpu
```

For example, for version 0.32.0 this resolves to:

```
https://github.com/vllm-project/vllm/releases/download/v0.32.0/vllm-0.32.0+cpu-cp38-abi3-manylinux_2_39_x86_64.whl
```

If you'd rather pick a version by hand instead of always pulling the latest, browse the [vLLM releases page](https://github.com/vllm-project/vllm/releases) and substitute its tag into the URL pattern above.

The CPU wheel depends on Intel's OpenMP runtime, which needs to be preloaded explicitly:

```bash
IOMP_PATH=$(sudo find / -iname "*libiomp5.so" 2>/dev/null | head -n1)
export LD_PRELOAD="$IOMP_PATH:$LD_PRELOAD"
```

If that `find` comes up empty — common on a bare Linux install or a fresh WSL2 distro, since nothing else on the system pulls in Intel's OpenMP runtime — install it first:

```bash
sudo apt install libomp-dev
```

Installing `libtcmalloc` is also worth doing — vLLM's own docs recommend it for noticeably better memory allocation behavior under load:

```bash
sudo apt-get install -y libtcmalloc-minimal4
TCMALLOC_PATH=$(sudo find / -iname "*libtcmalloc_minimal.so.4" 2>/dev/null | head -n1)
export LD_PRELOAD="$TCMALLOC_PATH:$IOMP_PATH:$LD_PRELOAD"
```

## 5. Tuning Memory and Threads

CPU inference has no separate GPU memory pool — the KV cache lives in system RAM, and inference threads compete with everything else running on the box. Two environment variables control this:

`VLLM_CPU_KVCACHE_SPACE` sets how much RAM (in GiB) is reserved for the KV cache. `VLLM_CPU_OMP_THREADS_BIND` pins OpenMP inference threads to specific CPU cores, and should leave one or two cores free for the server's own request handling. On a 6-core machine, for example:

```bash
export VLLM_CPU_KVCACHE_SPACE=4
export VLLM_CPU_OMP_THREADS_BIND=0-4
```

If you're on a hyperthreaded machine, run `lscpu -e` first and bind only one OpenMP thread per physical core — binding to both logical threads of the same core wastes scheduling overhead rather than adding real parallelism.

## 6. Serving Qwen3.5-0.8B

vLLM's official recipe for this model targets GPU, with a 262,144-token context window — far more KV cache than is practical to keep in RAM on a CPU-only box. For local CPU serving, cap the context length to something your available memory can comfortably hold, and set `bfloat16` as the compute dtype, which vLLM's CPU backend handles natively:

```bash
VLLM_TARGET_DEVICE=cpu TRITON_DEFAULT_BACKEND=cpu vllm serve Qwen/Qwen3.5-0.8B \
  --dtype=bfloat16 \
  --max-model-len 8192 \
  --reasoning-parser qwen3
```

The `--reasoning-parser qwen3` flag tells vLLM how to separate the model's "thinking" tokens from its final answer when the model is prompted to reason before responding. Drop it if you only care about non-thinking responses. `VLLM_TARGET_DEVICE=cpu` and `TRITON_DEFAULT_BACKEND=cpu` are both harmless on a GPU-less box but necessary on a machine that has an NVIDIA GPU installed (see Troubleshooting below). The server starts and exposes the familiar OpenAI-compatible API on `http://localhost:8000`.

## 7. Talking to the Server

Any OpenAI client library works against this endpoint. Using the official `openai` Python package:

```python
from openai import OpenAI

client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")
resp = client.chat.completions.create(
    model="Qwen/Qwen3.5-0.8B",
    messages=[{"role": "user", "content": "Explain what a KV cache is in one sentence."}],
    max_tokens=128,
)
print(resp.choices[0].message.content)
```

Or with plain `curl`, which is handy for quick sanity checks without any client dependencies:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3.5-0.8B",
    "messages": [{"role": "user", "content": "Explain what a KV cache is in one sentence."}],
    "max_tokens": 128
  }'
```

## 8. An Even Simpler Option: Podman

If you'd rather skip the install steps entirely, vLLM publishes pre-built CPU images, and Podman pulls and runs them the same way Docker would, with no daemon required:

```bash
podman pull vllm/vllm-openai-cpu:latest-x86_64

podman run --rm \
  --security-opt seccomp=unconfined \
  --cap-add SYS_NICE \
  --shm-size=4g \
  -p 8000:8000 \
  -e VLLM_CPU_KVCACHE_SPACE=4 \
  -e VLLM_TARGET_DEVICE=cpu \
  -e TRITON_DEFAULT_BACKEND=cpu \
  vllm/vllm-openai-cpu:latest-x86_64 \
  Qwen/Qwen3.5-0.8B \
  --dtype=bfloat16 \
  --max-model-len 8192 \
  --reasoning-parser qwen3
```

`--cap-add SYS_NICE` together with `--security-opt seccomp=unconfined` is the CPU backend's preferred, less-permissive alternative to running the container as `--privileged`. An `arm64` tag of the same image exists for Graviton and other ARM hosts.

## 9. What to Expect

Throughput on CPU is measured in tokens per second from a single machine, not the hundreds-to-thousands per second a GPU deployment reaches. A 0.8B model on a handful of modern CPU cores should feel responsive enough for single-user interactive use — short prompts complete in a second or two — but concurrent requests queue up quickly since there's no equivalent of GPU-style batched parallelism across many cores. Treat this setup as a development, prototyping, or low-traffic tool, not as a production serving path for real request volume.

## 10. Troubleshooting

An `AssertionError: Torch not compiled with CUDA enabled` crash at startup, even with a confirmed CPU-only torch install (`torch.version.cuda` is `None`), means something downstream still picked a CUDA code path despite the CPU build. Two independent things can cause this on a machine that has an NVIDIA GPU installed, even one you have no intention of using:

vLLM's own CUDA platform plugin probes for a GPU directly through `pynvml`, independent of which torch build is installed, and can select the CUDA platform before falling back to CPU. `VLLM_TARGET_DEVICE=cpu` routes around that probe and forces the CPU platform explicitly.

Separately — and this is the more likely culprit if the crash happens during the dummy sampler pass at startup — Triton's own backend auto-selection (`triton.runtime.driver._create_driver()`) prefers any active non-CPU driver it finds and only falls back to the CPU driver when none is active, so it can pick the NVIDIA backend out from under a CPU-only PyTorch build. This is tracked in [vLLM issue #62484](https://github.com/vllm-project/vllm/issues/62484). As of this writing it's open and unconfirmed by maintainers, but the reporter verified a working fix with:

```bash
export TRITON_DEFAULT_BACKEND=cpu
```

Setting both `VLLM_TARGET_DEVICE=cpu` and `TRITON_DEFAULT_BACKEND=cpu` up front, as in the serve commands above, sidesteps both mechanisms at once. If the crash persists, confirm you actually have the CPU wheel installed with `python -c "import vllm; print(vllm.__version__)"` — the version string should contain `cpu` — and cross-check against vLLM's own [CPU installation guide](https://docs.vllm.ai/en/latest/getting_started/installation/cpu/#cpu), since these environment variables and detection details are version-sensitive and can change between releases.

---

## Summary

vLLM's CPU backend turns any modern x86 or ARM Linux box into an OpenAI-compatible inference server, no GPU required. For Qwen3.5-0.8B, that means a pre-built wheel, an `LD_PRELOAD` for OpenMP and TCMalloc, a couple of environment variables to bound KV cache memory and thread placement, and a `vllm serve` command with a CPU-appropriate context length. The result is the same API, the same client code, and the same model family you'd use on a GPU — just scoped to the throughput a CPU can actually deliver.
