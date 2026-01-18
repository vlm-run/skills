<div align="center">
<p align="center" style="width: 100%;">
    <img src="https://raw.githubusercontent.com/vlm-run/.github/refs/heads/main/profile/assets/vlm-black.svg" alt="VLM Run Logo" width="80" style="margin-bottom: -5px; color: #2e3138; vertical-align: middle; padding-right: 5px;"><br>
</p>
<h2>VLM Run CLI Skill</h2>
<p align="center"><a href="https://docs.vlm.run"><b>Website</b></a> | <a href="https://app.vlm.run/"><b>Platform</b></a> | <a href="https://docs.vlm.run/"><b>Docs</b></a> | <a href="https://docs.vlm.run/blog"><b>Blog</b></a> | <a href="https://discord.gg/AMApC2UzVY"><b>Discord</b></a>
</p>
<p align="center">
<a href="https://github.com/vlm-run/vlmrun-cli-skill/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/github/license/vlm-run/vlmrun-cli-skill.svg"></a>
<a href="https://discord.gg/AMApC2UzVY"><img alt="Discord" src="https://img.shields.io/badge/discord-chat-purple?color=%235765F2&label=discord&logo=discord"></a>
<a href="https://twitter.com/vlmrun"><img alt="Twitter Follow" src="https://img.shields.io/twitter/follow/vlmrun.svg?style=social&logo=twitter"></a>
</p>
</div>

A Claude skill for interacting with VLM Run's [Orion visual AI agent](https://vlm.run/orion) via CLI. Process images, videos, and documents with natural language prompts.


## Features

### Image Intelligence
- **Understanding & Captioning**: Describe, analyze, and interpret images with state-of-the-art visual intelligence
- **Detection & Localization**: Detect and locate objects, people, faces, and custom entities with bounding boxes
- **Segmentation**: Segment objects, scenes, and regions with pixel-level precision
- **Generation & Editing**: Generate images from text, edit existing images, apply super-resolution, colorize B&W photos
- **Tools**: Crop, rotate, enhance resolution (4x-8x upscaling), de-oldify (colorization)
- **Visual Grounding**: Point to and extract specific elements using natural language queries
- **UI Parsing**: Extract UI elements, layouts, and hierarchies from screenshots

### Video Intelligence
- **Understanding & Captioning**: Describe video content, generate summaries and detailed scene analysis
- **Transcription**: Extract audio transcripts with timestamps
- **Tools**: Trim videos, extract keyframes, sample frames at intervals, detect highlights
- **Segmentation**: Identify and segment objects across video frames
- **Generation & Editing**: Generate videos from text prompts, edit existing videos

### Document Intelligence
- **Layout Understanding**: Detect headers, paragraphs, tables, figures, lists, and structural elements
- **Multi-Page Analysis**: Process and analyze PDFs with intelligent page-aware extraction
- **Markdown Extraction**: Convert documents to clean, structured markdown with preserved formatting
- **Visual Grounding**: Locate and extract specific fields, sections, or data points
- **Data Extraction**: Extract key information from invoices, receipts, contracts, forms into structured JSON

### Multi-modal Agents
- **Multi-Modal Reasoning**: Execute complex multi-step workflows across images, documents, and videos
- **Structured Outputs**: Get results in validated JSON schemas with automatic retry logic

See [docs](https://docs.vlm.run/agents/introduction) and [technical whitepaper](https://vlm.run/orion/whitepaper) for more information.

## Installation

### Prerequisites

1. Get your VLM Run API key from [app.vlm.run](https://app.vlm.run)
2. Have [uv](https://docs.astral.sh/uv/) installed for Python environment management

### Installing in Claude Code (CLI)

Skills in Claude Code are automatically loaded from `~/.claude/skills/` directory. You can simply copy the entire `vlmrun-cli-skill` folder to the `~/.claude/skills/vlmrun-cli-skill` directory or symlink it (via `ln -s <git-repo-path>/vlmrun-cli-skill ~/.claude/skills/vlmrun-cli-skill`).

#### Update the .env file

Once the skill is installed, create an `.env` file (by copying from the `.env.template` file) with your API key that you can get from the dashboard at [https://app.vlm.run](https://app.vlm.run) and base URL.

#### Verify Installation

Once installed, verify the skill is loaded by asking Claude Code (requires restart):
```
What skills are available in the /vlmrun-cli-skill?
```

### Installing in Claude for Desktop

You should be able to install the skill by simply asking Claude to create a new skill from the [`SKILL.md`](./vlmrun-cli-skill/SKILL.md) file. 

## Usage

Once installed, the skill triggers automatically when you ask Claude to process images, videos, and documents with natural language.

### Example Prompts

```
"Describe what's in this image" (with image attached)
"Generate an image of a sunset over mountains"
"Extract the text from this receipt"
"Summarize this meeting recording"
"What objects are visible in this photo?"
```

## Skill Specification

See [`SKILL.md`](./vlmrun-cli-skill/SKILL.md) for the full skill specification and CLI reference.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `VLMRUN_API_KEY` | Your VLM Run API key (required) |
| `VLMRUN_BASE_URL` | API base URL (default: `https://agent.vlm.run/v1`) |
| `VLMRUN_CACHE_DIR` | Cache directory (default: `~/.vlmrun/cache/artifacts/`) |

## License

See [LICENSE](./LICENSE) for details.
