---
name: dope-skills-talking-head-broll-video
description: Generate talking-head B-roll videos from an avatar image and audio file. Use when asked to create a UGC-style video ad with an AI avatar talking head overlaid on split-screen B-roll footage. Covers the full pipeline from audio splitting through Creatify Aurora avatar generation, split-screen B-roll image generation (Gemini), B-roll video generation (Grok Imagine), background removal (rembg), compound segment compositing, concatenation, and speed boost. Human-in-the-loop approval at every major step. Triggers on "talking head video", "b-roll video", "UGC video ad", "avatar video with b-roll", "dope skills video".
---

# Dope Skills - Talking Head B-Roll Video

Generate a complete talking-head B-roll video from an avatar image and audio file.

## Inputs Required

1. **Audio file** -- the speech audio (MP3, any length)
2. **Avatar image** -- a photo of the person (JPG/PNG, portrait orientation preferred)
3. **Optional: Product reference image** -- for segments that mention a specific product (include as inlineData in Gemini requests)

## Global Constants

These apply to EVERY step of the pipeline. Do not deviate.

| Constant | Value |
|----------|-------|
| Aspect ratio | 9:16 (portrait, vertical video) |
| Video resolution | 720x1280 |
| Frame rate | 25 fps |
| Avatar overlay position | Lower-right corner, no padding (edge to edge on bottom and right) |
| Avatar overlay width | 45% of viewport width (324px at 720px wide) |
| B-roll image format | Split-screen: two photos stacked vertically, separated by thin white horizontal line |
| Audio codec | AAC 128kbps |
| Video codec | H.264 (libx264), yuv420p, faststart |
| Max Creatify audio chunk | 59 seconds (hard limit is 60s, never exceed 59s) |
| Max A-roll segment duration | 5 seconds |
| Speed boost | User-specified (typically 20-25%) |

## Creative Guidelines

### B-Roll Images

ALL B-roll images MUST be:
- **9:16 portrait orientation**
- **Split-screen layout**: two distinct photos stacked top and bottom
- **Separated by a thin white horizontal line** in the middle
- **Photorealistic, high detail, cinematic quality**

When presenting B-roll options to the user:
- Always present exactly 3 options (A, B, C)
- Each option describes BOTH a top image and a bottom image
- Include the segment's script text before the options
- User may select one option or mix top from one and bottom from another

### Gemini Image Prompt Template

Always use this structure:
```
Generate a single 9:16 portrait image split screen with two photos stacked vertically separated by a thin white line. Top: [DESCRIPTION]. Bottom: [DESCRIPTION]. Both photorealistic, high detail, cinematic quality.
```

If user has provided a product reference image, include it as `inlineData` in the Gemini API request and reference it in the prompt: "matching the product in the reference image."

### Grok Video Prompt

Always use: `"Gentle slow camera movement, subtle parallax effect, cinematic photorealistic"`

## State Management

Track pipeline progress in `progress.json` in the project directory. Update after each approval.

```json
{
  "step": "compound-segments-v2",
  "a_roll_approved": true,
  "segments_total": 28,
  "broll_images_approved": [1, 2, 3, ...],
  "broll_videos_approved": [1, 2, 3, ...],
  "compound_segments_approved": [1, 2, 3, ...],
  "concat_approved": false,
  "speed_boost_approved": false,
  "speed_boost_percent": 25
}
```

This makes the pipeline resumable if the session restarts. Always read `progress.json` before starting work to determine where to resume.

## Pipeline Overview

```
Audio → Split at silence gaps → Creatify Aurora → A-roll video
    → Human approves A-roll
    → Segment A-roll into <=5s chunks (cut at silence gaps)
    → For each segment: generate split-screen B-roll image (Gemini) → human approves
    → For each segment: generate B-roll video (Grok Imagine) → human approves
    → For each segment: create compound (A-roll overlay on B-roll) → human approves
    → Concatenate all compound segments → human inspects
    → Apply speed boost → human approves
    → Final video
```

## Step-by-Step Process

### Step 1: Split Audio at Silence Gaps

Split the source audio into the minimum number of chunks, all under 59 seconds.

