---
name: vlmrun-cli-skill
description: "Use the VLM Run CLI (`vlmrun chat`) to run the Orion visual AI agent on images, videos, documents and audio with natural language. Orion plans and runs tools: image and video generation, detection, editing, OCR and extraction, video summaries, multi-step visual tasks, and stateful sessions with artifacts. Triggers: 'generate an image/video', 'analyze this image/video', 'extract text from', 'summarize this video', 'process this PDF', 'edit/crop/redact this image'. For one-model gateway calls (OCR to markdown, VLM chat, embeddings, transcription, System One decisions), use `vlmrun gw` and the `vlmrun-gw` skill."
---

# VLM Run CLI: Orion agent

`vlmrun chat` sends a prompt and local files to the Orion agent on the platform API (`https://api.vlm.run/v1`).
Orion plans, calls tools, and returns text, JSON and generated files (artifacts).

## Orion or the gateway

| The ask | Use |
| --- | --- |
| Many steps, tool use, generated or edited images and video, a session you continue | `vlmrun chat` (this skill) |
| One model, one call: document to markdown, OCR with boxes, layout | `vlmrun gw chat` |
| A question about an image, PDF or video with a named VLM | `vlmrun gw chat -p` |
| Embeddings | `vlmrun gw embed` |
| Speech to text | `vlmrun gw transcribe` |
| A yes/no, label or score with probabilities, or a pass/fail gate | `vlmrun gw s1` (System One) |

