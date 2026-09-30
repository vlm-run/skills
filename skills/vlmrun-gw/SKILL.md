---
name: vlmrun-gw
description: Use the `vlmrun` Python SDK and CLI. The `vlmrun gw` CLI parses documents and runs chat completions over text, images and video on the VLM Run gateway (gateway.vlm.run), plus multimodal embeddings, audio transcription, and System One typed decisions (noul / choice / score) over text, JSON, images and PDFs on the TypeSafe-compatible `/typesafe` route. The same gateway calls work from Python through the OpenAI SDK or `VLMRun().gateway`; System One also works through `typesafe-sdk` with `TYPESAFE_BASE_URL=https://gateway.vlm.run/typesafe`. The `VLMRun` client and `vlmrun generate` / `vlmrun execute` cover the platform API — schema-typed predictions, agent executions, files and hub domains. Covers every command and flag, model selection, methods, request knobs, response shapes, cost and errors. Use when asked to read/parse/OCR a PDF, scan, receipt or form, convert a document to markdown, extract text with bounding boxes, get a document's layout or tables, describe or answer questions about an image or video, embed images or text for search, transcribe speech, classify or route a document with calibrated probabilities, gate a pipeline with yes/no or multi-label decisions over vision, extract structured JSON from a document, or run and poll an agent execution.
license: Apache-2.0
---

# `vlmrun` SDK and `vlmrun gw` CLI

> VLMRun Gateway is a fully OpenAI-compatible API for visual intelligence, supporting OCR, visual question answering, detection, segmentation, and captioning behind a single endpoint. Point base_url to https://gateway.vlm.run/v1/openai and gain access to open-weight ocr, vlm, vite-pose and segmentation models without changing existing setup.

