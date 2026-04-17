---
name: dope-skills-talking-head-broll-video
description: Generate talking-head B-roll videos from an avatar image and audio file. Use when asked to create a UGC-style video ad with an AI avatar talking head overlaid on split-screen B-roll footage. Covers the full pipeline from audio splitting through Creatify Aurora avatar generation, split-screen B-roll image generation (Gemini), B-roll video generation (Grok Imagine), background removal (rembg), compound segment compositing, concatenation, and speed boost. Human-in-the-loop approval at every major step. Triggers on "talking head video", "b-roll video", "UGC video ad", "avatar video with b-roll", "dope skills video".
---

# Dope Skills - Talking Head B-Roll Video

Generate a complete talking-head B-roll video from an avatar image and audio file.

## Inputs Required

1. **Audio file** -- the speech audio (MP3, any length)
2. **Avatar image** -- a photo of the person (JPG/PNG, portrait orientation preferred)
3. **Optional: Product reference image** -- for segments that mention a specific product

## Pipeline Overview

```
Audio → Split at silence gaps → Creatify Aurora → A-roll video
    → Human approves A-roll
    → Segment A-roll into <=4s chunks
    → For each segment: generate split-screen B-roll image (Gemini) → human approves
    → For each segment: generate B-roll video (Grok Imagine) → human approves
    → For each segment: create compound (A-roll overlay on B-roll) → human approves
    → Concatenate all compound segments
    → Apply speed boost
    → Final video
```

## Step-by-Step Process

### Step 1: Split Audio at Silence Gaps

Split the source audio into the minimum number of chunks, all under 59 seconds.

**Critical rules:**
- Use `ffmpeg silencedetect` (noise=-30dB, d=0.2) to find silence gaps
- Split at the MIDPOINT of a silence gap near the optimal split point
- Re-encode each chunk with `-acodec libmp3lame -b:a 128k` (never use `-c copy` -- stream copy can produce chunks microseconds over 60s due to MP3 frame boundaries, and Creatify will reject them)
- Verify each chunk is under 59 seconds

```bash
# Find silence gaps
ffmpeg -i audio.mp3 -af "silencedetect=noise=-30dB:d=0.2" -f null - 2>&1 | grep "silence_"

# Split at silence midpoint (example: silence at 55.2-55.6s, midpoint 55.4s)
ffmpeg -y -i audio.mp3 -ss 0 -t 55.4 -acodec libmp3lame -b:a 128k chunk-1.mp3
ffmpeg -y -i audio.mp3 -ss 55.4 -acodec libmp3lame -b:a 128k chunk-2.mp3
```

### Step 2: Generate A-Roll via Creatify Aurora

Upload audio chunks and avatar image to public URLs (GitHub Release assets work), then submit to Creatify Aurora API.

**Key details:**
- Model: `aurora_v1_fast` (10 credits per 15s)
- Submit all chunks in parallel
- Poll every 30 seconds until status is "done"
- Download video output from `video_output` URL

**Credentials:** `source ~/.openclaw/workspace/.env.creatify`

### Step 3: Concatenate A-Roll and Send for Approval

```bash
# Concatenate with -c copy (no re-encode needed here -- same encoder settings)
ffmpeg -y -f concat -safe 0 -i concat.txt -c copy a-roll-full.mp4
```

**Human in the loop:** Send concatenated A-roll to user. Wait for approval before proceeding.

### Step 4: Segment A-Roll

Transcribe the A-roll using whisper to get precise timing:

```bash
whisper-cli -m models/whisper/ggml-base.en.bin -f a-roll-audio.wav --max-len 30
```

Split into segments of 4 seconds or less at natural phrase boundaries. Save segment plan as JSON:

```json
[
  {"seg": 1, "start": 0.00, "end": 4.16, "script": "text here"},
  ...
]
```

### Step 5: Generate Split-Screen B-Roll Images (Gemini)

For each segment, one at a time:

1. Present 3 options to the user, each with a top image and bottom image concept for a 9:16 split-screen layout
2. Include the segment script text when presenting options
3. User selects an option (or mixes top/bottom from different options)
4. Generate the image via Gemini Nano Banana 2 (`gemini-3.1-flash-image-preview`)

**Prompt template:**
```
Generate a single 9:16 portrait image split screen with two photos stacked vertically separated by a thin white line. Top: [description]. Bottom: [description]. Both photorealistic, high detail, cinematic quality.
```

**If user provides a product reference image:** Include it as `inlineData` in the Gemini request for segments that mention the product.

**Human in the loop:** Send each image to user. Wait for approval before moving to next segment.

### Step 6: Generate B-Roll Videos (Grok Imagine)

For each approved B-roll image, one at a time:

1. Compress image to under 100KB (resize to 360x640, JPEG quality 40)
2. Submit to Grok Imagine API (`grok-imagine-video`)
3. Duration: 4 seconds, aspect ratio: 9:16, resolution: 720p
4. Prompt: "Gentle slow camera movement, subtle parallax effect, cinematic photorealistic"

**Grok API format:**
```json
{
  "model": "grok-imagine-video",
  "prompt": "...",
  "image": {"url": "data:image/jpeg;base64,..."},
  "duration": 4,
  "resolution": "720p",
  "aspect_ratio": "9:16"
}
```

**SSL gotcha:** Python 3.9 with LibreSSL will fail on large base64 payloads. Build payload JSON with Python, submit via curl with `-d @payload.json`, then poll with Python requests (small responses work fine).