For the gateway, read the `vlmrun-gw` skill (install: `npx skills add vlm-run/skills@vlmrun-gw`, or read
https://vlm.run/gateway/SKILL.md). Gateway calls cost less per call and take URLs as inputs. Agents can also reach the
gateway over MCP at `https://gateway.vlm.run/mcp`.

## Setup

```bash
pip install -U vlmrun        # the CLI is in the base package; there is no [cli] extra
vlmrun --version
```

Check for a saved key before you ask the human for one. Do not print the key.

```bash
curl -sS "https://api.vlm.run/v1/models" \
  -H "Authorization: Bearer $VLMRUN_API_KEY"
```

| Status | Meaning |
| --- | --- |
| 200 | The key works. |
| 401 | `VLMRUN_API_KEY` is empty in this shell. |
| 403 `Invalid API Key` | The saved key is wrong. |

If there is no working key, follow the HTTP signup runbook at https://vlm.run/auth.md. The human must agree to the
terms. The runbook returns a key for a new account, or a new key for an existing account. Save it as
`VLMRUN_API_KEY`. A human can also copy a key from [Studio](https://app.vlm.run).

The CLI reads the key from `--api-key`, then `VLMRUN_API_KEY`, then `~/.vlmrun/config.toml`. With the environment
variable set, no config is needed. To save it to the config file, pass the variable so the secret is not in the
command text:

```bash
vlmrun config init                          # creates ~/.vlmrun/config.toml
vlmrun config set --api-key "$VLMRUN_API_KEY"
vlmrun config show                          # prints the key masked
```

| Setting | Where | Default |
| --- | --- | --- |
| API key | `--api-key`, `VLMRUN_API_KEY`, or `api_key` in the config | none |
| API base URL | `--base-url`, `VLMRUN_BASE_URL`, or `base_url` in the config | `https://api.vlm.run/v1` |
| Cache directory | `VLMRUN_CACHE_DIR` | `~/.vlmrun/cache` |

## Command

```bash
vlmrun chat "<prompt>" -i input.jpg [options]
```

| Flag | Description |
| --- | --- |
| `PROMPT` | Prompt text. `-` reads stdin. |
| `-p, --prompt` | Prompt text, a file path, or `stdin`. |
| `-i, --input` | Local input file, repeatable. Not a URL: download it first. |
| `-m, --model` | `vlmrun-orion-1:fast`, `vlmrun-orion-1:auto` (default), `vlmrun-orion-1:pro` |
| `-t, --toolset` | Limit tools to these categories, repeatable: `core`, `image`, `image-gen`, `world-gen`, `viz`, `document`, `video`, `web` |
| `-k, --skill` | Local skill directory sent inline, repeatable. Not with `--skill-id`. |
| `--skill-id` | Server-side skill as `<name>:<version>`, e.g. `my-skill:latest`. Repeatable. Not with `--skill`. |
| `-s, --session-id` | Continue a session by its UUID. |
| `-o, --output` | Artifact directory. Default `~/.vlmrun/cache/artifacts/<session_id>/`. |
| `-f, --format` | `json` prints JSON instead of formatted text. |
| `-ns, --no-stream` | Wait for the full reply. |
| `-nd, --no-download` | Do not download artifacts. |
| `--timeout` | Request timeout in seconds. |
| `--base-url` | Agent API base URL. Default `https://api.vlm.run/v1`. |

Input types: `.jpg .jpeg .png .gif .bmp .tiff .webp`, `.mp4 .mov .avi .mkv .webm`, `.pdf .doc .docx`,
`.mp3 .wav .m4a .flac .ogg`. Multiple inputs upload at the same time.

The formatted reply shows the session id in the panel title and the credits used in the footer. With
`--format json`, add `-ns` to get `session_id` in the JSON: the streamed JSON has only `content`, `latency_s` and
`usage`.

### Orion-2

The docs recommend `vlmrun-orion-2:auto` (code execution; variants `fast`, `auto`, `pro`). `vlmrun chat` 0.9.3
accepts only `vlmrun-orion-1:*` and rejects Orion-2 with `Invalid model`. Call Orion-2 through the API:

```bash
curl -sS https://api.vlm.run/v1/openai/chat/completions \
  -H "Authorization: Bearer $VLMRUN_API_KEY" -H "Content-Type: application/json" \
  -d '{"model": "vlmrun-orion-2:auto", "messages": [{"role": "user", "content": [
        {"type": "text", "text": "What is the total on this receipt?"},
        {"type": "image_url", "image_url": {"url": "https://storage.googleapis.com/vlm-data-public-prod/hub/examples/document.receipt/playground/2.jpg"}}]}]}'
```

In Python: `VLMRun().agent.completions.create(model="vlmrun-orion-2:auto", messages=[...])`.

## Examples

### Images

```bash
vlmrun chat "Describe what you see in this image in detail" -i photo.jpg
vlmrun chat "Detect and list all objects visible in this scene" -i scene.jpg
vlmrun chat "Extract all text and numbers from this document image" -i document.png
vlmrun chat "Compare these two images and describe the differences" -i before.jpg -i after.jpg
```

### Image generation and editing

```bash
vlmrun chat "Generate a photorealistic image of a cozy cabin in a snowy forest at sunset" -o ./generated
vlmrun chat "Remove the background from this product image and make it transparent" -i product.jpg -o ./output
```

### Video

```bash
vlmrun chat "Summarize the key points discussed in this meeting video" -i meeting.mp4
vlmrun chat "Find the top 3 highlight moments and create short clips from them" -i sports.mp4
vlmrun chat "Transcribe this lecture with timestamps for each section" -i lecture.mp4 --format json -ns
```

### Video generation

```bash
vlmrun chat "Generate a 5-second video of ocean waves crashing on a rocky beach at golden hour" -o ./videos
vlmrun chat "Create a smooth slow-motion video from this image" -i ocean.jpg -o ./output
```

### Documents

```bash
vlmrun chat "Extract the vendor name, line items, and total amount" -i invoice.pdf --format json -ns
vlmrun chat "Summarize the key terms and obligations in this contract" -i contract.pdf
```

### Prompt sources

The first source present wins: the argument, then `-p`, then piped stdin.

```bash
vlmrun chat "What objects and people are visible in this image?" -i photo.jpg   # argument
vlmrun chat -p long_prompt.txt -i photo.jpg                                       # file
echo "Describe this image in detail" | vlmrun chat - -i photo.jpg                 # stdin
echo "Describe this image in detail" | vlmrun chat -p stdin -i photo.jpg          # stdin
```

### Continue a session

A session keeps the chat history and its artifacts. Take the session id from the panel title, or from
`--format json -ns`.

```bash
vlmrun chat "Create an iconic scene of a ninja in a forest, practicing with a katana" -i photo.jpg
vlmrun chat "Create a new scene with the same character meditating under a tree" -s <session_id>
```

Attach the input again with `-i` when the follow-up must read it.

### Skip the artifact download

```bash
vlmrun chat "What objects and people are visible in this image?" -i photo.jpg -nd
```

Download one later with `vlmrun artifacts get <object_id> -s <session_id> [-o path]`.

### Inline skills

`-k/--skill` sends a local skill directory with the request. No upload is needed. The directory must contain
`SKILL.md` (YAML frontmatter `name` and `description` is optional) and may hold more files (schemas, scripts,
assets).

```bash
vlmrun chat "Extract the key fields from this receipt" -i receipt.jpg --skill ./receipt-extraction-skill
vlmrun chat "Parse this invoice" -i invoice.pdf --skill ./invoice-skill --format json
vlmrun chat "Analyze this chart" -i chart.png --skill ./chart-analysis-skill -m vlmrun-orion-1:pro
vlmrun chat "Summarize and extract tables" -i report.pdf --skill ./summarize-skill --skill ./table-extraction-skill
vlmrun chat "" -i form.pdf --skill ./form-extraction-skill       # the skill's instructions drive the request
vlmrun chat "Detect objects and visualize" -i photo.jpg --skill ./detection-skill -t viz
```

```
my-skill/
├── SKILL.md        # required: instructions, optional YAML frontmatter
├── schema.json     # optional: output JSON schema
└── assets/         # optional
```

Use `--skill` while you iterate. When the skill is ready, `vlmrun skills upload ./my-skill`, then refer to it with
`--skill-id`.

### Server-side skills

Versions look like `20260911-399ea5e3`. `latest` picks the newest; a bare name means `latest`.

```bash
vlmrun chat "Extract the key fields" -i receipt.jpg --skill-id receipt-extraction:latest
vlmrun chat "Parse this document" -i doc.pdf --skill-id invoice-parser:20260312-abc
vlmrun chat "Analyze this form" -i form.pdf --skill-id form-analysis:latest --format json
```

## Skills management

```bash
vlmrun skills list                          # -n/--limit (default 25), --offset, --order-by, --asc, -j
vlmrun skills list --grouped                # latest version of each name
vlmrun skills get my-skill                  # -V <version> to pin, -j for JSON
vlmrun skills upload ./my-skill             # name and description from SKILL.md; override with -n / -d
vlmrun skills download my-skill -o ./dir    # extracts to ./dir/my-skill/; default ~/.vlmrun/skills
vlmrun skills create --prompt "Extract invoice fields" --schema schema.json
vlmrun skills create --prompt-file prompt.md
vlmrun skills create --file-id <uploaded-zip-id> -n my-skill
```

There is no `skills update` command, and `skills create` has no `--session-id`.

## Notes

- `-o ./<dir>` saves artifacts relative to the working directory.
- `vlmrun chat` takes local files only. For a URL, download it first, or use the API with an `image_url` part.
- `vlmrun execute` runs a named agent as a job (`-n <agent>:<version>`, `--no-wait`, `--callback-url`). See
  `vlmrun execute --help`.
