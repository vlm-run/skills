---
name: vlmrun-gw
description: Call VLM Run Gateway models (gateway.vlm.run) with the `vlmrun gw` CLI, the OpenAI SDK, `VLMRun().gateway`, the MCP server, or raw HTTP. Use to OCR or parse a PDF, scan, receipt or form into markdown, text with boxes, layout or tables; ask questions about images, video or PDFs; detect, segment or estimate pose; embed images or text; transcribe speech; get typed, calibrated decisions (yes/no, label, score, box) with System One or `POST /decisions`; upload local files; and run VLM Run platform predictions and agent executions.
license: Apache-2.0
---

# VLM Run Gateway: `vlmrun gw` CLI, SDKs, MCP and HTTP

The Gateway serves OCR, document, vision, chat, embedding and transcription models behind one
OpenAI-compatible API, plus typed decisions (System One, Decisions). One model, one call, one
response, with the cost in `usage.cost`. Facts here were checked against Gateway 0.66.0 and
`vlmrun` 0.9.3. The live catalog wins over this file.

## Pick a surface

| You need | Use |
| --- | --- |
| A shell, local files, quick runs, loops | `vlmrun gw` CLI |
| Python code | OpenAI SDK at `https://gateway.vlm.run/openai`, or `VLMRun().gateway` |
| An agent with no shell, or a host that already lists `vlmrun` MCP tools | [MCP server](#mcp-server) at `https://gateway.vlm.run/mcp` |
| A label or probability, not text | [System One](#system-one-typed-decisions) (`vlmrun gw s1`), or [`POST /decisions`](#decisions) |
| Many frames from one camera or video | [Realtime WebSocket](#realtime-websocket) (no CLI) |
| A hub schema (`document.invoice`), your own Pydantic schema, or an agent job | [Platform API](#platform-api-predictions-and-agent-executions) (`VLMRun()`, `vlmrun generate`) |

## Base URLs

| Route | URL |
| --- | --- |
| OpenAI-compatible (chat, models, embeddings, transcriptions, files) | `https://gateway.vlm.run/openai` |
| System One (TypeSafe/Jev-compatible) | `https://gateway.vlm.run/typesafe` (`POST /v1/systemone`, `GET /v1/models`) |
| Decisions | `POST https://gateway.vlm.run/decisions` (alias `/v1/decisions`) |
| MCP | `https://gateway.vlm.run/mcp` |
| Realtime | `wss://gateway.vlm.run/v1/ws`, `wss://gateway.vlm.run/v1/ws/systemone` |
| Health | `GET https://gateway.vlm.run/health` (returns `version`) |

`/v1/openai` and `/v1/models` are aliases of `/openai` and `/openai/models`. They work, but new code
should use `/openai`. The SDK and CLI default to `VLMRUN_GATEWAY_URL=https://gateway.vlm.run/v1`,
append `/openai` (so they call the alias) and derive `/typesafe` from it. Override with
`VLMRUN_GATEWAY_BASE_URL` or `VLMRUN_GATEWAY_URL`. `VLMRUN_BASE_URL` is the platform API, not the Gateway.

## Install and auth

```bash
pip install -U "vlmrun[typesafe]"          # [typesafe] is needed for `vlmrun gw s1`; add [video] for s1 on video
uvx --from "vlmrun[typesafe]" vlmrun gw models   # no install
```

| Caller | Header | Rate limit |
| --- | --- | --- |
| Anonymous | `Authorization: Bearer vlmrun`, or no header on REST | 10/min, 30/hr, 100/day per client IP; $2.00 spend per IP per day |
| API key | `Authorization: Bearer $VLMRUN_API_KEY` | 240/min, 10,000/hr per user |

- **CLI anonymous:** set `VLMRUN_API_KEY=vlmrun`. With the variable unset, `vlmrun gw chat` fails with
  `Missing credentials`.
- **Model access:** most models are public. Some need an API key `(key)`, or an organization with a
  card on file `(paid)`. See [Models](#models). Refusals are `401`/`403` `model_not_entitled`, not billed.
- **Key only:** `POST /openai/files`, `POST /v1/ws/client_secrets`, `/v1/ws/systemone`, and `/decisions` (paid).
- **Anonymous rules:** a request with no `User-Agent` gets `429`. Replies carry `X-RateLimit-Limit`
  and `X-RateLimit-Remaining`.

Check a key without printing it. An empty variable falls back to anonymous and also returns 200, so test it first:

```bash
[ -n "$VLMRUN_API_KEY" ] && curl -sS -o /dev/null -w '%{http_code}\n' \
  https://gateway.vlm.run/openai/models -H "Authorization: Bearer $VLMRUN_API_KEY"
# 200: key works. 401: key is wrong.
```

No working key: follow [https://vlm.run/auth.md](https://vlm.run/auth.md) (the human must agree to the
terms), or copy one from [app.vlm.run](https://app.vlm.run/dashboard/settings/api-keys). Store it with
`vlmrun config set --api-key "$VLMRUN_API_KEY"`. If calls hit the wrong deployment, clear stale exports:
`unset VLMRUN_GATEWAY_URL VLMRUN_GATEWAY_BASE_URL VLMRUN_BASE_URL`.

## Commands

| Command | Does |
| --- | --- |
| `vlmrun gw models [<model>] [-j]` | Catalog, or one model's methods, notes and example commands |
| `vlmrun gw chat [<files/urls>...] -m <model> [-p <prompt>] [-M <method>] [--method-params JSON] [-e k=v]... [--json-mode \| --response-format F] [-ns] [-j] [--timeout S]` | OCR, parsing, vision models and chat over text, images, video, PDFs |
| `vlmrun gw embed [<files/urls>...] [-t <text>]... -m <model> [--join] [-d N] [-j] [--timeout S]` | Embeddings (`-p` is an alias of `-t`) |
| `vlmrun gw transcribe <file/url> \| --url <url> -m <model> [-f json\|text\|verbose_json\|srt\|vtt] [-l xx] [-p <hint>] [-j] [--timeout S]` | Speech to text (default `-f json`) |
| `vlmrun gw s1 [<inputs>...] [-s <state>] [-m <model>] [--noul ID="q"] [--choice ID="a\|b"] [--score ID="l0\|l1"] [-Q JSON] [--detail auto\|low\|high] [-R <effort>] [-g <gate>]... [-j] [--dry-run]` | System One decisions (alias `systemone`). `vlmrun gw s1 models` lists engines. |
| `vlmrun gw health` | Liveness probe |

`vlmrun gateway` is the same as `vlmrun gw`. `chat` sends every input in one request, sniffs each
file's type and picks `image_url`, `video_url` or `document_url`. It streams by default (`-ns` turns
that off) and prints model, tokens, pages and `usage.cost` in a footer.

- **`-e key=value`:** repeatable. Values parse as JSON when they can (`-e temperature=0`,
  `-e document_pages='[[0,10]]'`). Use it for OpenAI fields and for every Gateway kwarg.
- **`-j`:** prints `{"model", "content", "latency_s", "usage"}`; `usage.cost` is in it.
- **`--json-mode -j`:** prints the model's parsed JSON object, without `usage`.
- **Errors:** the CLI shows only `Error code: 400` for a Gateway error. Re-run the same request with
  `curl` or the OpenAI SDK to read the message.

## Models

Read the catalog before you pick. It has 30 public models at 0.66.0.

```bash
vlmrun gw models                         # id, task, inputs, methods (default marked *)
vlmrun gw models paddleocr/pp-ocrv6      # methods, method_params examples, realtime notes
curl -sS https://gateway.vlm.run/openai/models -H "Authorization: Bearer vlmrun"   # raw: pricing, capabilities, supported_parameters
```

Address a model by its full id (`zai-org/glm-ocr`), its short form (`glm-ocr`) or its Hugging Face id
(`zai-org/GLM-OCR`). The reply's `model` is the canonical id. Per-model prices, inputs and
`method_params` are on the [catalog](https://vlm.run/gateway/models) and in `pricing` on `GET /openai/models`.

### By task

Access: public unless marked `(key)` or `(paid)`.

| Task | Model, method | What you get |
| --- | --- | --- |
| PDF or scan to markdown | `zai-org/glm-ocr` `markdown` | Markdown per page |
| | `vlm-run/ocr:auto` `markdown` | Markdown from a CPU parser (falls back to `dots.mocr`). Bills $0.001 per call at any page count. |
| | `rednote-hilab/dots.mocr`, `paddlepaddle/paddleocr-vl-1.6`, `lightonai/lightonocr-3-4b`, `nvidia/nemotron-3.5-parse` `markdown` | Markdown per page. `dots.mocr` is the MCP `read_document` default. |
| Text lines with boxes and scores | `paddleocr/pp-ocrv6` `ocr` (default) | `doc.ocr.lines`. `method_params`: `score_threshold`, `word_boxes`. `text` returns plain text, `detect` boxes only. |
| Layout blocks (label + box) | `nvidia/nemotron-3.5-parse` `layout`, `lightonai/lightonocr-3-4b` `grounding` (with text) or `layout`, `rednote-hilab/dots.mocr` `parse_layout` (with text) or `parse_layout_only`, `vlm-run/ocr:auto` `layout` | `doc.page.blocks` |
| Tables, formulas, charts | `paddlepaddle/paddleocr-vl-1.6` `table`, `formula`, `chart` | `table` returns OTSL cell tokens (`<fcel>`, `<nl>`) |
| Captions, open-vocabulary boxes | `microsoft/florence-2-base-ft` `caption` (default), `detailed_caption`, `more_detailed_caption`, `od`, `dense_region_caption`, `region_proposal`, `ocr`, `ocr_with_region` | Text, or `img.detect.bboxes` |
| COCO object detection | `roboflow/rfdetr-large` `detect` | `img.detect.bboxes` |
| Segmentation | `facebook/sam3.1` `segment` (text prompt), `segment_box`, `segment_point`, `track` (video) | `img.segment.masks`, `vid.segment.masks` |
| | `roboflow/rfdetr-seg-large` `segment` (COCO classes), `detect` | `img.segment.masks` |
| Body pose (image or video) | `usyd-community/vitpose-plus-large` `pose` | `img.pose.kpts` (COCO 17 keypoints), `vid.pose.kpts` |
| Hand pose, 3D | `geopavlakos/hamer` `pose` | `img.pose.hands3d` |
| Chat VLM, public | `qwen/qwen3.5-0.8b`, `google/diffusiongemma-26b-a4b-it` | Free text. Text, images, PDF; video on qwen only. |
| Chat VLM, larger | `qwen/qwen3.8-27b` (key) | Reasoning, tools, 168k context, up to 64 images or 1 video |
| | `google/gemma4-26b-a4b-it` (paid) | Tools, up to 64 images or 1 video |
| Provider chat models (paid) | `google/gemini-3.5-flash-lite`, `gemini-3.7-flash`, `gemini-3.8-flash`, `gemini-4-argon`, `gemini-4-carbon`, `gemini-robotics-er-2-preview`, `moonshotai/kimi-k3`, `minimax/minimax-m3`, `meta/muse-spark-1.2`, `meta/muse-glimmer-30b` | Text, images, video, whole PDF. `muse-glimmer-30b` has no `json_schema`. |
| Embeddings | `qwen/qwen3-vl-embedding-2b` | 2048 dims; text, image, video |
| | `google/embeddinggemma-2` | 768 dims (truncate to 512, 256, 128); text, image, video, PDF |
| Transcription | `nvidia/parakeet-tdt-0.6b-v3` | Transcript, segments, subtitles |
| Typed decisions | System One: `google/diffusiongemma-26b-a4b-it` (default) | See [System One](#system-one-typed-decisions) |
| | Decisions: `google/gemini-decision-1` (paid) | See [Decisions](#decisions) |

Rules of thumb:

- Printed text goes to an OCR or document model, not a chat VLM. OCR models reject text-only prompts.
- Use a chat VLM for questions about a scene, a video, or a whole PDF. Plain prompt, no `--method`.
  The self-hosted chat VLMs also have `markdown` and `layout` methods, which return a `<document>`
  envelope like an OCR model.
- OCR, detection, segmentation and pose models take one image (or one PDF) per request. A second
  `image_url` is a `400`. Send several images to a chat VLM, or loop.
- `qwen/qwen3.8-27b` reasons by default, which spends completion tokens. Turn it off with
  `-e chat_template_kwargs='{"enable_thinking": false}'`.

## Documents

The Gateway rasterises a PDF server-side and runs the model per page (chat VLMs on `chat` read the
whole PDF in one call). Do not split pages yourself.

```bash
vlmrun gw chat report.pdf -m zai-org/glm-ocr                                   # markdown per page
vlmrun gw chat report.pdf -m vlm-run/ocr:auto                                  # markdown, $0.001 per call
vlmrun gw chat scan.png -m paddleocr/pp-ocrv6 --method ocr                     # lines with boxes and scores
vlmrun gw chat sheet.pdf -m rednote-hilab/dots.mocr --method parse_layout      # layout blocks with text
vlmrun gw chat statement.pdf -m paddlepaddle/paddleocr-vl-1.6 --method table
vlmrun gw chat book.pdf -m zai-org/glm-ocr -e document_pages='[[10,20]]' -e document_dpi=150
```

Text-mode envelope, one `<document>` per PDF, one `<page>` per page:

```text
<document mimetype="application/pdf" npages="1" dpi="96">
<page id="0" format="markdown" width="816" height="1056">
| Title of each class | Trading Symbol(s) | ...
</page>
</document>
```

- `<document>` attributes: `file_name` (omitted for `data:` URIs, so for CLI local files), `mimetype`,
  `npages`, `dpi`, and `language` when the model reports one.
- `<page>` attributes: `id` (0-based), `format` (`markdown`, or `json` for region methods), `width`, `height`, `status`.
- A failed page is self-closing with `status="error"`. Numbering stays intact. If every page fails, the call is a `502`.
- `document_pages` renumbers the selected pages from 0.

| Kwarg | Default | Meaning |
| --- | --- | --- |
| `document_dpi` | `96` | Raster DPI per page (72 to 400). Raise for fine print. |
| `document_pages` | all | 0-based indices and `[start, stop)` ranges, e.g. `[0, [2, 4]]` |
| `document_max_pages` | `128` | Page cap. A longer PDF is a `400` that names the ranges to send. Read it in sections. |
| `document_mode` | method's mode | Chat VLMs: `document` (whole PDF, default on `chat`) or `page` (default on `markdown`, `layout`) |

Limits: one `document_url` (or `file_url`, `file`) part per request; 256 MB; no images or video in the
same request. Prefer an `http(s)` URL over base64. For a local file, see [Files](#files).

## Chat over text, images and video

```bash
vlmrun gw chat -m qwen/qwen3.5-0.8b -p "In one sentence: what is OCR?"                     # text only
vlmrun gw chat receipt.jpg -m qwen/qwen3.5-0.8b -p "What store is this from?"
vlmrun gw chat a.jpg b.jpg -m qwen/qwen3.8-27b -p "What changed between these?"           # multi-image (key)
vlmrun gw chat clip.mp4 -m qwen/qwen3.5-0.8b -p "Describe what happens" -e video_fps=1 -e video_max_frames=8
vlmrun gw chat photo.jpg -m google/gemini-3.5-flash-lite -p "Describe it"                 # paid
vlmrun gw chat receipt.jpg -m qwen/qwen3.5-0.8b -p "Reply as JSON with keys store,total" --json-mode
vlmrun gw chat receipt.jpg -m qwen/qwen3.5-0.8b -p "Extract store and total" \
  --response-format '{"type":"json_schema","json_schema":{"name":"r","schema":{"type":"object","properties":{"store":{"type":"string"},"total":{"type":"number"}},"required":["store","total"]}}}'
```

- Chat VLM output passes through verbatim, with no envelope, in every `response_format`.
- `json_schema` works where `capabilities.supports_json_schema` is true: Qwen, `gemma4` and provider
  models, not `google/diffusiongemma-26b-a4b-it` or `meta/muse-glimmer-30b`. The CLI help says it is
  "not yet honored server-side"; that is wrong.
- Video: one clip per request. Gateway knobs: `video_fps`, `video_max_frames`, `video_resolution`
  (`256x192` to `640x480`). Images: `image_resolution` (`224x224` to `768x768`).
- `video_fps` with no video part is a `400`. On `minimax/minimax-m3` and `meta/muse-spark-1.2` it is
  a `400` `upstream_error` (their provider has no frame sampling). Other models accept it.
- Sampling fields are checked against the model's `supported_parameters`. An unlisted one is a `400`
  (`google/diffusiongemma-26b-a4b-it` refuses `temperature` and `seed`). OCR models accept and ignore them.

## Detection, segmentation and pose

These return records, not prose. Use `--json-mode` (`response_format={"type": "json_object"}` in code) when code parses them.

```bash
vlmrun gw chat street.jpg -m roboflow/rfdetr-large --json-mode -j                        # img.detect.bboxes
vlmrun gw chat street.jpg -m facebook/sam3.1 -p "hat" --method-params '{"mask_format":"none"}'   # masks for a text prompt
vlmrun gw chat street.jpg -m facebook/sam3.1 --method segment_box --method-params '{"bbox_xywh":[0.1,0.1,0.4,0.5]}'
vlmrun gw chat crowd.jpg -m usyd-community/vitpose-plus-large --json-mode -j               # img.pose.kpts
vlmrun gw chat hands.jpg -m geopavlakos/hamer --json-mode -j                               # img.pose.hands3d
vlmrun gw chat photo.jpg -m microsoft/florence-2-base-ft --method detailed_caption
```

`facebook/sam3.1` takes its target from `-p` or from `method_params.prompt`. Keys per model are in
[Extra Kwargs](https://docs.vlm.run/gateway/extra-kwargs#method-params-by-model).

## Embeddings

```bash
vlmrun gw embed photo.jpg -m qwen/qwen3-vl-embedding-2b                         # 1 vector, 2048 dims
vlmrun gw embed a.jpg b.jpg -t "a caption" -m qwen/qwen3-vl-embedding-2b       # 3 vectors
vlmrun gw embed photo.jpg -t "its caption" --join -m qwen/qwen3-vl-embedding-2b   # 1 fused vector
vlmrun gw embed -t "a blue parrot" -m qwen/qwen3-vl-embedding-2b -d 64
vlmrun gw embed photo.jpg -m google/embeddinggemma-2 --json                    # full vectors and usage.cost
```

Every file and every `-t` is its own vector unless `--join` fuses them (one file at most). The CLI
refuses a PDF on its side. `google/embeddinggemma-2` embeds a PDF over HTTP:
`input=[[{"type": "document_url", "document_url": {"url": ...}}]]`. Report vector count and
dimension, never raw floats.

## Transcription

```bash
vlmrun gw transcribe clip.mp3 -m nvidia/parakeet-tdt-0.6b-v3
vlmrun gw transcribe meeting.mp4 -m nvidia/parakeet-tdt-0.6b-v3            # a video's audio track
vlmrun gw transcribe clip.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -f srt        # subtitles
vlmrun gw transcribe clip.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -f verbose_json -j   # segments with timestamps
vlmrun gw transcribe --url https://example.com/a.mp3 -m nvidia/parakeet-tdt-0.6b-v3 -l en
```

- `-l` is a language hint. The model lists `language`, `response_format` and
  `timestamp_granularities`, not `prompt`, so `-p` is sent but has no listed effect.
- Audio with no speech returns an empty transcript, not an error.
- `json` and `verbose_json` carry `usage` (`input_audio_seconds`, `cost`). `text`, `srt` and `vtt` return the transcript only.

## Response shapes

| Mode | Request | Reply |
| --- | --- | --- |
| Text (default) | omit `response_format` | One block per request: a bare payload for an image, a `<document>` envelope for a PDF, verbatim text for a chat VLM |
| JSON | `--json-mode` / `{"type": "json_object"}` | One object: `model`, `method`, input metadata, then `content` (image, video) or `pages` (PDF). No `data` list. Buffered, even when streaming. |
| JSON schema | `--response-format '{"type":"json_schema",...}'` | Chat VLMs and provider models only. OCR, layout and region models return `400` `capability_violation` on `response_format`. |

JSON mode input keys: image `image_hash`, `image_width`, `image_height`; video `video_hash`,
`video_width`, `video_height`, `video_fps`, `video_nframes`, `video_duration`; PDF `file_name`,
`document_mimetype`, `document_npages`, `document_dpi`, `document_language`, then `pages[]` of
`{"object": "doc.page", "page_id", "page_width", "page_height", "content"}`.

Parse by `content.object`:

| Tag | From |
| --- | --- |
| `doc.ocr.lines` | `paddleocr/pp-ocrv6` `ocr`: `bbox_xywh`, `poly_xy`, `text`, `score` |
| `doc.page.blocks` | layout methods: `block_id`, `bbox_xywh`, `label`, `text` (when read) |
| `img.detect.bboxes` | `rfdetr-large`, `florence-2` `od`: `bbox_xywh`, `label`, `score` |
| `img.segment.masks` / `vid.segment.masks` | `sam3.1`, `rfdetr-seg-large`: items plus a label `mask` |
| `img.pose.kpts` / `vid.pose.kpts` | `vitpose-plus-large`: `kpts_xy`, `kpts_score`, container `kpts_labels` |
| `img.pose.hands3d` | `hamer`: `kpts_xy`, `kpts_xyz`, `pos_xyz`, `side` |

Coordinates are normalised 0 to 1 (`bbox_xywh` is `[x, y, w, h]`). Multiply by the image width and
height for pixels. `precision` (1 to 8, default 4) sets the decimals. Responses also carry
`served_model_id`, `backend` and `provider`, and an `x-request-id` header. Full types:
[Response Types](https://docs.vlm.run/gateway/response-types).

## Request knobs

Put Gateway kwargs at the top level of the JSON body, in `extra_body` on the OpenAI Python SDK, or in
`-e` on the CLI. `--method` and `--method-params` are CLI shorthands.

| Kwarg | Default | Applies to |
| --- | --- | --- |
| `method`, `method_params` | model's `default_method` | Which entrypoint runs, and its arguments |
| `document_dpi`, `document_pages`, `document_max_pages`, `document_mode` | `96`, all, `128`, by method | PDFs |
| `video_fps`, `video_max_frames`, `video_resolution` | model's own | `video_url` |
| `image_resolution` | none | Images |
| `precision` | `4` | Floats in structured payloads |
| `chat_template_kwargs` | none | Self-hosted chat models, e.g. `{"enable_thinking": false}` |

A method the model does not list is a `400` naming the supported ones. An unknown top-level key or
`method_params` key is ignored, so check spelling. Full list:
[Extra Kwargs](https://docs.vlm.run/gateway/extra-kwargs).

## Files

`POST /openai/files` uploads a local file once and returns a signed `url` (valid 7 days) to pass in
any `*_url` part. It needs an API key (anonymous is `401`). The upload is free; the read bills as usual.

```bash
curl -sS https://gateway.vlm.run/openai/files -H "Authorization: Bearer $VLMRUN_API_KEY" -F file=@scan.pdf
# {"id": "...", "object": "file", "bytes": ..., "filename": "...", "purpose": "assistants", "url": "https://storage.googleapis.com/..."}
curl -sS https://gateway.vlm.run/openai/files/<id> -H "Authorization: Bearer $VLMRUN_API_KEY"   # fresh url
```

In Python: `client.files.create(file=open("scan.pdf", "rb"), purpose="assistants").url`. A `.txt` file is a `415`.

## Python

### OpenAI SDK

```python
import base64, os, pathlib
from openai import OpenAI

client = OpenAI(base_url="https://gateway.vlm.run/openai", api_key=os.environ["VLMRUN_API_KEY"], timeout=700)
pdf = base64.b64encode(pathlib.Path("report.pdf").read_bytes()).decode()
r = client.chat.completions.create(
    model="zai-org/glm-ocr",
    messages=[{"role": "user", "content": [
        {"type": "document_url", "document_url": {"url": f"data:application/pdf;base64,{pdf}"}},
    ]}],
    extra_body={"method": "markdown", "document_pages": [[0, 10]]},
)
print(r.choices[0].message.content, r.usage.model_extra["cost"])

client.embeddings.create(model="qwen/qwen3-vl-embedding-2b", input=["a blue parrot"], dimensions=64)
client.audio.transcriptions.create(model="nvidia/parakeet-tdt-0.6b-v3", file=open("clip.mp3", "rb"), response_format="srt")
client.models.list()   # extra fields (methods, pricing, capabilities) on model.model_extra
```

- Content parts: `text`, `image_url`, `video_url`, `document_url` (alias `file_url`), `file`. URLs are `http(s)` or `data:` URIs.
- Streaming: `usage.cost` arrives on the final chunk.
- Errors raise `openai.APIStatusError`; the Gateway body is in `.body`.
- Keep the client timeout above the 600 s model deadline.

### `VLMRun().gateway`

The same OpenAI client, pre-wired. It reads `VLMRUN_API_KEY` and the gateway URL, and raises the
default timeout from 120 s to 600 s.

```python
from vlmrun.client import VLMRun

gw = VLMRun().gateway        # VLMRun() needs a key; VLMRUN_API_KEY=vlmrun works for anonymous use
gw.completions.create(model="glm-ocr", messages=[...], extra_body={...})   # gw.async_completions for asyncio
gw.embeddings.create(model="qwen/qwen3-vl-embedding-2b", input=["text"])
gw.transcriptions.create(model="nvidia/parakeet-tdt-0.6b-v3", file=open("clip.mp3", "rb"))
gw.models()                  # list of OpenAI Model objects
gw.health()                  # bool
gw.systemone.decide(state="I was charged twice", questions=[{"id": "billing", "type": "noul", "instructions": "Is this a billing issue?"}])
```

Raw HTTP works too: `POST https://gateway.vlm.run/openai/chat/completions`, kwargs at the top level next to `model` and `messages`.

## System One (typed decisions)

System One answers typed questions about a **state** (text, a JSON object or array, images, or one
PDF) and returns calibrated probabilities. Nothing is generated and parsed, so an answer cannot be
off-schema. Use it when the answer is a label or a probability. Use `chat` for free text.

`POST https://gateway.vlm.run/typesafe/v1/systemone` is wire-compatible with TypeSafe's Jev API. Same
bearer as the rest of the Gateway; anonymous `vlmrun` works on the public engines.

| Engine (`GET /typesafe/v1/models`) | Read | Access | Note |
| --- | --- | --- | --- |
| `google/diffusiongemma-26b-a4b-it` | diffusion | public | CLI default. Docs: prefer it when you threshold on probabilities. |
| `google/gemma4-26b-a4b-it` | generative | paid | The only engine that takes `reasoning_effort` |
| `qwen/qwen3.8-27b` | generative | key | |
| `qwen/qwen3.5-0.8b` | generative | public | Lowest cost per read; less peaked answers |

`model` is required; there is no `jev-latest` alias (`404`). Generative calibration is experimental:
validate thresholds on your own data, and again when you change engines.

### Questions

Up to 16 per request, keyed by your ids (ids are not sent to the model).

| `type` | Shape | Answer |
| --- | --- | --- |
| `noul` | yes/no; optional `criteria.true` / `criteria.false` | `noul`: P(yes) |
| `choice` | `criteria`: 2 to 128 labels, each a description or `null` | `choice`, `probabilities`, `confidence` |
| `score` | `criteria`: ordered list of 2 to 10 levels | `score` (expected level, 0-based, fractional), `legend`, `probabilities`, `confidence` |
| `box` | `instructions` names the target; needs an image; 8 per request | `box` `{top, bottom, left, right}` (0 to 1), `confidence`. Docs call it a preview. |
| `text`, `number`, `date`, `list` | a field to extract | `value` (typed or `null`), `confidence`, `p_null` |

Any question can return `{"type": "refusal"}`; the other answers still hold. `confidence` runs from 1
(all mass on one label) to 0 (uniform).

### Inputs and extensions

| Field | Notes |
| --- | --- |
| `content` | OpenAI-style parts: `image_url` (8 max, 5 MB each, `data:` or `http(s)`), `file` / `document_url` (1 PDF, first 16 pages at 96 DPI), `text`. Images or one PDF, never both (`422`). |
| `assets` | TypeSafe image format, keyed by a reference written into `state` (the `typesafe-sdk` `Image` type) |
| `detail` (per part) | `auto` or `high`: 1,072 vision tokens. `low`: 70. One `high` part lifts the whole read. |
| `reasoning_effort` | `none` (default), `minimal`, `low`, `medium`, `high`. `google/gemma4-26b-a4b-it` only; other engines `422`. |
| `steps` | Diffusion only, 1 to 8 (default 1) |
| `samples` | 1 to 32 noise draws; each draw is billed |
| `document_pages`, `document_max_pages` (1 to 16), `document_dpi` | PDF selection |

Unknown top-level fields are a `422`. Response: `{"id", "model", "answers": {...}, "usage": {"input_tokens", "output_tokens", "input_tokens_details", "reads", "cost"}}`.

### CLI: `vlmrun gw s1`

The CLI asks `noul`, `choice` and `score`. For `box` and field types, use HTTP or `typesafe-sdk`.
Give every `--noul` question text: a bare `--noul ID` fails in the CLI before it sends.

```bash
# Image, three questions, one read
vlmrun gw s1 -s "Answer about the attached page." receipt.jpg --detail high \
  --choice kind="invoice|receipt|contract|other" \
  --noul has_total="Is a total amount due visible on the page?" \
  --score legibility="unreadable|partly legible|clear"

# PDF routing; --detail low for coarse classification
vlmrun gw s1 -s "Answer about the document." packet.pdf --detail low \
  --choice kind="invoice|contract|report|other" --noul has_signature="Is a signature block visible?"

# Text state, a generative engine with reasoning (paid)
vlmrun gw s1 "I was charged twice for invoice 44" -m google/gemma4-26b-a4b-it -R low \
  --choice dept="billing|tax|technical"

# Gate a script: exit 0 pass, 1 a gate failed, 2 the request failed
vlmrun gw s1 -s "Answer about the page." scan.jpg --noul is_invoice="Is this an invoice?" \
  --gate 'is_invoice>0.8' && vlmrun gw chat scan.jpg -m zai-org/glm-ocr
```

- `-s` takes text, `@file` or `-`. Without `-s`, `.txt`/`.md`/`.json`/`.yaml` positionals become the state.
- `-Q` takes the wire map or a list form, inline, `@file.json` or `-`. `-B/--body` sends a full body.
- `-j` prints the response verbatim (with `usage.cost`) on stdout; timings go to stderr.
- `--dry-run` prints the body without sending. `--repeat N` measures spread across N reads.
- Video (local file, needs `[video]`): sampled client-side, one read per frame (`--fps`, default 1;
  `--max-frames`, default 60). `--gate-mode any|all|mean|sustained:N`. `--ws` reads over one socket.
- The CLI help is wrong on two points: it lists Qwen engines as able to reason (only gemma4 can), and says
  remote images must be inlined (the route accepts `http(s)` image URLs).

### Python (`typesafe-sdk`)

```python
import base64, os, pathlib
from typesafe_sdk import Box, Choice, Image, Noul, Score, TypeSafeClient

client = TypeSafeClient(api_key=os.environ["VLMRUN_API_KEY"], base_url="https://gateway.vlm.run/typesafe", compression=False)
page = base64.b64encode(pathlib.Path("receipt.jpg").read_bytes()).decode()
result = client.system_one(
    "Answer about the attached page.",
    {
        "kind": Choice(instructions="What kind of document is this?", criteria={"invoice": None, "receipt": None, "other": None}),
        "has_total": Noul(instructions="Is a total amount due visible?"),
        "legibility": Score(instructions="How legible is the text?", criteria=["unreadable", "partly legible", "clear"]),
    },
    extra_body={"content": [{"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{page}", "detail": "low"}}]},
    model="google/diffusiongemma-26b-a4b-it",
)
print(result.choices["kind"].choice, result.nouls["has_total"].noul, result.scores["legibility"].score)

box = client.system_one(state={"receipt": Image(pathlib.Path("receipt.jpg"))},
                        questions={"total": Box(instructions="The TOTAL line")}, model="qwen/qwen3.5-0.8b")
print(box.boxes["total"].box, box.boxes["total"].confidence)
```

- `pip install typesafe-sdk` (included in `vlmrun[typesafe]`). Env vars: `TYPESAFE_BASE_URL`,
  `TYPESAFE_API_KEY`, `TYPESAFE_DEFAULT_MODEL` (set it; the SDK's `jev-latest` default is a `404` here).
- `compression=False`: the Gateway does not decode zstd bodies. The SDK sends zstd by default on Python 3.14+.
- `typesafe-sdk` and `VLMRun().gateway.systemone` drop `usage.cost`. Read cost from raw HTTP or `vlmrun gw s1 -j`.
- Field questions (`text`, `number`, `date`, `list`): send them with any HTTP client; the SDK does not decode them.
- Never point `TYPESAFE_BASE_URL` at `/openai`.

### System One errors

| Status | Body | Fix |
| --- | --- | --- |
| `422` | `{"detail": [{"type", "loc", "msg"}]}` | Fix the field in `loc`: unknown field, images plus PDF, `reasoning_effort` on a non-gemma4 engine |
| `401` | `authentication_error` | Anonymous on a key or paid engine |
| `403` | `permission_error` | No card on file for a paid engine |
| `404` | `not_found_error` | `model` not served here; read `GET /typesafe/v1/models` |
| `429` | Gateway rate-limit body | Same buckets as the OpenAI routes |
| `502` | `api_error` | Engine failed; not billed; retry |
| `503` / `529` | `api_error` / `overloaded_error` | Retry after `retry-after` |

Docs: [System One](https://docs.vlm.run/gateway/system-one), [TypeSafe models](https://docs.vlm.run/gateway/typesafe-models).

## Decisions

`POST https://gateway.vlm.run/decisions` asks System One questions of `google/gemini-decision-1`, a
provider model. Paid accounts only. It is not in `GET /openai/models` and does not answer chat or
`/typesafe`. No CLI command; use HTTP.

```bash
curl -sS https://gateway.vlm.run/decisions -H "Authorization: Bearer $VLMRUN_API_KEY" -H "Content-Type: application/json" -d '{
  "model": "google/gemini-decision-1",
  "state": "Answer about the attached image.",
  "content": [{"type": "image_url", "image_url": {"url": "https://storage.googleapis.com/vlm-data-public-prod/hub/examples/document.receipt/playground/2.jpg"}}],
  "questions": {
    "kind": {"type": "choice", "instructions": "What kind of document is this?", "criteria": {"invoice": null, "receipt": null, "other": null}},
    "has_total": {"type": "noul", "instructions": "Is a total amount visible?"}
  }}'
```

- Same response shape as System One (`answers`, `usage.reads` = 1, `usage.cost`).
- Every question needs `instructions`; a missing one is a `400` that names it.
- `noul`, `choice`, `score` work. `box` is a `400` from the provider today; field types are a `400`.
  `reasoning_effort` other than `none`, `steps` and `samples` are a `400`.
- JSON body only. Media: `content` image parts, PDF pages, or `assets`.
- Error envelope: `{"error": {"code": 400, "message", "type", "param"}}`. `401` `authentication_error`,
  `403` `permission_error`, `404` `not_found_error`, `502`/`503` `api_error`.

## MCP server

`https://gateway.vlm.run/mcp`: Streamable HTTP, stateless, server name `vlmrun`. Same billing and
limits as REST.

```bash
claude mcp add --transport http vlmrun https://gateway.vlm.run/mcp --header "Authorization: Bearer $VLMRUN_API_KEY"
```

| Tool | Reads | Default model | Key arguments |
| --- | --- | --- | --- |
| `read_document` | PDF or page image to markdown or JSON | `rednote-hilab/dots.mocr` | `url`, `model`, `dpi`, `pages`, `method`, `method_params`, `json_mode` |
| `read_image` | One image: answer, or region records | `qwen/qwen3.5-0.8b` | `url`, `model`, `prompt`, `method`, `method_params`, `json_mode` |
| `read_audio` | Speech | `nvidia/parakeet-tdt-0.6b-v3` | `url`, `model`, `language`, `json_mode`, `timestamps` |
| `read_video` | One clip | `qwen/qwen3.8-27b` (key) | `url`, `model`, `prompt`, `fps`, `max_frames`, `method`, `method_params`, `json_mode` |
| `list_models`, `get_model_info`, `get_completion` | Model menus, reply schemas, a prior read's cost | | |
| `create_upload_token` | Single-use token (10 min) for one `POST /openai/files` upload | | none |

- **Auth:** `Bearer $VLMRUN_API_KEY`, or `Bearer vlmrun` for anonymous. No header is a `401` on
  `/mcp` (unlike REST). OAuth is available where the deployment enables it.
- **`url`:** `http(s)`, `data:` URI or base64, never a local path. Upload local files first.
- **Model menus:** OCR models, the self-hosted chat VLMs and the region models. No provider models,
  no embeddings, no System One.
- **Region records:** set `json_mode: true`.

Use MCP when the agent has no shell or already has the tools. Use the CLI for local files, loops,
embeddings, System One and provider models. Docs: [MCP](https://docs.vlm.run/gateway/mcp-server).

## Realtime WebSocket

For a stream of frames with one model and method. The CLI does not drive it, except `vlmrun gw s1 clip.mp4 --ws`.

- **Sockets:** `wss://gateway.vlm.run/v1/ws` returns the model's reply per frame
  (`vitpose-plus-large`, `hamer`, `sam3.1`, `pp-ocrv6`, `rfdetr-large`, and others that opt in).
  `wss://gateway.vlm.run/v1/ws/systemone` returns a System One decision per frame (API key only).
- **Flow:** open with `Authorization: Bearer <key>`, send `session.create` JSON
  (`{"type": "session.create", "model": "roboflow/rfdetr-large", "method": "detect"}`), wait for
  `session.created`, send each frame as one binary JPEG/PNG/WebP/GIF message, end with
  `session.close`, read `session.stats` (counts and `cost`).
- **Browser:** mint a client secret on your server with `POST /v1/ws/client_secrets` (API key), then
  offer subprotocols `["vlmrun.realtime", "vlmrun.bearer.<secret>"]`.
- **Anonymous `vlmrun` on `/v1/ws`:** 1 session per IP, 120 s open time per model per day.
- **Bounds:** a session lives 10 min, closes after 30 s idle, and caps spend at $5.00. Frames
  over 2 MiB or over the model's `max_fps` are dropped.

Docs: [Realtime](https://docs.vlm.run/gateway/realtime).

## Cost and limits

- **`usage.cost`:** USD per call, on the response; on the final SSE chunk when streaming. Also on
  embeddings, transcription (`json`, `verbose_json`), System One, Decisions and MCP reads.
- **Before a call:** `pricing` on `GET /openai/models/{id}` gives per-model rates (per 1M tokens,
  images, video seconds or audio seconds).
- **Minimum per call** ([Pricing](https://docs.vlm.run/gateway/pricing)):

| Models | Minimum per call |
| --- | --- |
| OCR, layout, detection, captioning, segmentation, pose, transcription | $0.001 |
| `google/embeddinggemma-2` | $0.000005 |
| Chat VLMs, provider models, `qwen/qwen3-vl-embedding-2b`, System One, Decisions | none |

- **Deadlines:** 30 s to first token (60 s on `qwen/qwen3.8-27b`, 120 s on provider models) and 600 s
  total. Past either is a `504`.
- **Cold model:** a model with no warm replica returns `503` `model_unavailable` with `Retry-After`.
  Back off and retry.
- **Rate limits:** see [Install and auth](#install-and-auth). A `429` carries `Retry-After`; wait that long once.

## Errors

OpenAI routes nest the error in `detail`:
`{"detail": {"error": {"message", "type", "code", "param"}}}`. Rate limits and a Gateway fault use a
top-level `error`. Every response has `x-request-id`; send it to support@vlm.run with the body.

| Status | `type` / `code` | Example message | Fix |
| --- | --- | --- | --- |
| 400 | `invalid_request` | `messages: Field required` | Fix the field named in `param` |
| 400 | `capability_violation`, `param: messages` | `Model 'zai-org/glm-ocr' requires at least one image_url, video_url, or document_url content part.` | Attach a file; text-only needs a chat VLM |
| 400 | `capability_violation`, `param: method` | `Model 'zai-org/glm-ocr' does not support method 'parse_layout'; supported: markdown.` | Use a listed method |
| 400 | `capability_violation`, `param: messages.content.image_url` | `Model 'paddleocr/pp-ocrv6' accepts at most 1 image_url part(s); got 2.` | One image per call, or a chat VLM |
| 400 | `capability_violation`, `param: response_format` | `... does not support response_format {'type': 'json_schema'} ...` | Use `--json-mode` |
| 400 | `capability_violation`, `param: <field>` | `Model '...' does not accept 'temperature'.` | Drop the field |
| 400 | `invalid_document` | `This PDF has 5 pages and one request reads at most 2. ... [0, 2], [2, 4], [4, 5]` | Send the named `document_pages` ranges |
| 400 | `invalid_media` | Fetch or decode failure | Check the URL, type and bytes |
| 400 | `upstream_error` | `... exposes no frame-sampling control, so 'video_fps' cannot be honoured.` | Drop `video_fps`, or re-encode the clip |
| 401 | `"Invalid API Key"` | | Wrong key; check for stale exports |
| 401 / 403 | `model_not_entitled` | `requires an API key` / `requires a credit card on file` | Use a key, or a paid org |
| 402 | plain detail | `Your credits are exhausted.` | Top up billing |
| 404 | `model_not_found` | Lists every served id | Copy the right id |
| 429 | `rate_limit_exceeded`, `usage_limit_exceeded` | | Wait `Retry-After`, or use a key |
| 502 | `model_error`, `document_inference_failed` | | Not billed; retry with backoff |
| 503 | `model_unavailable`, `model_busy` | | Retry after `Retry-After` |
| 504 | `ttft_timeout`, `inference_timeout` | | Retry; smaller input; `stream=true` |

Retry once after you re-read the catalog. Do not loop. If the second attempt fails, report the exact
command and response. Full table: [Error Codes](https://docs.vlm.run/gateway/error-codes).

## How to run a request well

1. **Read the ask:** inputs (expand `~`; `http(s)` URLs are fine) and intent. Ask one short question
   only if the intent is ambiguous. A missing model is not ambiguity.
2. **Pick from the catalog:** `vlmrun gw models`, then `vlmrun gw models <model>`. Honour any model or
   method the user named. Check its access tier against the key you have.
3. **Show the command** on one line, then run it. Add `-j` only for scripting.
4. **Let the CLI encode** local files. For MCP or raw HTTP, upload them with [Files](#files).
5. **Report** the text, answer, transcript, labels with probabilities, or vector count and dimension,
   plus `usage.cost` summed across a batch. If you chose the model or method, say which and why in one line.

## Worked examples

| Ask | Command |
| --- | --- |
| "parse this contract to markdown" | `vlmrun gw chat contract.pdf -m zai-org/glm-ocr` |
| "OCR a long PDF at the lowest cost" | `vlmrun gw chat book.pdf -m vlm-run/ocr:auto` (over 128 pages: `-e document_pages='[[0,128]]'`, then the next range) |
| "extract the text and boxes from this receipt" | `vlmrun gw chat receipt.png -m paddleocr/pp-ocrv6 --method ocr` |
| "just the text, no boxes" | `vlmrun gw chat receipt.png -m paddleocr/pp-ocrv6 --method text` |
| "get the layout of this page" | `vlmrun gw chat page.pdf -m nvidia/nemotron-3.5-parse --method layout --json-mode -j` |
| "pull the tables out" | `vlmrun gw chat statement.pdf -m paddlepaddle/paddleocr-vl-1.6 --method table` |
| "read pages 10 to 20" | `vlmrun gw chat book.pdf -m zai-org/glm-ocr -e document_pages='[[10,20]]'` |
| "read this fine print" | `vlmrun gw chat scan.pdf -m zai-org/glm-ocr -e document_dpi=300` |
| "caption this photo" | `vlmrun gw chat photo.jpg -m microsoft/florence-2-base-ft --method detailed_caption` |
| "what's in this image?" | `vlmrun gw chat photo.jpg -m qwen/qwen3.5-0.8b -p "What's in this image?"` |
| "summarise this video" | `vlmrun gw chat clip.mp4 -m qwen/qwen3.8-27b -p "Summarise this clip"` |
| "find the people" | `vlmrun gw chat street.jpg -m roboflow/rfdetr-large --json-mode -j` |
| "mask every hat" | `vlmrun gw chat photo.jpg -m facebook/sam3.1 -p "hat"` |
| "body pose" / "hand pose" | `vlmrun gw chat crowd.jpg -m usyd-community/vitpose-plus-large` / `-m geopavlakos/hamer` |
| "embed these photos" | `vlmrun gw embed shots/*.jpg -m qwen/qwen3-vl-embedding-2b --json` |
| "subtitle this recording" | `vlmrun gw transcribe talk.mp4 -m nvidia/parakeet-tdt-0.6b-v3 -f srt` |
| "is this scan an invoice?" | `vlmrun gw s1 -s "Answer about the page." scan.jpg --noul is_invoice="Is this an invoice?"` |
| "classify this PDF before OCR" | `vlmrun gw s1 -s "Answer about the document." doc.pdf --detail low --choice kind="invoice\|contract\|other"` |
| "where is the total?" | `typesafe-sdk` `Box(...)` or HTTP `{"type": "box"}` (no CLI flag) |
| "is the gateway up?" | `vlmrun gw health`, or `curl -sS https://gateway.vlm.run/health` |

## Batches

```bash
for f in scans/*.pdf; do
  vlmrun gw chat "$f" -m zai-org/glm-ocr -ns -j --timeout 700 > "out/$(basename "${f%.pdf}").json"
done
jq -s 'map(.usage.cost // 0) | add' out/*.json
```

Bound concurrency (for example `xargs -P 4`) to stay under the rate limit. Anonymous callers get 30
calls per hour, so batch with a key.

## Platform API: predictions and agent executions

The platform API (`https://api.vlm.run/v1`, `VLMRUN_BASE_URL`) returns typed JSON from a hub domain,
your own schema, or an agent that runs as a job. Same `VLMRUN_API_KEY`. It goes through `VLMRun()`,
never through `vlmrun gw`.

```python
from pydantic import BaseModel
from vlmrun.client import VLMRun
from vlmrun.client.types import AgentExecutionConfig, GenerationConfig

client = VLMRun()   # validates the key on construction

class Invoice(BaseModel):
    vendor: str
    total: float

r = client.document.generate(file="invoice.pdf", domain="document.invoice", batch=True,
                             config=GenerationConfig(response_model=Invoice, confidence=True))
r = client.predictions.wait(r.id, timeout=600, sleep=5)   # raises TimeoutError
Invoice(**r.response)

e = client.agent.execute(name="my-agent:v1", inputs={"document": "https://example.com/invoice.pdf"},
                         config=AgentExecutionConfig(prompt="Extract line items"), toolsets=["core", "document"])
e = client.executions.wait(e.id, timeout=600, sleep=5)
```

| Call | Notes |
| --- | --- |
| `client.image.generate(domain=, images=[...] \| urls=[...], batch=False)` | Synchronous by default |
| `client.document` / `.audio` / `.video` `.generate(file= \| url=, domain=, batch=False, config=)` | Uploads the file, then submits. `autocast=True` casts a synchronous result. |
| `client.document.execute(name=, version="latest", ...)` | A named agent instead of a domain |
| `client.predictions.get(id)` / `.wait(id)` / `.list()` | |
| `client.agent.execute(name=, inputs=, config=, model="vlmrun-orion-1:auto", toolsets=)` | Batch only; `batch=False` raises `NotImplementedError`. Toolsets: `core`, `image`, `image-gen`, `world-gen`, `viz`, `document`, `video`, `web`. |
| `client.executions.get(id)` / `.wait(id)` / `.list(skip=0, limit=10)` | |
| `client.hub.list_domains()` | Domains and schema versions |

- `GenerationConfig`: `prompt`, `response_model` or `json_schema`, `temperature` (0), `max_tokens`,
  `detail` (`auto`, `lo`, `hi`), `confidence`, `grounding`, `page_indices`, `skills`, video segment fields.
- Status: `enqueued`, `pending`, `running`, `completed`, `failed`, `paused`.
- `client.agent.completions` is the OpenAI-compatible agent chat (`vlmrun chat` uses it). It is not `client.gateway.completions`.
- CLI: `vlmrun generate -i <file> -d <domain> [--schema s.json] [-p ...] [-k <skill dir>] [--skill-id name:ver] [--timeout 300] [-f json]`,
  `vlmrun execute -n my-agent:v1 -i <file> [-p ...] [-t core -t document] [-m vlmrun-orion-1:auto] [--callback-url U] [-f json]` (waits by default),
  `vlmrun executions list|get <id> [--wait]`, `vlmrun predictions list|get`, `vlmrun hub list|schema <domain>`, `vlmrun files list|upload|get|delete`.
- Errors derive from `vlmrun.client.exceptions.VLMRunError`, each with a `suggestion`:
  `AuthenticationError` (401/403), `ValidationError` (400/422), `ResourceNotFoundError` (404),
  `RateLimitError` (429), `RequestTimeoutError`, `NetworkError`, `ServerError` (5xx),
  `ConfigurationError`, `DependencyError` (`pip install "vlmrun[doc]"`, `[video]`, `[all]`), `InputError`.

## Re-checking this document

Flags come from `vlmrun <cmd> --help`. Catalog, envelopes and errors come from the live Gateway.
Depth lives at `https://docs.vlm.run/gateway/<page>`.

```bash
curl -sS https://gateway.vlm.run/health
curl -sS https://gateway.vlm.run/openai/models -H "Authorization: Bearer vlmrun" | jq '.data | length'
curl -sS https://gateway.vlm.run/typesafe/v1/models -H "Authorization: Bearer vlmrun"
vlmrun gw chat --help; vlmrun gw s1 --help
```