**Why silence gaps matter:** If you split mid-word, the avatar will generate with a cut-off syllable at the chunk boundary. This causes an audible glitch in the final video that cannot be fixed without regenerating the avatar.

**Procedure:**

1. Get total audio duration
2. Calculate minimum number of chunks needed (ceiling of duration / 59)
3. Find silence gaps near the optimal split points using silencedetect
4. Split at the MIDPOINT of each silence gap
5. Re-encode each chunk (never use `-c copy`)
6. Verify each chunk is under 59 seconds

See `references/command-reference.md` → "Audio Splitting" for exact commands.

### Step 2: Generate A-Roll via Creatify Aurora

Upload audio chunks and avatar image to public URLs, then submit to Creatify Aurora API.

**Procedure:**

1. Upload files to GitHub Release assets (temporary hosting)
2. Submit all chunks to Aurora API in parallel
3. Poll every 30 seconds until all chunks report status "done"
4. Download video output from each chunk
5. Clean up temporary GitHub release

See `references/command-reference.md` → "Creatify Aurora" for exact commands.

### Step 3: Concatenate A-Roll and Send for Approval

Concatenate all A-roll chunks into one video and send to user.

```bash
echo "file 'chunk-1.mp4'" > concat.txt
echo "file 'chunk-2.mp4'" >> concat.txt
ffmpeg -y -f concat -safe 0 -i concat.txt -c copy a-roll-full.mp4
```

**HUMAN IN THE LOOP:** Send concatenated A-roll to user. Do NOT proceed until user approves. If rejected, determine what's wrong and redo from Step 1 or Step 2 as needed.

### Step 4: Segment A-Roll

This step uses BOTH whisper transcription AND silence detection to plan segment boundaries. Whisper tells you where the content breaks are. Silence detection tells you where to actually cut.

#### 4a. Transcribe A-Roll

```bash
ffmpeg -y -i a-roll-full.mp4 -vn -ar 16000 -ac 1 -c:a pcm_s16le a-roll-audio.wav
whisper-cli -m models/whisper/ggml-base.en.bin -f a-roll-audio.wav --max-len 30
```

#### 4b. Detect All Silence Gaps in A-Roll

```bash
ffmpeg -i a-roll-full.mp4 -af "silencedetect=noise=-30dB:d=0.15" -f null - 2>&1 | grep "silence_"
```

Build a list of all silence gap midpoints from the output.

#### 4c. Plan Segments

Using the whisper transcript, identify natural phrase/sentence boundaries that produce segments of **5 seconds or less**.

For EACH planned boundary:
1. Find the nearest silence gap midpoint within +/- 0.5 seconds of the whisper boundary
2. If a silence gap exists within that window, snap the cut point to that silence midpoint
3. If no silence gap exists within the window, use the whisper timestamp as fallback

**Why this matters:** Whisper timestamps are approximate -- they estimate word boundaries but are not frame-perfect. Cutting at a whisper timestamp can land in the middle of a phoneme, producing a micro-clip at the segment boundary. Across 25+ segments concatenated together, these micro-clips create an unnatural choppy feel. Cutting at actual silence gaps eliminates this entirely.

Save the final segment plan as `segments.json`:

```json
[
  {"seg": 1, "start": 0.00, "end": 4.16, "dur": 4.16, "script": "45 years at the emergency Johnson Clinic and they still", "cut_method": "silence_gap"},
  {"seg": 2, "start": 4.16, "end": 8.25, "dur": 4.09, "script": "don't believe me when I tell them...", "cut_method": "silence_gap"}
]
```

The `cut_method` field documents whether each boundary was snapped to a silence gap or used a whisper fallback.

### Step 5: Generate Split-Screen B-Roll Images (Gemini)

Process segments ONE AT A TIME. For each segment:

1. Present the segment script to the user
2. Present 3 B-roll options (A, B, C), each with top and bottom image descriptions
3. Wait for user to select an option
4. Generate the image via Gemini Nano Banana 2
5. Send the image to user for approval
6. If rejected, offer new options or regenerate
7. If approved, update `progress.json` and move to next segment