`vlmrun gw` (alias `vlmrun gateway`) ships with the [`vlmrun` Python SDK](https://github.com/vlm-run/vlmrun-python-sdk)
and drives `https://gateway.vlm.run/v1`: OCR, document-parsing, vision-language, embedding and
transcription models behind one OpenAI-compatible API, plus System One on `https://gateway.vlm.run/typesafe`
(typed `noul` / `choice` / `score` decisions, TypeSafe/Jev-compatible). This skill document is everything an agent needs
to use the CLI in accessing the Gateway: which command and model to pick, how to shape a request, what comes back, what it costs, and what each error means. It also covers the rest of the SDK: the same gateway calls from Python (OpenAI SDK, `VLMRun().gateway`), System One via `typesafe-sdk` or `vlmrun gw s1`, and the platform API (`VLMRun` client, predictions, agent executions).

**Use it for:** All `vlmrun gw` commands, such as documents → markdown or structured OCR; chat completions over text, images and video; multimodal embeddings; speech transcription; [System One](#system-one-typesafe-decisions) reads (typed `noul` / `choice` / `score` decisions with calibrated probabilities over text, JSON, images and PDFs). The gateway is a passthrough: one model, one call → the response. Use System One when the answer is a label or probability, not generated text. Use `vlmrun gw chat` when you need markdown, a caption, or a free-form answer. Use the [platform API](#platform-api-predictions-agents-and-executions) when the ask needs a hub domain schema (`document.invoice`), your own Pydantic `response_model`, or an agent that runs as a job.

## Quickstart

```bash
# No install: uvx runs the CLI straight from PyPI
uvx --from vlmrun vlmrun gw models
uvx --from vlmrun vlmrun gw chat report.pdf -m zai-org/glm-ocr

# Or install once
pip install -U vlmrun && vlmrun config set --api-key <key>
# Get your API key (VLMRUN_API_KEY) from https://app.vlm.run
```

Auth is a bearer token, the same `VLMRUN_API_KEY` used across VLM Run:

| Caller    | How                                                                                 | Rate limit                          |
| --------- | ----------------------------------------------------------------------------------- | ----------------------------------- |
| Anonymous | no `Authorization` header, or `Bearer vlmrun`; for the CLI, `VLMRUN_API_KEY=vlmrun` | **10/min · 30/hr · 100/day** per IP |
| API key   | `Authorization: Bearer <VLMRUN_API_KEY>`                                                 | **240/min · 10 000/hr**             |

Anonymous access needs no signup. Good for basic exploration and test runs. Anonymous replies carry advisory `X-RateLimit-Limit` / `X-RateLimit-Remaining` headers. If a command picks up the wrong deployment, clear stale exports first: `unset VLMRUN_API_KEY VLMRUN_BASE_URL VLMRUN_GATEWAY_URL`.

## Commands

| Command                                                                                                                                                                 | Does                                                                         |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `vlmrun gw models [<model>] [--json]`                                                                                                                                   | List the catalog, or one model's methods, params and example commands        |
| `vlmrun gw chat <files/urls>... -m <model> [-p <prompt>] [--method M] [--method-params JSON] [-e k=v]... [--json-mode \| --response-format F] [-ns] [-j] [--timeout S]` | OCR, document parsing and chat completions over text, images, video and PDFs |
| `vlmrun gw embed [<files>...] [-t <text>]... -m <model> [--join] [-d N] [-j] [--timeout S]`                                                                             | Multimodal embeddings                                                        |
| `vlmrun gw transcribe <file> \| --url <url> -m <model> [-f json\|text\|verbose_json\|srt\|vtt] [-l xx] [-p <hint>] [-j] [--timeout S]`                                  | Speech to text                                                               |
| `vlmrun gw s1 [<files>...] -s <state> [-m <model>] [--noul ID[=text]] [--choice ID="a\|b\|c"] [--score ID="l0\|l1\|..."] [-Q JSON] [--detail auto\|low\|high] [--gate expr]... [-j] [--timeout S]` (alias `systemone`) | Typed decisions: yes/no, multi-class, ordered score, batched in one read     |
| `vlmrun gw health`                                                                                                                                                      | Liveness probe                                                               |

The CLI reads local files or `http(s)` URLs, sniffs the MIME type, picks the right content part
(`document_url` / `image_url` / `video_url`), streams by default, and prints `usage.cost` and
throughput in a footer. `-e key=value` is repeatable; values parse as JSON when they can
(`-e temperature=0`, `-e stop='["\n"]'`, `-e document_pages='[[0,10]]'`). Standard OpenAI fields
(`temperature`, `max_tokens`, `top_p`, `stop`, `n`, `logprobs`, ...) and gateway extensions
(`method`, `document_dpi`, `video_fps`, ...) both go through `-e`; the CLI routes each to the
right place. `--json` / `-j` prints the raw response for scripting.

The gateway URL comes from `VLMRUN_GATEWAY_URL` (default `https://gateway.vlm.run/v1`); it is
independent of `VLMRUN_BASE_URL`, which points the rest of the CLI at the platform API.

## The same requests from Python

The gateway is OpenAI-compatible. Any OpenAI client works with `base_url=https://gateway.vlm.run/v1/openai`.
Gateway extensions (`method`, `document_dpi`, `video_fps`, ...) go in `extra_body`; they map 1:1 to `-e key=value` on the CLI.

```python
import base64, os, pathlib
from openai import OpenAI

client = OpenAI(base_url="https://gateway.vlm.run/v1/openai", api_key=os.environ["VLMRUN_API_KEY"], timeout=600)
pdf = base64.b64encode(pathlib.Path("report.pdf").read_bytes()).decode()
r = client.chat.completions.create(
    model="zai-org/glm-ocr",
    messages=[{"role": "user", "content": [
        {"type": "document_url", "document_url": {"url": f"data:application/pdf;base64,{pdf}"}},
    ]}],
    extra_body={"method": "markdown", "document_dpi": 96, "document_pages": [[0, 10]]},
)
print(r.choices[0].message.content, r.usage.model_extra["cost"])

client.embeddings.create(model="qwen/qwen3-vl-embedding-2b", input=["a blue parrot"], dimensions=64)
client.audio.transcriptions.create(model="nvidia/parakeet-tdt-0.6b-v3", file=open("clip.mp3", "rb"), response_format="srt")
client.models.list()                      # catalog; extra fields on model.model_extra
```

Content parts: `image_url` (`http(s)` or `data:image/...;base64,`), `video_url`, `document_url`. Use `http(s)` URLs when you can; base64 is for local files.

`VLMRun().gateway` is the same OpenAI client, pre-wired. It reads `VLMRUN_API_KEY` and `VLMRUN_GATEWAY_URL`, and raises the default timeout from 120 s to 600 s (an explicit `VLMRun(timeout=...)` is kept as is).

```python
from vlmrun.client import VLMRun

client = VLMRun()
gw = client.gateway
gw.completions.create(model="zai-org/glm-ocr", messages=[...], extra_body={...})   # gw.async_completions for asyncio
gw.embeddings.create(model="qwen/qwen3-vl-embedding-2b", input=["text"])
gw.transcriptions.create(model="nvidia/parakeet-tdt-0.6b-v3", file=open("clip.mp3", "rb"))
gw.models()        # list of OpenAI Model objects
gw.health()        # bool
```

Raw HTTP works too: `POST https://gateway.vlm.run/v1/openai/chat/completions` with `Authorization: Bearer <VLMRUN_API_KEY>` and the same JSON body, extensions at the top level next to `model` and `messages`. Prefer the OpenAI client. It handles streaming, retries and typing for you.

## Models

Always list and inspect the model catalog for an accurate list of serving models:

```bash
vlmrun gw models                       # every model with task, methods and inputs
vlmrun gw models zai-org/glm-ocr       # methods, params and example commands for one model
vlmrun gw models --json                # raw catalog, including per-model capabilities
```

Each model has a `task` (`chat`, `embed`, `transcribe`), a list of `methods` with a default, the
inputs it accepts, and `capabilities` such as `supports_json_schema`. Use the full `<org>/<name>` id or name alias.

Examples of models per category and workflow (check `vlmrun gw models` for the current list):

| Category | Example models | Typical methods | Use for |
| --- | --- | --- | --- |
| Document to markdown | `zai-org/glm-ocr`, `infly/infinity-parser2-flash`, `baidu/unlimited-ocr` | `markdown` | Contracts, reports, books. Start with `glm-ocr`. |
| Layout-aware parsing | `rednote-hilab/dots.mocr`, `deepseek-ai/deepseek-ocr-2` | `parse_layout`, `grounding_ocr`, `markdown` | Keep headings and tables, locate text on the page |
| OCR with regions | `paddleocr/pp-ocrv6`, `microsoft/florence-2-base-ft` | `ocr`, `detect`, `text`, `ocr_with_region` | Text with confidence and boxes, receipts, forms |
| Tables, formulas, charts | `paddlepaddle/paddleocr-vl-1.6` | `table`, `formula`, `chart` | Statements, papers, dashboards |
| Vision and captioning | `microsoft/florence-2-base-ft` | `caption`, `detailed_caption`, `od` | Captions and object detection on one image |
| Pose | `usyd-community/vitpose-plus-large` | `pose` | 2D human pose on images and video |
| Chat VLMs | `google/gemini-3.7-flash`, `google/gemini-3.5-flash-lite`, `qwen/qwen3.8-27b`, `qwen/qwen3.5-0.8b` | none (plain prompt) | Questions about images, video, or a whole PDF (Gemini) |
| Embeddings | `qwen/qwen3-vl-embedding-2b` | `embed` | Text, image and video vectors for search |
| Transcription | `nvidia/parakeet-tdt-0.6b-v3` | `transcribe` | Speech to text, subtitles |

Rules of thumb:

- For text on a page, use an OCR or document model. Small chat VLMs misread dense text, and an
  OCR model costs less.
- Use a chat VLM when the question is about a scene or video understanding, or when it spans an entire PDF. Only chat VLMs that list `document_url` in their inputs accept a PDF with a prompt.
- Text-only prompts work on chat VLMs only. OCR models reject them.

## Documents

The gateway rasterises a PDF server-side, runs the model per page, and stitches the pages back.
You do not split or loop on your side.

```bash
vlmrun gw chat report.pdf -m zai-org/glm-ocr                      # → markdown per page
vlmrun gw chat scan.png -m paddleocr/pp-ocrv6 --method ocr        # → regions with scores + polygons
vlmrun gw chat sheet.pdf -m rednote-hilab/dots.mocr --method parse_layout
vlmrun gw chat statement.pdf -m paddlepaddle/paddleocr-vl-1.6 --method table
vlmrun gw chat form.png -m deepseek-ai/deepseek-ocr-2 --method grounding_ocr \
    --method-params '{"prompt": "<image>\nLocate <|ref|>title<|/ref|> in the image."}'
```

Every document reply uses the same envelope: one `<document>` per input PDF, one `<page>` per
page. Example output for a 1-page PDF through `zai-org/glm-ocr`:

```xml
<document mimetype="application/pdf" num_pages="1" dpi="96">
<page page_index="0" format="markdown" page_width="900" page_height="500">
<table><tr><td>Invoice #A-1042</td></tr><tr><td>TOTAL: $46.25</td></tr></table>
</page>
</document>
```

- `format` is `markdown` for markdown methods, `json` for region methods (`ocr`, `detect`,
  `parse_layout`).
- A page that fails is self-closing with `status="error"`. The document still returns with page
  numbering intact. Do not assume a missing body means an empty page.
- `<document>` also carries `file_name` for `http(s)` inputs (omitted for data URIs).

### Document knobs

| Field                              | Default                          | Meaning                                                                                                                                           |
| ---------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `method`                           | `markdown` when the model has it | Per-page backend method                                                                                                                           |
| `document_dpi`                     | `96`                             | Rasterisation DPI; honoured to 400 while the page's long edge fits 20 000 px, else `min(dpi, 140)`. Raise it for fine print.                      |
| `document_pages`                   | all                              | 0-indexed pages and/or `[start, stop)` ranges, e.g. `[[0, 10], 15]`. Only those pages are rasterised, so sections cost no more than one big call. |
| `document_max_pages`               | `128`                            | Per-request page cap                                                                                                                              |

Hard limits: at most 8 `document_url`/`file_url` parts per request; documents cannot be mixed with
`image_url` or `video_url` in the same request. A PDF longer than `document_max_pages` returns a 400
that names the ranges to use. Read it in sections with `document_pages`. Do not truncate.

## Chat completions over text, images and video

Chat VLMs take a plain prompt with no `--method`. Their reply passes through verbatim.

```bash
vlmrun gw chat -m qwen/qwen3.5-0.8b -p "In one sentence: what is OCR?"     # text only, no file
vlmrun gw chat street.jpg -p "How many people are wearing helmets?" -m google/gemini-3.7-flash
vlmrun gw chat a.jpg b.jpg c.jpg -p "What changed between these?" -m qwen/qwen3.8-27b
vlmrun gw chat clip.mp4 -p "Describe what happens" -m qwen/qwen3.8-27b
vlmrun gw chat https://example.com/photo.jpg -p "Describe it" -m google/gemini-3.5-flash-lite
vlmrun gw chat img.jpg -p "Reply as JSON with keys label,count" -m qwen/qwen3.8-27b --json-mode
```

- Multi-image: chat VLMs reason over all images and answer once (`≤64` for the qwen models);
  structured models run per image and return one block each.
- Video: one clip per request. The chat VLMs and Gemini models ingest video natively.
- Streaming is on by default (`-ns` / `--no-stream` to disable).

### Video sampling

| Field              | Default                 | Meaning                                                           |
| ------------------ | ----------------------- | ----------------------------------------------------------------- |
| `video_fps`        | uniform across the clip | Target sample rate                                                |
| `video_max_frames` | `8`                     | Hard cap on sampled frames                                        |
| `video_resolution` | source                  | 4:3 preset: `256x192`, `320x240`, `448x336`, `512x384`, `640x480` |

```bash
vlmrun gw chat clip.mp4 -p "How many distinct scenes?" -m google/gemini-3.7-flash \
    -e video_fps=1 -e video_max_frames=16
```

Both fields are gateway extensions. Natively served models and Gemini honour them. Other
provider-routed models reject `video_fps` with a 400. Either field without a video input is also
a 400.

For images, `image_resolution` accepts `224x224`, `336x336`, `384x384`, `448x448`, `512x512`,
`768x768`. The gateway resizes and center-crops before the model's own processor.

## Embeddings

```bash
vlmrun gw embed photo.jpg -m qwen/qwen3-vl-embedding-2b            # 1 vector, 2048 dims
vlmrun gw embed a.jpg b.jpg -t "a caption" -m qwen/qwen3-vl-embedding-2b   # 3 vectors
vlmrun gw embed photo.jpg -t "its caption" --join -m qwen/qwen3-vl-embedding-2b   # 1 fused vector
vlmrun gw embed -t "a blue parrot" -m qwen/qwen3-vl-embedding-2b -d 64
vlmrun gw embed photo.jpg -m qwen/qwen3-vl-embedding-2b --json      # full vectors for a store
```

Every file and every `-t` is its own vector unless `--join` fuses them (max one file per fused
vector). Images, video and text only. The gateway rejects a PDF, so OCR it and embed the text.

Report vector count and dimension. Never paste raw floats into a reply.

## Transcription

```bash
vlmrun gw transcribe clip.mp3 -m nvidia/parakeet-tdt-0.6b-v3
vlmrun gw transcribe meeting.mp4 -m nvidia/parakeet-tdt-0.6b-v3        # video's audio track
vlmrun gw transcribe clip.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -f srt    # subtitles
vlmrun gw transcribe clip.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -f verbose_json   # timestamps
vlmrun gw transcribe --url https://example.com/a.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -l en
vlmrun gw transcribe call.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -p "Acme, Käthe, Kubernetes"
```

`-f srt`/`vtt` for subtitles, `verbose_json` for timestamps, `-l` for a language hint, `-p` to bias
proper nouns. A clip with no speech returns an empty transcript, not an error.

## System One (TypeSafe decisions)

System One answers **typed questions** about a **state** (plain text, JSON object/array, or vision)
and returns **calibrated probabilities**. Nothing is generated and nothing is parsed, so an answer
cannot be off-schema. The route is wire-compatible with TypeSafe's Jev API: point `typesafe-sdk` at
`https://gateway.vlm.run/typesafe` and existing clients work unchanged.

| Surface | Base URL | Endpoint |
| --- | --- | --- |
| HTTP | `https://gateway.vlm.run/typesafe` | `POST /v1/systemone`, `GET /v1/models` |
| WebSocket (per-frame video/stream) | `wss://gateway.vlm.run/typesafe` | `WS /ws` (same question set, binary frames in) |

Auth is the same bearer token as the rest of the gateway: `Authorization: Bearer <VLMRUN_API_KEY>`.
Anonymous tier (`Bearer vlmrun` / `VLMRUN_API_KEY=vlmrun`) applies here too, with the same IP quotas
as [Quickstart](#quickstart). For the official SDK, set `TYPESAFE_API_KEY` to the same key (or pass
`api_key=` in code). The SDK appends `/v1/systemone` to `TYPESAFE_BASE_URL`; there is no mapping layer.

```bash
export TYPESAFE_BASE_URL="https://gateway.vlm.run/typesafe"
export TYPESAFE_API_KEY="$VLMRUN_API_KEY"
# Required: upstream `jev-latest` is not served here; name a real model id.
export TYPESAFE_DEFAULT_MODEL="google/diffusiongemma-26b-a4b-it"
```

`VLMRUN_GATEWAY_URL` does **not** cover System One; keep `/typesafe` separate from `/v1/openai`.

### When to use which entry point

| Ask | Use | Why |
| --- | --- | --- |
| "Is this scan an invoice?" (probability) | System One `noul` | One calibrated P(yes), never a malformed yes/no string |
| "Which of these 6 document kinds?" | System One `choice` | Full label distribution + `confidence` |
| "Gate the expensive OCR pass" | System One | Cheap read before `vlmrun gw chat` |
| "Transcribe this page to markdown" | `vlmrun gw chat` | Free-form generation |
| "Extract line items as JSON" with a hub schema | Platform `vlmrun generate` | Typed job + optional grounding, not a single forward read |

Default vision engine: `google/diffusiongemma-26b-a4b-it`. List the current catalog:

```bash
curl -sS https://gateway.vlm.run/typesafe/v1/models -H "Authorization: Bearer $VLMRUN_API_KEY"
vlmrun gw s1 --help    # CLI flags; `s1` is shorthand for `systemone`
```

Other served ids (check `GET /typesafe/v1/models`): `qwen/qwen3.5-0.8b`, `google/gemma-4-26b-a4b-it`,
`qwen/qwen3.8-27b`. There is no `jev-latest` alias; an unknown `model` is `404`.

### Question types

Up to **16 questions** per request, keyed by caller-chosen ids (ids key `answers`, they are not sent
to the model). Three `type` values:

| `type` | Question shape | Answer fields |
| --- | --- | --- |
| `noul` | Yes/no; optional `criteria.true` / `criteria.false` descriptions | `noul`: P(yes) in [0, 1] |
| `choice` | `criteria`: map of 2–128 labels → description or `null` | `choice`, `probabilities`, `confidence` |
| `score` | `criteria`: ordered array of 2–10 level strings | `score` (expected level, 0-indexed), `legend`, `probabilities`, `confidence` |

`confidence` is `1 - H(p)/ln(K)` (1 = peaked, 0 = uniform). Threshold on it for human review or
fallback to a larger model. Probabilities are model-specific; revalidate thresholds when you change
`model`.

### Request body (`POST /typesafe/v1/systemone`)

| Field | Required | Meaning |
| --- | --- | --- |
| `state` | yes | Text, JSON object, or JSON array the questions refer to |
| `model` | yes | Served id from `/typesafe/v1/models` |
| `questions` | yes | Map id → `{type, instructions?, criteria?}` |
| `content` | no | Media as OpenAI-style parts (see below). Bare string = one `text` part |
| `reasoning_effort` | no | Generative engines only: `none` (default), `minimal`, `low` (`medium`/`high` → `422` today) |
| `steps` | no | Diffusion engines: denoise steps 1–8 (default `1`, Jev contract) |
| `samples` | no | Noise draws to average, 1–32 (each draw is billed) |

**Content parts** (in `content`, or pass files as `vlmrun gw s1` positionals after `-s`):

| Part | Limit | Notes |
| --- | --- | --- |
| `image_url` | ≤8 images, 5 MB each | `url` must be a `data:image/...;base64,...` URL (`http(s)` rejected) |
| `file` or `document_url` | 1 PDF | First 8 pages rasterised at 96 DPI; `file_data` may be `data:...` or `http(s)` |
| `text` | any | Appended after `state` |

Send **images or one document, never both** (`422`). `detail` per part: `auto`/`high` = 280 vision
tokens, `low` = 70. Use `low` for coarse routing/classification when fine print does not matter.

Unknown top-level fields → `422` (stricter than upstream Jev, which ignores extras).

### Response

```json
{
  "model": "google/diffusiongemma-26b-a4b-it",
  "answers": {
    "kind": {
      "type": "choice",
      "choice": "invoice",
      "probabilities": { "invoice": 0.88, "receipt": 0.07, "contract": 0.02, "other": 0.03 },
      "confidence": 0.64
    },
    "has_total": { "type": "noul", "noul": 0.91 }
  },
  "usage": {
    "input_tokens": 486,
    "output_tokens": 0,
    "input_tokens_details": { "cached_tokens": 0, "image_tokens": 280, "text_tokens": 206 },
    "reads": 1,
    "cost": 0.000129
  }
}
```

`usage.reads`, `usage.cost`, and `usage.input_tokens_details` are gateway extensions; `typesafe-sdk`
ignores unknown fields and still parses. Responses include `x-typesafe-request-id` (mirrors
`x-request-id`). Report winning labels, key probabilities, `confidence`, `usage.reads`, and
`usage.cost`.

### CLI: `vlmrun gw s1`

```bash
# Vision + batched questions (invoice page)
vlmrun gw s1 -s "Answer about the attached page." invoice-page.jpg --detail high \
  -m google/diffusiongemma-26b-a4b-it \
  --choice kind="invoice|receipt|contract|other" \
  --noul has_total="Is a total amount due visible on the page?" \
  --score legibility="unreadable|partly legible|clear"

# PDF routing (first 8 pages)
vlmrun gw s1 -s "Answer about the document." packet.pdf --detail high \
  -m google/diffusiongemma-26b-a4b-it \
  --choice kind="invoice|contract|report|other" \
  --noul has_signature="Is a signature block visible?"

# Text-only gate (no file)
vlmrun gw s1 -s "Customer email: please refund order 42 immediately." \
  -m qwen/qwen3.5-0.8b \
  --noul refund_requested="Does the state request a refund?" \
  --choice urgency="low|medium|high" -j

# Script gate: exit 1 if condition fails
vlmrun gw s1 -s "..." scan.jpg -m google/diffusiongemma-26b-a4b-it \
  --noul is_invoice="Is this an invoice?" --gate 'is_invoice>0.8' \
  && vlmrun gw chat scan.jpg -m zai-org/glm-ocr
```

`-Q` accepts the wire map or a list form (`@file.json` / stdin). `--body` sends a full JSON body;
other flags override. Video: sample frames with `--fps` / `--max-frames`, or `--ws` for one WebSocket
session. `--dry-run` prints the JSON without sending.

### Python (`typesafe-sdk`)

```python
import base64, os, pathlib
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

client = TypeSafeClient(
    api_key=os.environ["VLMRUN_API_KEY"],
    base_url="https://gateway.vlm.run/typesafe",
)
page = base64.b64encode(pathlib.Path("invoice-page.jpg").read_bytes()).decode()

result = client.system_one(
    "Answer about the attached page.",
    {
        "kind": Choice(
            instructions="What kind of document is this page from?",
            criteria={"invoice": None, "receipt": None, "contract": None, "other": None},
        ),
        "has_total": Noul(instructions="Is a total amount due visible on the page?"),
        "legibility": Score(
            instructions="How legible is the text?",
            criteria=["unreadable", "partly legible", "clear"],
        ),
    },
    extra_body={
        "content": [
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{page}", "detail": "high"}},
        ]
    },
    model="google/diffusiongemma-26b-a4b-it",
)
print(result.choices["kind"].choice, result.nouls["has_total"].noul, result.usage.cost)
```

`pip install typesafe-sdk`. TypeScript: `@typesafe-ai/sdk` with `baseURL: "https://gateway.vlm.run/typesafe"`.

### Multi-gateway / Jevify-style base URL

Any tool that speaks TypeSafe's Jev HTTP API can use VLM Run as a drop-in host: set base URL to
`https://gateway.vlm.run/typesafe`, swap the API key to `VLMRUN_API_KEY`, and set an explicit
`model` (or `TYPESAFE_DEFAULT_MODEL`). Shapes, question types, and answer fields match; only gateway
extensions (`content`, `reasoning_effort`, `usage.cost`, vision) differ. Do not point `TYPESAFE_BASE_URL`
at `/v1/openai`; that path is chat completions only.

### System One errors

| Status | When | Fix |
| --- | --- | --- |
| `422` | Invalid body, unknown field, images+PDF together, `medium`/`high` reasoning, diffusion + `reasoning_effort` | Read `detail[]`; fix schema; send only images or one PDF |
| `404` | `model` not on `/typesafe` | `GET /typesafe/v1/models`; avoid `jev-latest` |
| `429` | Rate limit | Same tiers as gateway; back off or authenticate |
| `503` | Engine failure | Retry after `retry-after` (often 2 s) |
| `529` | Overloaded queue | Retry after `retry-after` (often 1 s) |

OpenAPI and field-level reference: `https://gateway.vlm.run/openapi.json` (paths under `/typesafe/`),
docs: `https://docs.vlm.run/gateway/system-one`.

## Request knobs, all of them

Standard OpenAI fields (`temperature`, `max_tokens`, `top_p`, `frequency_penalty`,
`presence_penalty`, `stop`, `n`, `logprobs`) and the gateway extensions below all go through
`-e key=value`. `--method` / `--method-params` are shorthands for the first two rows.

| Field                | Type    | Default         | Applies to                                     |
| -------------------- | ------- | --------------- | ---------------------------------------------- |
| `method`             | str     | model's default | Which operation the model runs                 |
| `method_params`      | object  | none            | That method's arguments                        |
| `document_dpi`       | int     | `96`            | PDF rasterisation                              |
| `document_pages`     | list    | all             | Page/range selection                           |
| `document_max_pages` | int     | `128`           | Page cap                                       |
| `video_fps`          | float   | uniform         | Video decode rate                              |
| `video_max_frames`   | int     | `8`             | Frame cap                                      |
| `video_resolution`   | preset  | source          | Frame resize                                   |
| `image_resolution`   | preset  | source          | Image resize                                   |
| `precision`          | int 1–8 | `4`             | Decimal places on normalised coords and scores |

### `response_format`

`method` selects _what_ is computed; `response_format` selects _how_ it is serialised. Any
combination is valid.

| Value                                                            | Reply                                                                         |
| ---------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| omitted / `--response-format text`                               | Text blocks (the envelope shown above)                                        |
| `--json-mode` (= `--response-format json_object`)                | One JSON object: `{"model": …, "method": …, "data": [<entry per input>]}`     |
| `--response-format '{"type":"json_schema","json_schema":{...}}'` | Schema-constrained generation, chat VLMs and provider-routed models only       |

`json_schema` on an OCR, detection or layout model returns a 400 `capability_violation` on
`param: "response_format"`. Those replies are an envelope the gateway assembles, so it cannot apply
a schema. Use `--json-mode` there; every model supports it. Check `capabilities.supports_json_schema`
in the catalog. Chat VLM output always passes through verbatim, with no JSON envelope.

## Response shapes

| Input               | Text-mode shape                                                              |
| ------------------- | ---------------------------------------------------------------------------- |
| One image           | The bare block, no wrapper                                                   |
| Several images      | Repeated `<image image_hash=… format=… image_width=… image_height=…>` blocks |
| PDF                 | `<document …>` with one `<page …>` per page                                  |
| Chat VLM, any input | The model's own text, verbatim                                               |

Region payloads are self-describing, with normalised coordinates. `pp-ocrv6 --method ocr` output:

```json
{
  "object": "pp_ocrv6.ocr.regions",
  "items": [
    {
      "bbox_xywh": [0.0522, 0.078, 0.0978, 0.026],
      "poly_xy": [
        [0.0522, 0.078],
        [0.15, 0.078],
        [0.15, 0.104],
        [0.0522, 0.104]
      ],
      "text": "ACME SUPPLY CO.",
      "score": 0.9864
    }
  ]
}
```

Multiply by pixel width/height to get absolute coordinates; `precision` controls the decimals.
Responses also carry `served_model_id` (the canonical id even when you sent an alias) and
`backend` (`vllm`, `transformers`, ...). Log them when you compare results across runs.

## Cost and latency

`usage.cost` is USD on the response (non-streaming), or on the final SSE chunk when streaming.
Document models bill per page; the rest bill on tokens. A single OCR page or a short chat
completion costs a fraction of a cent, and a warm model answers in about a second.

Cold starts are the one latency trap. A model with no warm replica can take minutes on the first
request; the same call on a warm replica takes about a second. Pass a generous `--timeout` on a
first call, and do not read a slow first response as a hang.

## Errors, with the messages

| Status / code                                        | Message shape                                                                                       | Fix                                                              |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| 400 `capability_violation`, `param: messages`        | `Model 'zai-org/glm-ocr' requires at least one image_url, video_url, or document_url content part.` | Attach a file; text-only needs a chat VLM                        |
| 400 `capability_violation`, `param: method`          | `Model 'zai-org/glm-ocr' does not support method 'parse_layout'; supported: markdown.`              | Use a listed method, or drop `--method`                          |
| 400 `capability_violation`, `param: response_format` | schema unsupported                                                                                  | Use `--json-mode`                                                |
| 404 `model_not_found`                                | Message lists every available id                                                                    | Copy the right full `<org>/<name>`                               |
| 400 `invalid_media`                                  | Fetch/decode failure on an image or video                                                           | Check the URL, content type and file bytes                       |
| 400 `invalid_document`                               | Fetch/decode failure on a PDF                                                                       | Same; a 0-page or corrupt PDF fails here                         |
| 400 (document too long)                              | Names the ranges that cover the PDF                                                                 | Re-send with `document_pages` sections                           |
| 400 (`-e video_fps` on a routed model)               | Provider exposes no frame sampling                                                                  | Drop the flag, or use a natively served or Gemini model          |
| 403                                                  | `Invalid API Key`                                                                                   | Key is wrong or shadowed; `unset` env vars, `vlmrun config show` |
| 422                                                  | Pydantic validation error                                                                           | Malformed request body                                           |
| 429                                                  | Rate limit; `X-RateLimit-*` headers                                                                 | Back off, or authenticate for the higher tier                    |
| 500                                                  | Sanitized error                                                                                     | Retry once; then report it                                       |

Retry a mapping error once, after you re-read the catalog. Do not loop. If the second attempt
fails, report the exact command and the response.

## How to run a request well

1. **Read the ask**: inputs (expand `~`; `http(s)` URLs are fine) and intent. Ask one short
   question only if the intent is ambiguous. A missing model is not ambiguity.
2. **Pick from the catalog**, not from memory: `vlmrun gw models`, then
   `vlmrun gw models <model>` for its methods, params and example commands. Honor any model or
   method the user named.
3. **Show the command** you are about to run, on one line, then run it. Default to the readable
   panel; add `--json` only for scripting or when the user wants raw output.
4. **Let the CLI encode.** It sniffs magic bytes and picks `document_url` / `image_url` /
   `video_url` itself, so a `.jpg` that is really WebP still works.
5. **Report** the extracted text, markdown, answer, transcript, or vector count and dimension,
   plus `usage.cost` when it is relevant, summed across a batch. If you chose the model or method
   yourself, say which and why in one line.

## Worked examples

| Ask                                     | Command                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------ |
| "parse this contract to markdown"       | `vlmrun gw chat contract.pdf -m zai-org/glm-ocr`                                     |
| "extract the text from this receipt"    | `vlmrun gw chat receipt.png -m paddleocr/pp-ocrv6 --method ocr`                      |
| "just the text, no boxes"               | `vlmrun gw chat receipt.png -m paddleocr/pp-ocrv6 --method text`                     |
| "where is the text on this form?"       | `vlmrun gw chat form.jpg -m paddleocr/pp-ocrv6 --method detect`                      |
| "get the layout of this drawing"        | `vlmrun gw chat sheet.pdf -m rednote-hilab/dots.mocr --method parse_layout`          |
| "pull the tables out of this statement" | `vlmrun gw chat statement.pdf -m paddlepaddle/paddleocr-vl-1.6 --method table`       |
| "read pages 10–20 of this book"         | `vlmrun gw chat book.pdf -m zai-org/glm-ocr -e document_pages='[[10,20]]'`           |
| "read this fine print"                  | `vlmrun gw chat scan.pdf -m zai-org/glm-ocr -e document_dpi=300`                     |
| "caption this photo"                    | `vlmrun gw chat photo.jpg -m microsoft/florence-2-base-ft --method detailed_caption` |
| "what's in this image?"                 | `vlmrun gw chat street.jpg -p "What's in this image?" -m google/gemini-3.7-flash`    |
| "summarise this video"                  | `vlmrun gw chat clip.mp4 -p "Summarise this clip" -m qwen/qwen3.8-27b`               |
| "where are the people looking?" (pose)  | `vlmrun gw chat crowd.jpg -m usyd-community/vitpose-plus-large`                      |
| "embed these product photos"            | `vlmrun gw embed shots/*.jpg -m qwen/qwen3-vl-embedding-2b --json`                   |
| "subtitle this recording"               | `vlmrun gw transcribe talk.mp4 -m nvidia/parakeet-tdt-0.6b-v3 -f srt`                |
| "is this scan an invoice?" (probability) | `vlmrun gw s1 -s "Answer about the page." scan.jpg -m google/diffusiongemma-26b-a4b-it --noul is_invoice="Is this an invoice?"` |
| "classify this PDF before OCR"          | `vlmrun gw s1 -s "Answer about the document." doc.pdf --choice kind="invoice\|contract\|other" -m google/diffusiongemma-26b-a4b-it` |
| "is the gateway up?"                    | `vlmrun gw health`                                                                   |

## Batches

Shell loop, one output per input, cost totalled:

```bash
for f in scans/*.pdf; do
  vlmrun gw chat "$f" -m zai-org/glm-ocr --json --timeout 600 \
    > "out/$(basename "${f%.pdf}").json"
done
jq -s 'map(.usage.cost // 0) | add' out/*.json
```

For large batches, bound concurrency (for example `xargs -P 4`) to stay under the rate limit, and
pass a generous `--timeout` for the first call to a cold model.

## Platform API: predictions, agents and executions

The gateway returns what one model says. The platform API (`https://api.vlm.run/v1`, `VLMRUN_BASE_URL`) returns
typed JSON: hub domains with fixed schemas, your own Pydantic schema, or an agent that runs as a job.
Same `VLMRUN_API_KEY`. Everything here goes through `VLMRun()`, never through `vlmrun gw`.

```python
from vlmrun.client import VLMRun
from vlmrun.client.types import AgentExecutionConfig, GenerationConfig
from pydantic import BaseModel

client = VLMRun()                                  # validates the key against /health on construction
```

### Predictions

| Call | Input | Notes |
| --- | --- | --- |
| `client.image.generate(images=[...] \| urls=[...], domain="document.invoice")` | PIL images or URLs | Synchronous by default. `batch=True` returns a job. |
| `client.document.generate(file=path \| file_id, url=..., domain=..., batch=True)` | PDF | Uploads the file (cached by hash) then submits. Same for `client.audio` and `client.video`. |
| `client.document.execute(name="my-agent", version="latest", ...)` | Same | Run a named agent instead of a domain |
| `client.predictions.get(id)` / `.wait(id, timeout=600, sleep=5)` / `.list()` | Job id | `wait` raises `TimeoutError` |
| `client.hub.list_domains()` / `client.hub.info()` | none | Every domain with its schema version |

```python
class Invoice(BaseModel):
    vendor: str
    total: float

r = client.document.generate(
    file="invoice.pdf", domain="document.invoice", batch=True,
    config=GenerationConfig(response_model=Invoice, confidence=True, grounding=False),
)
r = client.predictions.wait(r.id)
Invoice(**r.response)                              # wait returns a dict; a synchronous call with autocast=True casts it for you
```

`GenerationConfig` fields: `prompt`, `response_model` or `json_schema`, `temperature` (default 0), `max_tokens`, `detail` (`auto|lo|hi`), `confidence`, `grounding`, `video_segment_duration`. `PredictionResponse` carries `id`, `status` (`enqueued|pending|running|completed|failed|paused`), `response`, `usage`, `created_at`, `completed_at`.

CLI equivalent: `vlmrun generate -i invoice.pdf -d document.invoice [--schema s.json] [-p "..."] [--timeout S] [--format json]`, plus `vlmrun hub list`, `vlmrun hub schema <domain>`, `vlmrun predictions list|get <id>`, `vlmrun files list|upload|get|delete`.

### Agents and executions

Agents run as batch jobs on `vlmrun-orion-1` (`:lite|fast|auto|pro`). `batch=False` raises `NotImplementedError`.

```python
r = client.agent.execute(
    name="my-agent:v1",                              # or omit name and pass config.prompt
    inputs={"document": "https://example.com/invoice.pdf"},
    config=AgentExecutionConfig(prompt="Extract line items", json_schema={...}),
    toolsets=["core", "document"],                  # core, image, image-gen, world-gen, viz, document, video, web
    callback_url=None,
)
r = client.executions.wait(r.id, timeout=600, sleep=5)
r.status, r.response, r.usage
client.executions.list(skip=0, limit=10); client.executions.get(id)
client.agent.list(); client.agent.get(name=... | id=... | prompt=...); client.agent.create(...)
```

`client.agent.completions` is an OpenAI-compatible chat interface to the same agent (`vlmrun chat` uses it). Do not confuse it with `client.gateway.completions`.

CLI equivalent: `vlmrun execute -n my-agent:v1 -p "..." -i file [--schema s.json] [-t core -t document] [-m vlmrun-orion-1:auto] [--timeout S] [--callback-url U] [-f json]` (waits by default), `vlmrun executions list|get <id> [--wait --timeout S --poll-interval S]`.

### Errors

All SDK errors derive from `vlmrun.client.exceptions.VLMRunError`, each with a `suggestion`. Catch the specific one:

| Exception | Cause |
| --- | --- |
| `AuthenticationError` (401/403) | Key missing or wrong. `VLMRun()` raises it at construction. |
| `ValidationError` (400/422) | Bad domain, schema or input |
| `ResourceNotFoundError` (404) | Unknown prediction, execution, file or agent |
| `RateLimitError` (429) | Back off |
| `RequestTimeoutError`, `NetworkError`, `ServerError` (5xx) | Retry once, then report |
| `ConfigurationError`, `DependencyError`, `InputError` | Client side: missing key or `base_url`, missing optional package (`pip install "vlmrun[doc]"`, `[video]`, `[all]`), bad local input |

Gateway calls through `client.gateway` or the OpenAI SDK raise `openai.APIStatusError` subclasses instead, with the gateway's error body in `.body`.

## Re-checking this document

Flags come from `vlmrun <cmd> --help`. The catalog, capabilities, response envelopes and error
messages come from the live gateway. System One shapes come from `https://gateway.vlm.run/openapi.json`
(`/typesafe/*`) and `GET /typesafe/v1/models`. Client signatures come from `vlmrun/client/*.py` in the
SDK repo. To re-check:

```bash
VLMRUN_API_KEY=vlmrun uvx --from vlmrun vlmrun gw models --json
curl -sS https://gateway.vlm.run/typesafe/v1/models -H "Authorization: Bearer vlmrun"
uvx --from vlmrun vlmrun gw s1 --help
uvx --from vlmrun vlmrun --help
```
