---
name: minimax-mmx-cli
description: Official MiniMax CLI for vision, search, text chat, TTS, voice, music, and video generation. Install via npm.
triggers:
  - "set up minimax cli"
  - "minimax web search"
  - "minimax vision"
  - "mmx cli"
  - "official minimax agent"
---

# MiniMax mmx-cli Skill

Official command-line tool from MiniMax AI providing text, vision, search, TTS, voice cloning, music generation, and video generation.

## Installation

```bash
npm install -g mmx-cli
```

## Authentication

```bash
mmx auth login --api-key YOUR_MINIMAX_API_KEY --region global
```

The API key is stored in `~/.mmx/config.json`. Get your key from [MiniMax dashboard](https://platform.minimax.io).

To test auth:
```bash
mmx auth status
```

## Key Commands

### Vision (image understanding)
```bash
mmx vision describe --image /path/to/image.jpg --prompt "What do you see?" --output json --quiet
```
- `--quiet` suppresses progress output
- `--output json` gives structured JSON output
- Works with JPG, PNG, WebP

**Use case**: Read text from images, analyze photos, extract data.

### Web Search
```bash
mmx search query "search query" --limit 5 --output json
```
- `--limit N` controls result count
- `--output json` for structured output
- Returns organic results with title, link, snippet, date

**Use case**: General web search as an alternative to OpenRouter search.

### Text Chat
```bash
mmx text chat "your prompt" --model MiniMax-M2.7 --output json --quiet
```
- `--model` defaults to `MiniMax-M2.7`

### TTS (text-to-speech)
```bash
mmx tts speak "Hello world" --voice "male-qn-qingse" --output /tmp/hello.mp3
```
- List available voices: `mmx voice list`

### Voice Cloning
```bash
mmx voice clone --name "my-voice" --audio /path/to/reference.mp3
```

### Music Generation
```bash
mmx music generate --prompt "uplifting rock anthem" --duration 30 --output /tmp/song.mp3
```

### Image Generation
```bash
mmx image generate --prompt "a photo of a cat" --model MiniMax-Image-01 --output /tmp/cat.png
```

### Video Generation
```bash
mmx video generate --prompt "a flowing waterfall" --duration 5 --output /tmp/video.mp4
```

## Check Quota
```bash
mmx quota show
```
Shows usage for the current week across all models. Quotas reset weekly.

## ⚠️ Image Generation: NOT Free — Paid Plans Only

**`image-01` is NOT available on free or Starter ($10/mo) plans.** It requires Plus ($20/mo) or higher.

| Plan | Price | image-01 daily limit |
|------|-------|----------------------|
| Starter | $10/mo | ❌ Not included |
| **Plus** | **$20/mo** | **50 images/day** |
| Max | $50/mo | 120 images/day |
| Ultra-Highspeed | $150/mo | 800 images/day |

**Consumer app vs Developer API:** The free Android app has separate consumer quotas — these do NOT apply to the developer API. If you need image gen via API, you must pay.

If you need free image generation, use these alternatives (see Free Image Gen Alternatives skill).

## MiniMax MCP vs mmx-cli

- **`minimax-mcp`** (uvx package): MCP server providing generation tools only (TTS, voice, music, video gen). NO vision, NO search.
- **`mmx-cli`** (npm package): Full CLI with vision (`vision describe`), search (`search query`), text chat, and generation. **This is the one to use for vision and search.**

## Tips

1. **Vision first**: When given an image, always use `mmx vision describe` before other tools — it's MiniMax's official VLM interface.
2. **Quota monitoring**: Check `mmx quota show` if planning heavy usage. Vision calls count against `coding-plan-vlm`.
3. **JSON output**: Use `--output json --quiet` for clean, parseable output in scripts.
4. **API key format**: MiniMax API keys start with `sk-cp-`, 125 characters, end with `YHCU` (for platform.minimax.io).
5. **Region**: Use `--region global` for the main API. Other regions may have different endpoints.

## API Key Storage

The API key is stored at `~/.mmx/config.json`. The full key is also stored locally at `/tmp/mmkey3.txt` (for backup/reference).

## Files

- `/root/.mmx/config.json` — authenticated session
- `/tmp/mmkey3.txt` — backup of full API key
