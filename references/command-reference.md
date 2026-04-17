# Command Reference

Copy-paste command templates for every step of the pipeline. Replace `$VARIABLES` with actual values.

## Variables

```bash
WORKDIR="/path/to/projects/<project-name>"
AUDIO="$WORKDIR/source-audio.mp3"
AVATAR="$WORKDIR/avatar.jpg"
```

---

## Audio Splitting

### Find silence gaps

```bash
ffmpeg -i "$AUDIO" -af "silencedetect=noise=-30dB:d=0.2" -f null - 2>&1 | grep "silence_"
```

### Get total duration

```bash
DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$AUDIO")
echo "Duration: ${DURATION}s"
```

### Calculate optimal split points

```python
import math
duration = $DURATION
chunk_len = 59.0
num_chunks = math.ceil(duration / chunk_len)
each_chunk = duration / num_chunks
print(f"Chunks: {num_chunks}, each ~{each_chunk:.2f}s")
```

### Split at silence midpoint

```bash
# Replace $SPLIT with the midpoint of the silence gap
ffmpeg -y -i "$AUDIO" -ss 0 -t $SPLIT -acodec libmp3lame -b:a 128k "$WORKDIR/audio-chunks/chunk-1.mp3"
ffmpeg -y -i "$AUDIO" -ss $SPLIT -acodec libmp3lame -b:a 128k "$WORKDIR/audio-chunks/chunk-2.mp3"
```

### Verify chunk durations

```bash
for f in "$WORKDIR"/audio-chunks/chunk-*.mp3; do
  DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$f")
  echo "$(basename $f): ${DUR}s"
done
```

---

## Creatify Aurora

### Upload files to GitHub Release

```bash
cd "$WORKDIR"
gh release create temp-upload --repo $GITHUB_USER/$GITHUB_REPO --title "temp" --notes "temp" \
  audio-chunks/chunk-1.mp3 audio-chunks/chunk-2.mp3 avatar.jpg
```

### Get public URLs

```bash
gh release view temp-upload --repo $GITHUB_USER/$GITHUB_REPO --json assets --jq '.assets[] | "\(.name): \(.url)"'
```

### Submit to Aurora

```bash
source ~/.openclaw/workspace/.env.creatify

curl -s -X POST "https://api.creatify.ai/api/aurora/" \
  -H "Content-Type: application/json" \
  -H "X-API-ID: $CREATIFY_API_ID" \
  -H "X-API-KEY: $CREATIFY_API_KEY" \
  -d '{
    "image": "'$IMAGE_URL'",
    "audio": "'$AUDIO_URL'",
    "model_version": "aurora_v1_fast"
  }'
```

### Poll for completion

```bash
curl -s -X GET "https://api.creatify.ai/api/aurora/$JOB_ID/" \
  -H "X-API-ID: $CREATIFY_API_ID" \
  -H "X-API-KEY: $CREATIFY_API_KEY"
```

Poll every 30 seconds. When `status` is `"done"`, download from `video_output` URL.

### Check remaining credits

```bash
curl -s -X GET "https://api.creatify.ai/api/remaining_credits/" \
  -H "X-API-ID: $CREATIFY_API_ID" \
  -H "X-API-KEY: $CREATIFY_API_KEY"
```

### Clean up temp release

```bash
gh release delete temp-upload --repo $GITHUB_USER/$GITHUB_REPO --yes
```

---

## Whisper Transcription & Silence Detection

### Extract audio from video

```bash
ffmpeg -y -i "$WORKDIR/a-roll-full.mp4" -vn -ar 16000 -ac 1 -c:a pcm_s16le "$WORKDIR/work/a-roll-audio.wav"
```

### Transcribe with timestamps

```bash
whisper-cli -m /path/to/models/whisper/ggml-base.en.bin \
  -f "$WORKDIR/work/a-roll-audio.wav" \
  --max-len 30
```

### Detect silence gaps in A-roll

```bash
ffmpeg -i "$WORKDIR/a-roll-full.mp4" -af "silencedetect=noise=-30dB:d=0.15" -f null - 2>&1 | grep "silence_"
```

### Build silence gap midpoint list (Python)

```python
import subprocess, re

result = subprocess.run(
    ["ffmpeg", "-i", "a-roll-full.mp4", "-af", "silencedetect=noise=-30dB:d=0.15", "-f", "null", "-"],
    capture_output=True, text=True
)

silence_gaps = []
starts = re.findall(r'silence_start: ([\d.]+)', result.stderr)
ends = re.findall(r'silence_end: ([\d.]+)', result.stderr)

for s, e in zip(starts, ends):
    midpoint = (float(s) + float(e)) / 2
    silence_gaps.append(midpoint)

print(f"Found {len(silence_gaps)} silence gaps")
```

### Snap segment boundaries to silence gaps (Python)

```python
def snap_to_silence(target_time, silence_gaps, window=0.5):
    """Find the nearest silence gap midpoint within +/- window seconds."""
    best = None
    best_dist = float('inf')
    for gap in silence_gaps:
        dist = abs(gap - target_time)
        if dist <= window and dist < best_dist:
            best = gap
            best_dist = dist
    return best if best is not None else target_time
```

---

## Gemini Image Generation

### Without reference image

```bash
APIKEY=$(grep GOOGLE_AI_API_KEY ~/.openclaw/workspace/.env.google | cut -d= -f2)

curl -s --max-time 180 \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image-preview:generateContent?key=$APIKEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{"parts": [{"text": "Generate a single 9:16 portrait image split screen with two photos stacked vertically separated by a thin white line. Top: [DESCRIPTION]. Bottom: [DESCRIPTION]. Both photorealistic, high detail, cinematic quality."}]}],
    "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}
  }'
```

### With reference image