**Error handling:**
- If Gemini returns `IMAGE_SAFETY`: rephrase the prompt to be more neutral, avoid suggestive language
- If Gemini returns empty/no image: retry once, if still fails try a simplified prompt
- If the API times out: retry with `--max-time 300`

See `references/command-reference.md` → "Gemini Image Generation" for exact commands.

### Step 6: Generate B-Roll Videos (Grok Imagine)

Process segments ONE AT A TIME. For each approved B-roll image:

1. Compress image: resize to 360x640, JPEG quality 40, target under 100KB
2. Build JSON payload with Python (base64 encode the image)
3. Submit via curl (NOT Python requests -- SSL breaks on large payloads)
4. Poll via Python requests until status is "done" (small responses work fine)
5. Download video, send to user for approval
6. If approved, update `progress.json` and move to next segment

**Error handling:**
- If SSL error on submit: verify image is under 100KB, use curl not Python requests
- If Grok returns "failed": retry once with same image
- If persistent failures: regenerate the source image at lower resolution

See `references/command-reference.md` → "Grok Video Generation" for exact commands.

### Step 7: Create Compound Segments

Process segments ONE AT A TIME. For each segment:

#### 7a. Extract A-Roll Segment (Frame-Accurate)

**CRITICAL -- READ THIS CAREFULLY:**

You MUST re-encode when extracting A-roll segments. Do NOT use `-c copy`.

**Why:** Using `-c copy` cuts on keyframes, not at the exact requested timestamp. Each segment comes out ~0.08-0.2s longer than requested because ffmpeg must extend to the nearest keyframe. Over 28 segments, this drift compounds to +2.24 seconds of cumulative error. The result is that the avatar's lip movements desync from the audio -- the audio plays ahead of the video, getting progressively worse with each segment.

The re-encode approach cuts at the exact frame boundary, producing segments with the precise duration requested. The frame count matches the audio duration exactly, maintaining perfect lip sync across all segments.

```bash
# CORRECT: Re-encode for frame-accurate cut
ffmpeg -y -ss $START -t $DUR -i a-roll-full.mp4 \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k \
  aroll-segment.mp4

# WRONG: Never do this -- causes cumulative drift
# ffmpeg -y -ss $START -t $DUR -i a-roll-full.mp4 -c copy aroll-segment.mp4
```

#### 7b. Remove Avatar Background (rembg)

Extract frames from the A-roll segment, remove background with rembg, keep as PNG sequence.

```python
from rembg import remove, new_session
session = new_session("u2net")
# For each frame: remove(input_bytes, session=session) → output_bytes with alpha
```

**CRITICAL:** Keep frames as PNG files. PNG preserves the alpha channel. Do NOT encode to WebM with alpha as an intermediate step -- WebM alpha gets flattened to black when composited into MP4 via ffmpeg overlay.

#### 7c. Retime B-Roll to Match A-Roll Duration

The B-roll video (from Grok) is always 4 seconds. The A-roll segment may be shorter or longer. Adjust B-roll speed so its duration exactly matches the A-roll segment.

```bash
ffmpeg -y -i broll.mp4 \
  -filter:v "setpts=${AROLL_DUR}/${BROLL_DUR}*PTS" \
  -an -r 25 \
  -c:v libx264 -preset fast -crf 18 \
  broll-retimed.mp4
```

#### 7d. Composite: Overlay PNG Frames on B-Roll

Combine the retimed B-roll (background), transparent avatar PNG frames (overlay), and A-roll audio into the final compound segment.

```bash
ffmpeg -y \
  -i broll-retimed.mp4 \
  -framerate 25 -i nobg-frames/frame_%05d.png \
  -i aroll-segment.mp4 \
  -filter_complex "[1:v]scale=324:-1[av];[0:v][av]overlay=W-w:H-h:shortest=1[out]" \
  -map "[out]" -map "2:a" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  compound-segment.mp4
```

**Key details:**
- Input 0: retimed B-roll (background video, no audio)
- Input 1: PNG frames with alpha (avatar overlay)
- Input 2: A-roll segment (source of audio ONLY)
- `scale=324:-1`: avatar at 45% of 720px viewport width
- `overlay=W-w:H-h`: lower-right corner, edge to edge
- `shortest=1`: stop when the shortest input ends
- Audio comes from input 2 (A-roll), NOT the B-roll

