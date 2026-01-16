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

1. **Clone this repository** to your local machine:
   ```bash
   git clone https://github.com/vlm-run/vlmrun-cli-skill.git
   ```

2. **Add the skill** using Claude Code's `/install-skill` command:
   ```bash
   claude
   > /install-skill /path/to/vlmrun-cli-skill
   ```

   Or manually add to your Claude Code settings (`~/.claude/settings.json`):
   ```json
   {
     "skills": [
       "/path/to/vlmrun-cli-skill"
     ]
   }
   ```

3. **Set up the environment** (Claude will do this automatically when the skill triggers, or run manually):
   ```bash
   cd /path/to/vlmrun-cli-skill
   uv venv
   source .venv/bin/activate
   uv pip install "vlmrun[cli]"
   ```

4. **Configure your API key** by creating a `.env` file:
   ```bash
   cat <<EOF > .env
   VLMRUN_API_KEY="your-api-key"
   VLMRUN_BASE_URL="https://agent.vlm.run/v1"
   EOF
   ```

### Installing in Claude for Desktop

1. **Clone this repository** to your local machine:
   ```bash
   git clone https://github.com/vlm-run/vlmrun-cli-skill.git
   ```

2. **Open Claude Desktop settings**:
   - macOS: `Claude` menu → `Settings` → `Skills`
   - Windows: `File` menu → `Settings` → `Skills`

3. **Add the skill path**:
   - Click "Add Skill"
   - Browse to and select the cloned `vlmrun-cli-skill` directory

4. **Set up the environment** in the skill directory:
   ```bash
   cd /path/to/vlmrun-cli-skill
   uv venv
   source .venv/bin/activate
   uv pip install "vlmrun[cli]"
   ```

5. **Configure your API key** by creating a `.env` file in the skill directory:
   ```bash
   cat <<EOF > .env
   VLMRUN_API_KEY="your-api-key"
   VLMRUN_BASE_URL="https://agent.vlm.run/v1"
   EOF
   ```

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