**Human in the loop:** Send each video to user. Wait for approval before moving to next segment.

### Step 7: Create Compound Segments

For each segment, one at a time:

#### 7a. Extract A-Roll Segment (Frame-Accurate)

**CRITICAL: Re-encode, do NOT use `-c copy`.**

Using `-c copy` cuts on keyframes, producing segments slightly longer than requested. Over 28 segments, this drift compounds to 2+ seconds of audio/video desync.

```bash
ffmpeg -y -ss $START -t $DUR -i a-roll-full.mp4 \
  -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k \
  aroll-segment.mp4
```

#### 7b. Remove Avatar Background (rembg)

Extract frames, run rembg with u2net model, keep as PNG sequence (alpha channel preserved).

```python
from rembg import remove, new_session
session = new_session("u2net")
# Process each frame...
```

**CRITICAL: Never use WebM alpha as an intermediate format.** WebM alpha gets flattened to black when composited into MP4. Use PNG frames directly in the ffmpeg overlay filter.

#### 7c. Retime B-Roll to Match A-Roll Duration

```bash
ffmpeg -y -i broll.mp4 \
  -filter:v "setpts=${AROLL_DUR}/${BROLL_DUR}*PTS" -an -r 25 \
  -c:v libx264 -preset fast -crf 18 \
  broll-retimed.mp4
```

#### 7d. Composite: Overlay PNG Frames on B-Roll

Avatar in lower-right corner, no padding (edge to edge on bottom and right).

```bash
ffmpeg -y \
  -i broll-retimed.mp4 \
  -framerate 25 -i nobg-frames/frame_%05d.png \
  -i aroll-segment.mp4 \
  -filter_complex "[1:v]scale=324:-1[av];[0:v][av]overlay=W-w:H-h:shortest=1[out]" \
  -map "[out]" -map "2:a" \
  -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  compound-segment.mp4
```

**Key:** Audio comes from the A-roll segment (`-map "2:a"`), NOT the B-roll. B-roll audio is stripped.

**Human in the loop:** Send each compound segment to user. Wait for approval before moving to next.

### Step 8: Concatenate All Compound Segments

```bash
ffmpeg -y -f concat -safe 0 -i concat-all.txt -c copy final-no-speed.mp4
```

**Human in the loop:** Send concatenated video (no speed change) for inspection. Check lip sync.

### Step 9: Apply Speed Boost

User specifies speed increase (e.g., 25%).

```bash
# 25% speed boost: setpts=0.8*PTS, atempo=1.25
ffmpeg -y -i final-no-speed.mp4 \
  -filter_complex "[0:v]setpts=0.8*PTS[v];[0:a]atempo=1.25[a]" \
  -map "[v]" -map "[a]" \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  final-sped-up.mp4
```

**Human in the loop:** Send final sped-up video for approval.

### Step 10: Delivery

If final video exceeds 50MB (Telegram send limit), compress with higher CRF (e.g., 26) without changing timing. Send via Telegram.

Final export must be: H.264 + yuv420p + AAC + faststart (QuickTime-safe).

## Critical Rules (Failure Patterns from Production)

1. **Audio splitting:** Always split at silence gaps. Never split mid-word -- it causes downstream artifacts in the avatar lip sync.

2. **Frame-accurate extraction:** Always re-encode A-roll segments (`-c:v libx264`). Using `-c copy` cuts on keyframes and causes cumulative drift that desyncs audio from video over multiple segments.

3. **Transparency:** PNG frames with alpha → ffmpeg overlay = correct transparency. WebM alpha → ffmpeg overlay → MP4 = black background (alpha flattened). Never use WebM as an intermediate.

4. **B-roll duration:** Always retime B-roll to match A-roll duration exactly using `setpts` filter. A-roll timing is the source of truth.

5. **Grok API SSL:** Python 3.9 + LibreSSL breaks on large base64 payloads. Build JSON payload with Python, submit with curl, poll with Python.

6. **Gemini safety filter:** Avoid suggestive language in image prompts. If blocked, rephrase to be more neutral.

7. **A-roll audio is source of truth:** Always use A-roll audio in compound segments. Never use B-roll audio.

8. **Segment duration:** Keep segments at 4 seconds or less for visual variety and B-roll generation compatibility.

## File Organization

```
projects/<project-name>/
  source-audio.mp3          -- original audio
  avatar.jpg                -- avatar image
  audio-chunks/             -- split audio chunks
  a-roll/                   -- raw Creatify output
  a-roll-full.mp4           -- concatenated A-roll
  segments.json             -- segment timing plan
  broll-images/             -- approved split-screen images
  broll-videos/             -- approved B-roll videos
  compound-segments-v2/     -- final compound segments (frame-accurate)
  work/                     -- intermediate files (frames, nobg, retimed)
  final-v2-no-speed.mp4     -- concatenated, no speed change
  final-v2-sped-up.mp4      -- final with speed boost
```

## API Reference

- **Creatify Aurora:** See `skills/creatify-api/SKILL.md` or `wiki/pages/creatify-api.md`
- **Gemini Nano Banana 2:** API key in `.env.google`, endpoint `generativelanguage.googleapis.com`
- **Grok Imagine:** API key in `.env.xai`, endpoint `api.x.ai/v1/videos/generations`
- **rembg:** `pip3 install "rembg[cpu]"`, model `u2net` (auto-downloads ~176MB)
- **whisper:** `whisper-cli` with `ggml-base.en.bin` model

## Prerequisites

See `references/setup.md` for full installation and API key setup instructions.
