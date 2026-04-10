---
name: vlmrun-cli-skill
description: "Use the VLM Run CLI (`vlmrun`) to interact with Orion visual AI agent. Process images, videos, and documents with natural language. Triggers: image understanding/generation, object detection, OCR, video summarization, document extraction, image generation, visual AI chat, 'generate an image/video', 'analyze this image/video', 'extract text from', 'summarize this video', 'process this PDF'."
---

# VLM Run CLI

Chat with VLM Run's Orion visual AI agent via CLI.

## Setup
```bash
uv venv && source .venv/bin/activate
uv pip install "vlmrun[cli]"
```

## Configuration

Configure your API key and base URL using the CLI (get your key from [app.vlm.run](https://app.vlm.run)):

```bash
vlmrun config init
vlmrun config set --api-key <your-api-key>
vlmrun config show
```

| Setting | Type | Description |
|---------|------|-------------|
| `api_key` | Required | Your VLM Run API key (required) |
| `base_url` | Optional | Base URL (default: `https://api.vlm.run/v1`) |
| `cache_dir` | Optional | Cache directory (default: `~/.vlmrun/cache/artifacts/`) |

## Command

```bash
vlmrun chat "<prompt>" -i input.jpg [options]
```

## Options

| Flag | Description |
|------|-------------|
| `-p, --prompt` | Prompt text, file path, or `stdin` |
| `-i, --input` | Input file(s) - images, videos, docs (repeatable) |
| `-o, --output` | Artifact directory (default: `~/.vlmrun/cache/artifacts/`) |
| `-m, --model` | `vlmrun-orion-1:fast`, `vlmrun-orion-1:auto` (default), `vlmrun-orion-1:pro` |
| `-k, --skill` | Path to a local skill directory (inline). Repeatable. Cannot be used with `--skill-id`. |
| `--skill-id` | Server-side skill as `<name>:<version>` (e.g. `my-skill:latest`). Repeatable. Cannot be used with `--skill`. |
| `-t, --toolset` | Tool category to enable (repeatable): `core`, `image`, `image-gen`, `world-gen`, `viz`, `document`, `video`, `web` |
| `-s, --session-id` | Session UUID to continue a previous session |
| `-f, --format` | Output format (`json` for JSON output) |
| `-ns, --no-stream` | Disable streaming |
| `-nd, --no-download` | Skip artifact download |

## Examples

### Images
```bash
vlmrun chat "Describe what you see in this image in detail" -i photo.jpg
vlmrun chat "Detect and list all objects visible in this scene" -i scene.jpg
vlmrun chat "Extract all text and numbers from this document image" -i document.png
vlmrun chat "Compare these two images and describe the differences" -i before.jpg -i after.jpg
```

### Image Generation
```bash
vlmrun chat "Generate a photorealistic image of a cozy cabin in a snowy forest at sunset" -o ./generated
vlmrun chat "Remove the background from this product image and make it transparent" -i product.jpg -o ./output
```

### Video
```bash
vlmrun chat "Summarize the key points discussed in this meeting video" -i meeting.mp4
vlmrun chat "Find the top 3 highlight moments and create short clips from them" -i sports.mp4
vlmrun chat "Transcribe this lecture with timestamps for each section" -i lecture.mp4 --json
```

### Video Generation
```bash
vlmrun chat "Generate a 5-second video of ocean waves crashing on a rocky beach at golden hour" -o ./videos
vlmrun chat "Create a smooth slow-motion video from this image" -i ocean.jpg -o ./output
```

### Documents
```bash
vlmrun chat "Extract the vendor name, line items, and total amount" -i invoice.pdf --json
vlmrun chat "Summarize the key terms and obligations in this contract" -i contract.pdf
```

### Prompt Sources
```bash
# Direct prompt
vlmrun chat "What objects and people are visible in this image?" -i photo.jpg

# Prompt from file
vlmrun chat -p long_prompt.txt -i photo.jpg

# Prompt from stdin
echo "Describe this image in detail" | vlmrun chat - -i photo.jpg
```

### Continuing a previous session
If you want to keep the past conversation and generated artifacts in context, you can use the `-s` flag to continue a previous session using the session ID generated when you started the session.

```bash
# Start a new session of an image generation task where a new character is generated
vlmrun chat "Create an iconic scene of a ninja in a forest, practicing his skills with a katana?" -i photo.jpg

# Use the previous chat session in context to retain the same character and scene context (where the session ID is <session_id>)
vlmrun chat "Create a new scene with the same character meditating under a tree" -i photo.jpg -s <session_id>
```

### Skipping artifact download
If you want to skip the artifact download, you can use the `-nd` flag.
```bash
vlmrun chat "What objects and people are visible in this image?" -i photo.jpg -nd
```

### Inline Skills

Inline skills let you attach a local skill directory directly to a `chat` request using the `--skill` (`-k`) flag. The directory is bundled and sent with the request — no server-side upload needed. Each skill directory must contain a `SKILL.md` file (with optional YAML frontmatter for name/description) and may include additional files (schemas, scripts, assets).

```bash
# Use a local skill to process an image
vlmrun chat "Extract the key fields from this receipt" -i receipt.jpg --skill ./receipt-extraction-skill

# Use a local skill to process a document
vlmrun chat "Parse this invoice" -i invoice.pdf --skill ./invoice-skill --format json

# Combine an inline skill with a specific model
vlmrun chat "Analyze this chart" -i chart.png --skill ./chart-analysis-skill -m vlmrun-orion-1:pro

# Use multiple inline skills in a single request
vlmrun chat "Summarize and extract tables" -i report.pdf --skill ./summarize-skill --skill ./table-extraction-skill

# Let the skill's built-in prompt drive the request (empty prompt)
vlmrun chat "" -i form.pdf --skill ./form-extraction-skill

# Inline skill with a toolset for visualization output
vlmrun chat "Detect objects and visualize" -i photo.jpg --skill ./detection-skill -t viz
```

**Skill directory structure:**
```
my-skill/
├── SKILL.md        # Required — instructions + optional YAML frontmatter
├── schema.json     # Optional — output JSON schema
└── assets/         # Optional — additional resources
    └── examples.md
```

**Example SKILL.md frontmatter:**
```markdown
---
name: receipt-extraction
description: Extract vendor, line items, and totals from receipt images
---

# Receipt Extraction

Extract structured data from receipt images...
```

> **Tip:** Use `--skill` for rapid local development and iteration. Once your skill is ready, upload it with `vlmrun skills upload ./my-skill` so it can be referenced by name using `--skill-id`.

### Server-side Skills

Reference previously uploaded skills by name and version using the `--skill-id` flag.

```bash
# Use a server-side skill by name (defaults to latest version)
vlmrun chat "Extract the key fields" -i receipt.jpg --skill-id receipt-extraction:latest

# Pin a specific skill version
vlmrun chat "Parse this document" -i doc.pdf --skill-id invoice-parser:20260312-abc

# Combine a server-side skill with JSON output
vlmrun chat "Analyze this form" -i form.pdf --skill-id form-analysis:latest --format json
```

## Skills Management

Manage reusable skills via `vlmrun skills`.

```bash
# List / inspect
vlmrun skills list
vlmrun skills list --grouped
vlmrun skills get my-skill
vlmrun skills get my-skill -V 20260312-abc

# Upload a skill folder (name/description from SKILL.md frontmatter)
vlmrun skills upload ./my-skill

# Download a skill
vlmrun skills download my-skill
vlmrun skills download my-skill -o ./local-dir

# Create from prompt or session
vlmrun skills create --prompt "Extract invoice fields"
vlmrun skills create --session-id <session-uuid>
```

## Notes

- Use `-o ./<directory>` to save generated artifacts (images, videos) relative to your current working directory
- Without `-o`, artifacts save to `~/.vlmrun/cache/artifacts/<session_id>/`
- Multiple input files upload concurrently
