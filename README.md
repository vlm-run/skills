# vlmrun-cli-skill

A Claude skill for interacting with VLM Run's Orion visual AI agent via CLI. Process images, videos, and documents with natural language prompts.

## Features

- Image understanding, analysis, and generation
- Video summarization, transcription, and generation
- Document extraction (PDF, invoices, contracts)
- Object detection and OCR
- Multi-file comparison

## Installation

### Prerequisites

1. Get your VLM Run API key from [VLM Run](https://vlm.run)
2. Have [uv](https://docs.astral.sh/uv/) installed for Python environment management

### Installing in Claude Code (CLI)

Skills in Claude Code are automatically loaded from specific directories. Choose one of the following options:

#### Option A: Personal Skill (available across all projects)

1. **Clone directly into your personal skills folder**:
   ```bash
   git clone https://github.com/vlm-run/vlmrun-cli-skill.git ~/.claude/skills/vlmrun-cli-skill
   ```

2. **Set up the environment**:
   ```bash
   cd ~/.claude/skills/vlmrun-cli-skill
   uv venv
   source .venv/bin/activate
   uv pip install "vlmrun[cli]"
   ```

3. **Configure your API key** by creating a `.env` file:
   ```bash
   cat <<EOF > .env
   VLMRUN_API_KEY="your-api-key"
   VLMRUN_BASE_URL="https://agent.vlm.run/v1"
   EOF
   ```

#### Option B: Project Skill (available to anyone working in a specific repository)

1. **Clone into your project's skills folder**:
   ```bash
   cd /path/to/your/project
   git clone https://github.com/vlm-run/vlmrun-cli-skill.git .claude/skills/vlmrun-cli-skill
   ```

2. **Set up the environment**:
   ```bash
   cd .claude/skills/vlmrun-cli-skill
   uv venv
   source .venv/bin/activate
   uv pip install "vlmrun[cli]"
   ```

3. **Configure your API key** by creating a `.env` file:
   ```bash
   cat <<EOF > .env
   VLMRUN_API_KEY="your-api-key"
   VLMRUN_BASE_URL="https://agent.vlm.run/v1"
   EOF
   ```

#### Verify Installation

Once installed, verify the skill is loaded by asking Claude:
```
What skills are available?
```

### Installing in Claude for Desktop

Claude Desktop also uses the `~/.claude/skills/` directory for personal skills.

1. **Clone directly into your personal skills folder**:
   ```bash
   git clone https://github.com/vlm-run/vlmrun-cli-skill.git ~/.claude/skills/vlmrun-cli-skill
   ```

2. **Set up the environment**:
   ```bash
   cd ~/.claude/skills/vlmrun-cli-skill
   uv venv
   source .venv/bin/activate
   uv pip install "vlmrun[cli]"
   ```

3. **Configure your API key** by creating a `.env` file:
   ```bash
   cat <<EOF > .env
   VLMRUN_API_KEY="your-api-key"
   VLMRUN_BASE_URL="https://agent.vlm.run/v1"
   EOF
   ```

4. **Restart Claude Desktop** to load the new skill.

## Usage

Once installed, the skill triggers automatically when you ask Claude to:

- Analyze or describe images
- Generate images or videos
- Extract text from documents (OCR)
- Summarize videos
- Process PDFs
- Detect objects in images

### Example Prompts

```
"Describe what's in this image" (with image attached)
"Generate an image of a sunset over mountains"
"Extract the text from this receipt"
"Summarize this meeting recording"
"What objects are visible in this photo?"
```

## Skill Specification

See [SKILL.md](./SKILL.md) for the full skill specification and CLI reference.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `VLMRUN_API_KEY` | Your VLM Run API key (required) |
| `VLMRUN_BASE_URL` | API base URL (default: `https://agent.vlm.run/v1`) |
| `VLMRUN_CACHE_DIR` | Cache directory (default: `~/.vlmrun/cache/artifacts/`) |

## License

See [LICENSE](./LICENSE) for details.
