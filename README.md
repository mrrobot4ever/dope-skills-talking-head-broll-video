# Dope Skills - Talking Head B-Roll Video

An OpenClaw agent skill for generating talking-head B-roll video ads from an avatar image and audio file.

## What It Does

Takes an audio file (your script) and an avatar image (your spokesperson), then produces a complete 9:16 vertical video ad with:

- AI-generated talking head (via Creatify Aurora)
- Split-screen B-roll footage behind the avatar
- Transparent avatar overlay in the lower-right corner
- Human-in-the-loop approval at every step
- Optional speed boost for pacing

## Pipeline

1. **Audio splitting** -- splits at silence gaps to stay under Creatify's 59s limit
2. **A-roll generation** -- Creatify Aurora creates lip-synced avatar video
3. **Segmentation** -- divides A-roll into <=4s segments at natural phrase boundaries
4. **B-roll image generation** -- Gemini creates split-screen 9:16 images for each segment
5. **B-roll video generation** -- Grok Imagine animates each image with cinematic camera movement
6. **Compositing** -- removes avatar background (rembg), overlays transparent PNG frames on B-roll with synced audio
7. **Final assembly** -- concatenates all segments, applies speed boost

## Requirements

- macOS or Linux
- ffmpeg, Python 3.9+, whisper.cpp, rembg, Pillow
- API keys: Creatify, Google Gemini, xAI Grok
- GitHub CLI (for temporary file hosting)

See [references/setup.md](references/setup.md) for full installation instructions.

## Usage with OpenClaw

Copy the skill into your OpenClaw workspace:

```bash
cd ~/.openclaw/workspace/skills
git clone https://github.com/mrrobot4ever/dope-skills-talking-head-broll-video.git
```

Then tell your agent: *"I want to create a talking head b-roll video"* and provide an audio file and avatar image.

The agent reads `SKILL.md` and follows the pipeline step by step, asking for your approval at each stage.

## Documentation

| File | Purpose |
|------|---------|
| [SKILL.md](SKILL.md) | Main pipeline playbook -- the agent reads this |
| [references/setup.md](references/setup.md) | Installation, API keys, cost estimates |
| [references/command-reference.md](references/command-reference.md) | Copy-paste command templates for every step |
| [references/lessons-learned.md](references/lessons-learned.md) | Failure patterns and fixes from production |

## Key Design Decisions

- **Split-screen B-roll**: Every B-roll image is a 9:16 vertical split screen with two photos (top and bottom) separated by a thin white line
- **Human-in-the-loop**: User approves at every major step -- A-roll, each B-roll image, each B-roll video, each compound segment, final video
- **PNG transparency**: Avatar background removal uses PNG frame sequences, never WebM alpha (which flattens to black in MP4)
- **Frame-accurate extraction**: A-roll segments are re-encoded (not stream-copied) to prevent cumulative audio/video drift
- **State tracking**: Pipeline progress saved to `progress.json` for session resumability

## License

MIT
