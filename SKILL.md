---
name: vlmrun-cli-skill
description: "Use the VLM Run CLI (`vlmrun`) to interact with Orion visual AI agent. Process images, videos, and documents with natural language. Triggers: image understanding/generation, object detection, OCR, video summarization, document extraction, image generation, visual AI chat, 'generate an image/video', 'analyze this image/video', 'extract text from', 'summarize this video', 'process this PDF'."
---

# VLM Run CLI

Chat with VLM Run's Orion visual AI agent via CLI.

## Setup

```bash
uv pip install "vlmrun[cli]"

# Create .env file with your API key
cat <<EOF > .env
VLMRUN_API_KEY="your-api-key"
VLMRUN_BASE_URL="https://agent.vlm.run/v1"
EOF
```

## Command

```bash
uv run vlmrun chat "prompt" -i input.jpg [options]
```

## Options

| Flag | Description |
|------|-------------|
| `-p, --prompt` | Prompt text, file path, or `stdin` |
| `-i, --input` | Input file(s) - images, videos, docs (repeatable) |
| `-o, --output` | Artifact directory (default: `~/.vlmrun/cache/artifacts/`) |
| `-m, --model` | `vlmrun-orion-1:fast`, `vlmrun-orion-1:auto` (default), `vlmrun-orion-1:pro` |
| `-j, --json` | Raw JSON output |
| `-ns, --no-stream` | Disable streaming |
| `-nd, --no-download` | Skip artifact download |

## Examples

### Images
```bash
uv run vlmrun chat "Describe what you see in this image in detail" -i photo.jpg
uv run vlmrun chat "Detect and list all objects visible in this scene" -i scene.jpg
uv run vlmrun chat "Extract all text and numbers from this document image" -i document.png
uv run vlmrun chat "Compare these two images and describe the differences" -i before.jpg -i after.jpg
```

### Image Generation
```bash
uv run vlmrun chat "Generate a photorealistic image of a cozy cabin in a snowy forest at sunset" -o ./generated
uv run vlmrun chat "Remove the background from this product image and make it transparent" -i product.jpg -o ./output
```

### Video
```bash
uv run vlmrun chat "Summarize the key points discussed in this meeting video" -i meeting.mp4
uv run vlmrun chat "Find the top 3 highlight moments and create short clips from them" -i sports.mp4
uv run vlmrun chat "Transcribe this lecture with timestamps for each section" -i lecture.mp4 --json
```

### Video Generation
```bash
uv run vlmrun chat "Generate a 5-second video of ocean waves crashing on a rocky beach at golden hour" -o ./videos
uv run vlmrun chat "Create a smooth slow-motion video from this image" -i ocean.jpg -o ./output
```

### Documents
```bash
uv run vlmrun chat "Extract the vendor name, line items, and total amount" -i invoice.pdf --json
uv run vlmrun chat "Summarize the key terms and obligations in this contract" -i contract.pdf
```

### Prompt Sources
```bash
# Direct prompt
uv run vlmrun chat "What objects and people are visible in this image?" -i photo.jpg

# Prompt from file
uv run vlmrun chat -p long_prompt.txt -i photo.jpg

# Prompt from stdin
echo "Describe this image in detail" | uv run vlmrun chat - -i photo.jpg
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `VLMRUN_API_KEY` | API key (required) |
| `VLMRUN_BASE_URL` | Base URL (default: `https://agent.vlm.run/v1`) |
| `VLMRUN_CACHE_DIR` | Cache directory (default: `~/.vlmrun/cache/artifacts/`) |

## Notes

- Use `-o ./directory` to save generated artifacts (images, videos) relative to your current working directory
- Without `-o`, artifacts save to `~/.vlmrun/cache/artifacts/<session_id>/`
- Multiple input files upload concurrently
