---
name: vlmbench
description: >
  VLM benchmark CLI — run, compare, and reproduce VLM inference benchmarks.
  Use when the user wants to: (1) benchmark a VLM model's throughput, latency,
  or VRAM usage, (2) compare results across models or configs, (3) choose or
  configure a backend (vLLM Docker, vLLM native, Ollama, SGLang, cloud API),
  (4) look up model-specific --serve-args, or (5) debug server startup or
  GPU detection issues. Triggers on: vlmbench, VLM benchmarking, OCR model
  evaluation, inference throughput, tokens/sec, TTFT, TPOT, VRAM.
---

# vlmbench

All CLI logic is in `vlmbench/cli.py` (single file). Entry point: `main()`.

## Running benchmarks

Prefer `uvx` — no install needed:

```bash
# macOS (Ollama auto-starts)
uvx vlmbench run -m qwen3-vl:2b -i ./images/

# Linux (vLLM Docker auto-starts with --gpus all)
uvx vlmbench run -m Qwen/Qwen3-VL-8B-Instruct -i ./images/

# Linux (native vLLM)
uvx vlmbench run -m Qwen/Qwen3-VL-8B-Instruct -i ./images/ --backend vllm

# Compare results
uvx vlmbench compare results/*.json
```

`run` is the default subcommand — flags like `--model` implicitly invoke it.

## Backend resolution

| Platform | `auto` resolves to | Model ID style |
|---|---|---|
| macOS | `ollama` | `qwen3-vl:2b` |
| Linux | `vllm-openai:latest` (Docker) | `Qwen/Qwen3-VL-2B-Instruct` |

Docker backends use `--gpus all --ipc=host`. Servers run in tmux sessions (`vlmbench-vllm`, `vlmbench-ollama`, `vlmbench-sglang`).

## Model-specific serve-args

See [MODELS.md](MODELS.md) for the full table of tested models and their required `--serve-args`. Reference this file when the user asks which models are supported or how to serve a specific model.

## Concurrency sweep

Use `--concurrency 4,8,16,32,64` to run at multiple concurrency levels in a single invocation. Produces one JSON per level (tagged `c4`, `c8`, etc.) and a consolidated comparison table. A single value (e.g. `--concurrency 8`) runs at that level only.

## Key metrics

TTFT (ms), TPOT (ms), tok/s, img/s, latency s/img, VRAM peak (MiB), prompt/completion token counts. Results saved as JSON to `./results/{backend}-v{version}-{model}-{gpu}-{tag}.json`.

## GPU detection

`CUDA_VISIBLE_DEVICES` is respected via `_nvidia_gpu_index()` -> `nvidia-smi --id={idx}`. When VRAM or GPU name looks wrong, check this env var first.
