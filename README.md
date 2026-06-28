# Shorts Factory

**Autonomous short-form video production pipeline** — script generation through 4K upscale and multi-platform scheduling, running fully autonomous on a defined weekly window.

> Source code is private. This repo showcases the system architecture, tech stack, and design decisions.

---

## What It Does

Shorts Factory is a 26-module Python pipeline that autonomously produces, enhances, and publishes short-form video content across 6 YouTube channels. Given a theme and topic bank, it:

1. Scans competitor videos via YouTube Data API to surface trending topic signals
2. Generates scripts using a local LLM (Ollama/llama3.2) gated by a Gemini pre-render quality score
3. Synthesizes voiceover via Edge TTS (30+ voice pool, async per line)
4. Fetches or generates background footage (Pexels API / Archive.org / AI video generation)
5. Assembles ASS karaoke captions, mixes audio, composites final video via FFmpeg
6. Upscales frames with RealESRGAN x2 (spandrel) and interpolates to 60fps
7. Schedules uploads to YouTube at peak audience hours via OAuth2

The pipeline runs **~119 videos/month** across 7 content themes within YouTube's API quota.

---

## Architecture

See [`docs/architecture.html`](docs/architecture.html) for the full interactive system diagram — all 26 modules, data flow, API endpoints, state machine, and engineering decisions.

---

## Tech Stack

| Layer | Stack |
|---|---|
| Runtime | Python 3.13, Windows 10 |
| Web / API | Flask + Server-Sent Events (SSE) |
| AI — Script | Ollama (llama3.2), Gemini 2.5 Flash/Pro |
| AI — Image | FLUX.1-dev fp8, SDXL |
| AI — Video | Wan2.1 T2V (1.3B / 14B nf4) |
| TTS | Edge TTS (async), Kokoro (local fallback) |
| Video | FFmpeg, spandrel RealESRGAN x2, minterpolate |
| Publishing | YouTube Data API v3, TikTok OAuth2 (sandbox) |
| GPU | NVIDIA RTX 3090 (24GB VRAM) |

---

## Key Features

### Closed-Loop Intelligence
- **Competitor scanner** — weekly YouTube Data API scan ranks trending topic phrases by view velocity, injects top signals into script generation
- **Retention analysis** — pulls audience watch-ratio curves via YouTube Analytics; injects RETENTION WARNING into Gemini prompt if avg drop-at < 35%
- **A/B thumbnail testing** — registers variant B at +48h, measures CTR delta, auto-swaps winner via `thumbnails.set`

### Multi-Model AI Routing
`provider_score.py` dynamically selects the best video/image backend per render using a weighted scoring model across task fit (30%), quality (20%), control (15%), reliability (15%), cost (10%), and latency (10%). Every decision logged to `output/provider_log.jsonl`.

### Video State Machine
Each video moves through: `review` → `ready` → `scheduled` → `published`. Upload verifier detects silent failures and auto-resets missing videos to `review`. State persisted as flat JSON sidecars alongside each `.mp4` — no database.

### Dashboard
40+ REST endpoints, SSE job streaming for live render progress, full library management. No auth (localhost only).

### Multilingual Dubbing
Top 3 videos auto-dubbed into es/pt/fr/de/hi via Gemini Flash translation + Edge TTS re-synthesis, published to dedicated language channels.

---

## Configuration Schema

See [`config.schema.json`](config.schema.json) for the full configuration shape (keys redacted).

---

## Engineering Notes

- **spandrel over basicsr** — basicsr fails on Python 3.13 (`KeyError: '__version__'`); spandrel is a modern ESRGAN runner with no basicsr dependency
- **Flat file library** — no database; each video is a `.mp4` + `.json` sidecar pair; survives power cuts, diffs cleanly in git
- **Fail-soft everywhere** — every intelligence module wrapped in try/except that never blocks the main pipeline; a YouTube Analytics scope error doesn't stop the upload batch
- **SSE for job streaming** — `make_short` runs in a background thread, stdout captured and streamed to dashboard; no WebSocket dependency

---

## Output Scale

| Metric | Value |
|---|---|
| Videos generated/month | ~119 |
| YouTube uploads/month (English) | ~85 |
| Dubbed uploads/month (es + pt) | ~12 |
| Active YouTube channels | 6 |
| Content themes | 7 |
| Autopilot window | Sun 10pm – Thu 5pm |

---

*Built by [Landon Armstrong](https://github.com/Larmstrong1127) · Python 3.13 · Windows 10 · RTX 3090*