```python
import base64, json

with open("reference-product.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

payload = {
    "contents": [{"parts": [
        {"text": "Generate a single 9:16 portrait image split screen... matching the product in the reference image."},
        {"inlineData": {"mimeType": "image/jpeg", "data": img_b64}}
    ]}],
    "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}
}
# Submit via curl with -d @payload.json
```

### Extract image from response

```python
import json, base64

with open("response.json") as f:
    d = json.load(f)

for part in d["candidates"][0]["content"]["parts"]:
    if "inlineData" in part:
        data = base64.b64decode(part["inlineData"]["data"])
        with open("output.png", "wb") as f:
            f.write(data)
        break
```

---

## Grok Imagine Video Generation

### Compress image for upload

```python
from PIL import Image
import os

img = Image.open("broll-image.png").convert("RGB").resize((360, 640))
img.save("compressed.jpg", "JPEG", quality=40)
# Verify under 100KB
print(f"{os.path.getsize('compressed.jpg') // 1024}KB")
```

### Build payload and submit (curl, NOT Python requests)

```python
import base64, json

with open("compressed.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

payload = {
    "model": "grok-imagine-video",
    "prompt": "Gentle slow camera movement, subtle parallax effect, cinematic photorealistic",
    "image": {"url": f"data:image/jpeg;base64,{img_b64}"},
    "duration": 4,
    "resolution": "720p",
    "aspect_ratio": "9:16"
}

with open("grok-payload.json", "w") as f:
    json.dump(payload, f)
```

```bash
XAI_KEY=$(grep XAI_API_KEY ~/.openclaw/workspace/.env.xai | cut -d= -f2)

RESP=$(curl -s --max-time 120 -X POST "https://api.x.ai/v1/videos/generations" \
  -H "Authorization: Bearer $XAI_KEY" \
  -H "Content-Type: application/json" \
  -d @grok-payload.json)

REQ_ID=$(echo "$RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['request_id'])")
```

### Poll for completion (Python requests is OK for small responses)

```python
import requests, time

headers = {"Authorization": f"Bearer {XAI_KEY}"}

while True:
    time.sleep(10)
    resp = requests.get(f"https://api.x.ai/v1/videos/{req_id}", headers=headers, timeout=30)
    data = resp.json()
    if data["status"] == "done":
        video_url = data["video"]["url"]
        # Download with curl -L
        break
    elif data["status"] == "failed":
        # Handle failure
        break
```

### Download video

```bash
curl -s -L --max-time 120 "$VIDEO_URL" -o broll-video.mp4
```

---

## Compound Segment Creation

### Extract A-roll segment (MUST re-encode)

```bash
ffmpeg -y -ss $START -t $DUR -i "$WORKDIR/a-roll-full.mp4" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k \
  "$WORKDIR/work/aroll-segments/seg-$SEG.mp4"
```

### Extract frames

```bash
ffmpeg -y -i "$WORKDIR/work/aroll-segments/seg-$SEG.mp4" -vsync 0 \
  "$WORKDIR/work/seg${SEG}-frames/frame_%05d.png"
```

### Remove background (Python)

```python
import glob, os
from rembg import remove, new_session

session = new_session("u2net")
frames = sorted(glob.glob("seg-frames/frame_*.png"))

for fp in frames:
    out = os.path.join("seg-nobg", os.path.basename(fp))
    if os.path.exists(out):
        continue
    with open(fp, "rb") as f:
        data = f.read()
    with open(out, "wb") as f:
        f.write(remove(data, session=session))
```

### Retime B-roll

```bash
AROLL_DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 aroll-segment.mp4)
BROLL_DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 broll-video.mp4)

ffmpeg -y -i broll-video.mp4 \
  -filter:v "setpts=${AROLL_DUR}/${BROLL_DUR}*PTS" \
  -an -r 25 \
  -c:v libx264 -preset fast -crf 18 \
  broll-retimed.mp4
```

### Composite overlay

```bash
ffmpeg -y \
  -i broll-retimed.mp4 \
  -framerate 25 -i seg-nobg/frame_%05d.png \
  -i aroll-segment.mp4 \
  -filter_complex "[1:v]scale=324:-1[av];[0:v][av]overlay=W-w:H-h:shortest=1[out]" \
  -map "[out]" -map "2:a" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  compound-segment.mp4
```

---

## Final Assembly

### Concatenate all compound segments

```bash
for i in $(seq -f "%02g" 1 $TOTAL); do
  echo "file 'compound-${i}.mp4'" >> concat-all.txt
done

ffmpeg -y -f concat -safe 0 -i concat-all.txt -c copy final-no-speed.mp4
```

### Apply speed boost

```bash
# For 25% speed boost:
ffmpeg -y -i final-no-speed.mp4 \
  -filter_complex "[0:v]setpts=0.8*PTS[v];[0:a]atempo=1.25[a]" \
  -map "[v]" -map "[a]" \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  final-sped-up.mp4
```

### Compress for Telegram (if over 50MB)

```bash
ffmpeg -y -i final-sped-up.mp4 \
  -c:v libx264 -preset fast -crf 26 \
  -c:a copy \
  -pix_fmt yuv420p -movflags +faststart \
  final-compressed.mp4
```

---

## Telegram Delivery

```python
import json, os, subprocess

config_path = os.path.expanduser('~/.openclaw/openclaw.json')
with open(config_path) as f:
    config = json.load(f)
token = config['channels']['telegram']['botToken']

subprocess.run([
    'curl', '-s', '-X', 'POST',
    f'https://api.telegram.org/bot{token}/sendVideo',
    '-F', f'chat_id={CHAT_ID}',
    '-F', f'video=@{VIDEO_PATH}',
    '-F', f'caption={CAPTION}',
    '-F', 'supports_streaming=true'
], capture_output=True, text=True)
```
