<div align="center">
<p align="center" style="width: 100%;">
    <img src="https://raw.githubusercontent.com/vlm-run/.github/refs/heads/main/profile/assets/vlm-black.svg" alt="VLM Run Logo" width="80" style="margin-bottom: -5px; color: #2e3138; vertical-align: middle; padding-right: 5px;"><br>
</p>
<h2>VLM Run Skills</h2>
<p align="center"><a href="https://docs.vlm.run"><b>Website</b></a> | <a href="https://app.vlm.run/"><b>Platform</b></a> | <a href="https://docs.vlm.run/"><b>Docs</b></a> | <a href="https://docs.vlm.run/blog"><b>Blog</b></a> | <a href="https://discord.gg/AMApC2UzVY"><b>Discord</b></a>
</p>
<p align="center">
<a href="https://github.com/vlm-run/skills/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/github/license/vlm-run/skills.svg"></a>
<a href="https://discord.gg/AMApC2UzVY"><img alt="Discord" src="https://img.shields.io/badge/discord-chat-purple?color=%235765F2&label=discord&logo=discord"></a>
<a href="https://twitter.com/vlmrun"><img alt="Twitter Follow" src="https://img.shields.io/twitter/follow/vlmrun.svg?style=social&logo=twitter"></a>
</p>
</div>

VLM Run Skills are definitions for visual AI tasks like image understanding, video processing, and document extraction. They are interoperable with Anthropic's Claude Code.

The Skills in this repository follow the standardized [Agent Skill](https://agentskills.io/home) format.

## How do Skills work?

In practice, skills are self-contained folders that package instructions, scripts, and resources together for an AI agent to use on a specific use case. Each folder includes a `SKILL.md` file with YAML frontmatter (name and description) followed by the guidance your coding agent follows while the skill is active.

<p align="center">
  <a href="https://vlm.run" target="_blank" rel="noopener noreferrer">
    <img
      src="https://github.com/user-attachments/assets/68c63ddf-b0c0-4724-8caa-e7ae818c2edb"
      style="max-width: 400px; width: 70%;"
      alt="vlm.run"
    />
  </a>
</p>

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

### Claude Code

1. Register the repository as a plugin marketplace:

```
/plugin marketplace add vlm-run/skills
```

2. To install a skill, run:

```
/plugin install <skill-name>@vlm-run/skills
```

For example:

```
/plugin install vlmrun-cli-skill@vlm-run/skills
```

#### Configure your API key

Once the skill is installed, configure your API key using the CLI (get your key from [app.vlm.run](https://app.vlm.run)):

```bash
vlmrun config init
vlmrun config set --api-key <your-api-key>
vlmrun config show
```

#### Verify Installation

Once installed, verify the skill is loaded by asking Claude Code (requires restart):
```
What skills are available in the /vlmrun-cli-skill?
```

### Installing in Claude for Desktop

You should be able to install the skill by simply asking Claude to create a new skill from the [`SKILL.md`](./skills/vlmrun-cli-skill/SKILL.md) file.

## Skills

This repository contains skills for interacting with VLM Run's Orion visual AI agent. You can also contribute your own skills to the repository.

### Available skills

| Name | Description | Documentation |
|------|-------------|---------------|
| `vlmrun-cli-skill` | Use the VLM Run CLI to interact with Orion visual AI agent. Process images, videos, and documents with natural language. Supports image understanding/generation, object detection, OCR, video summarization, document extraction, and visual AI chat. | [SKILL.md](skills/vlmrun-cli-skill/SKILL.md) |
| `mm-skill` | Use the mm CLI to index, explore, query, and extract content from multi-modal directories containing images, videos, PDFs, code, and other files. | [SKILL.md](skills/mm-skill/SKILL.md) |

### Using skills in your coding agent

Once a skill is installed, mention it directly while giving your coding agent instructions:

- "Use the VLM Run CLI skill to describe what's in this image"
- "Use the VLM Run CLI skill to generate an image of a sunset over mountains"
- "Use the VLM Run CLI skill to extract the text from this receipt"
- "Use the VLM Run CLI skill to summarize this meeting recording"

Your coding agent automatically loads the corresponding `SKILL.md` instructions and helper scripts while it completes the task.

### Contribute or customize a skill

1. Copy one of the existing skill folders (for example, `skills/vlmrun-cli-skill/`) and rename it.
2. Update the new folder's `SKILL.md` frontmatter:
   ```markdown
   ---
   name: my-skill-name
   description: Describe what the skill does and when to use it
   ---

   # Skill Title
   Guidance + examples + guardrails
   ```
3. Add or edit supporting scripts, templates, and documents referenced by your instructions.
4. Add an entry to `.claude-plugin/marketplace.json` with a concise, human-readable description.
5. Reinstall or reload the skill bundle in your coding agent so the updated folder is available.

## Configuration

Configure your API key and settings using the VLM Run CLI:

```bash
vlmrun config init
vlmrun config set --api-key <your-api-key>
vlmrun config show
```

| Setting | Description |
|---------|-------------|
| `api_key` | Your VLM Run API key (required) |
| `base_url` | API base URL (default: `https://api.vlm.run/v1`) |
| `cache_dir` | Cache directory (default: `~/.vlmrun/cache/artifacts/`) |

## License

See [LICENSE](./LICENSE) for details.
