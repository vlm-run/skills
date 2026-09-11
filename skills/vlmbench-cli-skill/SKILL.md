---
name: vlmbench-cli-skill
description: "Use the `vlmbench` CLI to benchmark and compare VLM/LLM inference performance — the quickest way to measure any OpenAI-compatible provider or local server. One `uvx vlmbench` command reports throughput (img/s, tok/s), TTFT, TPOT, ITL, E2EL, latency percentiles and VRAM, then saves reproducible JSON. Triggers: vlmbench, benchmark a VLM, benchmark a provider/endpoint/API, compare providers or models, inference throughput, tokens per second, TTFT, time to first token, TPOT, VRAM usage, OCR model evaluation, vLLM/Ollama/SGLang benchmark, 'how fast is this model', 'which provider is faster', 'measure my endpoint'."
license: Apache-2.0
---

# vlmbench

The fastest way to benchmark a VLM provider: one `uvx` command, nothing to install, reproducible JSON out.

Point it at any OpenAI-compatible endpoint — a hosted provider or a local server — and it measures throughput, TTFT, TPOT and latency percentiles under real concurrency. Works for text-only LLM endpoints too.

## Quickstart — prefer `uvx`

```bash
# Benchmark a provider. Model is auto-detected from GET /v1/models.
uvx vlmbench --base-url https://gateway.vlm.run/v1/openai --api-key $VLMRUN_API_KEY

# Pin the model, use your own images/PDFs
uvx vlmbench -m Qwen/Qwen3-VL-8B-Instruct -i ./images/ \
  --base-url https://api.example.com/v1 --api-key $API_KEY

# Sweep concurrency to find peak throughput
uvx vlmbench -m gpt-4o-mini -d hf://vlm-run/FineVision-vlmbench-mini --max-samples 64 \
  --base-url https://api.openai.com/v1 --api-key $OPENAI_API_KEY \
  --concurrency 4,8,16,32,64
```

`run` is the default subcommand, so any invocation starting with a flag implies it. With no `-i`/`-d`, a
sample image is used — which makes the first command above a complete, zero-setup benchmark.
`pip install vlmbench` works too, but `uvx` needs no environment.

## Comparing providers

Give each run the same input and a `--tag`, then compare them in one table:

```bash
uvx vlmbench -d hf://vlm-run/FineVision-vlmbench-mini --max-samples 64 \
  --base-url https://provider-a.example.com/v1 --api-key $A_KEY --tag provider-a
uvx vlmbench -d hf://vlm-run/FineVision-vlmbench-mini --max-samples 64 \
  --base-url https://provider-b.example.com/v1 --api-key $B_KEY --tag provider-b

uvx vlmbench compare                     # auto-loads ~/.vlmbench/benchmarks/
uvx vlmbench compare results/*.json      # or pass files explicitly
```

`compare` accepts `--sort-by img/s|tok/s`, `--max-rows-per-model N`, and `-c vram,backend,hw,quant`
for extra columns.

## Benchmarking a local model

Pass `--serve` and vlmbench starts the server itself in a tmux session with a GPU monitor pane.
Without `--serve` it reuses an already-running server, or prints the exact command to start one.

```bash
uvx vlmbench -m qwen3-vl:2b -i ./images/ --serve                     # macOS → Ollama
uvx vlmbench -m Qwen/Qwen3-VL-2B-Instruct -i ./images/ --serve       # Linux → vLLM Docker (--gpus all)
uvx vlmbench --profile deepseek-ocr -i ./images/ --serve             # profile bundles model + serve-args
```

`--backend`: `auto` (Ollama on macOS, `vllm-openai:latest` on Linux), `ollama`, `vllm` (native),
`vllm-openai:<tag>`, `sglang:<tag>`. `uvx vlmbench profiles` lists bundled model profiles.
Watch the server live with `tmux attach -t vlmbench-vllm`.

## Flags worth knowing

| Flag | Default | Notes |
|---|---|---|
| `-m` / `--model` | auto-detect | From `GET /v1/models`; required only with `--serve` |
| `-i` / `--input` | sample image | File, directory, or URL — images, PDFs, videos |
| `-d` / `--dataset` | — | `hf://org/name`; add `--dataset-text-col <col>` for text-only LLM runs |
| `--base-url` | auto-detect | Any OpenAI-compatible endpoint |
| `--api-key` | `$OPENAI_API_KEY`, else `no-key` | Sent as `Authorization: Bearer` |
| `--prompt` | `"Extract all text from this document."` | Pass `""` to send a text column as the whole message |
| `--concurrency` | `8` | Single value or sweep, e.g. `4,8,16,32,64` |
| `--runs` / `--warmup` | `3` / `1` | Warmup runs are not recorded |
| `--max-samples` | all | Cap inputs for a quick dry-run |
| `--max-tokens` | `4096` | Completion cap |
| `--task` | `completion` | Or `embedding` / `score` for pooling and reranker models |
| `--tag` | — | Label in the filename and metadata — use it when comparing providers |
| `--upload` | off | Push results to `vlm-run/vlmbench-results` (needs `HF_TOKEN`) |

## Output

Each run prints throughput (img/s and tok/s), TTFT, TPOT, ITL, E2EL, per-worker latency with p50/p95/p99,
prompt/completion token stats, VRAM peak and request reliability.

Results are saved as JSON to `~/.vlmbench/benchmarks/{backend}-v{version}-{model}-{gpu}-{tag}.json`,
one file per concurrency level (sweeps append `c4`, `c8`, … to the tag). Those files are what `compare` reads.

Tested models and their required `--serve-args` are listed in
[MODELS.md](https://github.com/vlm-run/vlmbench/blob/main/.claude/skills/vlmbench/MODELS.md).
