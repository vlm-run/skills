# Supported Models

Default vLLM serving arguments for OCR/VLM models. Use `--serve-args` to pass these when running benchmarks:

```bash
uvx vlmbench run --serve --backend vllm --model <model-id> --serve-args '<args>' -i <input>
```

| Model | Params | vLLM `--serve-args` | Notes |
|---|---|---|---|
| `lightonai/LightOnOCR-2-1B` | 1B | `--limit-mm-per-prompt '{"image": 1}' --mm-processor-cache-gb 0 --no-enable-prefix-caching` | |
| `zai-org/GLM-OCR` | 0.9B | `--allowed-local-media-path /` | **Profile: `glm-ocr`** — requires transformers >= 5.0.0, vLLM nightly, MTP speculative decoding. Use `--profile glm-ocr` or `PROFILE=glm-ocr`. |
| `rednote-hilab/dots.ocr` | 3B | `--trust-remote-code --gpu-memory-utilization 0.95` | |
| `allenai/olmOCR-2-7B-1025-FP8` | 8B (FP8) | `--max-model-len 16384` | Based on Qwen2.5-VL-7B |
| `Qwen/Qwen3-VL-8B-Instruct` | 9B | `--mm-encoder-tp-mode "data"` | |
| `Qwen/Qwen3-VL-8B-Instruct-FP8` | 9B (FP8) | `--mm-encoder-tp-mode "data"` | |
| `deepseek-ai/DeepSeek-OCR-2` | 3B | N/A | Not supported in upstream vLLM; requires [custom wheel](https://github.com/deepseek-ai/DeepSeek-OCR-2) |

## Model Profiles

Some models need custom environments (non-standard vLLM images, extra pip installs, special serve args). These are defined as profile directories in `vlmbench/profiles/<name>/` containing `config.toml` + `setup.sh`, shipped via PyPI.

```bash
# List available profiles
uvx vlmbench profiles

# Local: build once, serve, benchmark
make build PROFILE=glm-ocr
make serve PROFILE=glm-ocr
make benchmark PROFILE=glm-ocr BENCHMARK_ARGS="--no-serve --base-url http://localhost:8000/v1"

# HF Jobs (setup.sh runs at container start)
make hf-benchmark PROFILE=glm-ocr FLAVOR=a100-large
```

Available profiles:

| Profile | Model | Task | Custom Image | Custom Setup |
|---|---|---|---|---|
| `glm-ocr` | `zai-org/GLM-OCR` | completion | `vllm/vllm-openai:nightly` | transformers from source |
| `qwen3-vl-2b-embed` | `Qwen/Qwen3-VL-Embedding-2B` | embedding | — | — |
| `dse-qwen2-vl` | `MrLight/dse-qwen2-2b-mrl-v1` | embedding | — | — |
| `qwen3-vl-2b-reranker` | `Qwen/Qwen3-VL-Reranker-2B` | score | — | — |