**HUMAN IN THE LOOP:** Send each compound segment to user. Wait for approval before moving to next.

### Step 8: Concatenate All Compound Segments

```bash
# Generate concat list
for i in $(seq -f "%02g" 1 $TOTAL_SEGMENTS); do
  echo "file 'compound-${i}.mp4'" >> concat-all.txt
done

ffmpeg -y -f concat -safe 0 -i concat-all.txt -c copy final-no-speed.mp4
```

**HUMAN IN THE LOOP:** Send concatenated video (NO speed change) to user for lip sync inspection. If lip sync is off, there's a drift issue in the compound segment creation -- investigate per Step 7a.

### Step 9: Apply Speed Boost

Ask user what speed increase they want (typically 20-25%).

```bash
# Example: 25% speed boost (1.25x speed)
# Video: setpts = 1/1.25 = 0.8
# Audio: atempo = 1.25

ffmpeg -y -i final-no-speed.mp4 \
  -filter_complex "[0:v]setpts=0.8*PTS[v];[0:a]atempo=1.25[a]" \
  -map "[v]" -map "[a]" \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  final-sped-up.mp4
```

**HUMAN IN THE LOOP:** Send final sped-up video for approval.

### Step 10: Delivery

If final video exceeds 50MB (Telegram send limit), compress with higher CRF:

```bash
ffmpeg -y -i final-sped-up.mp4 \
  -c:v libx264 -preset fast -crf 26 \
  -c:a copy \
  -pix_fmt yuv420p -movflags +faststart \
  final-compressed.mp4
```

Final export MUST be: H.264 + yuv420p + AAC + faststart (QuickTime-safe).

## Error Handling Quick Reference

| Error | Cause | Fix |
|-------|-------|-----|
| Creatify rejects audio | Chunk is >=60s | Re-split with re-encode at 59s or less |
| Gemini `IMAGE_SAFETY` | Prompt has suggestive content | Rephrase neutrally |
| Gemini empty response | API timeout or overload | Retry with `--max-time 300` |
| Grok SSL error | Large base64 payload via Python | Use curl for submit, Python for polling |
| Black avatar background | Used WebM alpha intermediate | Use PNG frames directly in overlay |
| Lip sync drift | Used `-c copy` for segment extraction | Re-encode segments (libx264) |
| Audio cut mid-word | Split at arbitrary time, not silence gap | Re-split at silence midpoint |
| Choppy segment transitions | Segment boundaries at whisper timestamps, not silence gaps | Snap boundaries to nearest silence gap midpoint (+/- 0.5s) |

## File Organization

```
projects/<project-name>/
  source-audio.mp3              -- original audio input
  avatar.jpg                    -- avatar image input
  progress.json                 -- pipeline state tracker
  segments.json                 -- segment timing plan
  audio-chunks/                 -- split audio chunks
    chunk-1.mp3, chunk-2.mp3
  a-roll/                       -- raw Creatify output
    chunk-1.mp4, chunk-2.mp4
  a-roll-full.mp4               -- concatenated A-roll (approved)
  broll-images/                 -- approved split-screen images
    seg-01.png ... seg-28.png
  broll-videos/                 -- approved B-roll videos
    seg-01.mp4 ... seg-28.mp4
  compound-segments-v2/         -- final compound segments
    compound-01.mp4 ... compound-28.mp4
  work/                         -- intermediate files
    aroll-segments/             -- extracted A-roll segments
    seg{N}-frames/              -- raw extracted frames
    seg{N}-nobg/                -- frames with background removed
    broll-{N}-retimed.mp4       -- retimed B-roll clips
  final-v2-no-speed.mp4         -- concatenated, no speed change
  final-v2-sped-up.mp4          -- final with speed boost
```

## Prerequisites

See `references/setup.md` for full installation and API key setup instructions.
See `references/command-reference.md` for copy-paste command templates.
See `references/lessons-learned.md` for failure patterns and fixes.
