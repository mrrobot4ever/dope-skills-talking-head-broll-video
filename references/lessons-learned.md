# Lessons Learned

Hard-won lessons from the first production run (2026-04-16).

## Audio Splitting

**Problem:** Splitting audio at an arbitrary midpoint truncated a word, causing the avatar to generate with a cut-off syllable at the chunk boundary.

**Fix:** Use `ffmpeg silencedetect` to find silence gaps near the optimal split point. Split at the midpoint of the silence gap, not at a fixed time offset.

## A-Roll Segment Extraction: `-c copy` vs Re-encode

**Problem:** Using `ffmpeg -ss $START -t $DUR -c copy` to extract segments cuts on keyframes, not at the exact requested timestamp. Each segment came out ~0.08-0.2s longer than planned. Over 28 segments, this compounded to +2.24 seconds of cumulative drift, causing severe audio/video lip sync desync.

**Fix:** Always re-encode when extracting segments:
```bash
ffmpeg -y -ss $START -t $DUR -i input.mp4 \
  -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k \
  output.mp4
```
This is slower but gives frame-accurate cuts with zero drift.

## Transparency: WebM Alpha Flattens to Black

**Problem:** Created an intermediate WebM VP9 alpha video of the transparent avatar, then used it in ffmpeg's overlay filter to composite onto B-roll. The alpha channel was flattened to black in the MP4 output because H.264 doesn't support alpha -- ffmpeg drops the alpha before compositing.

**Fix:** Skip the intermediate WebM entirely. Feed PNG frames (which natively retain alpha) directly into the ffmpeg overlay filter:
```bash
ffmpeg -y \
  -i broll-retimed.mp4 \
  -framerate 25 -i nobg-frames/frame_%05d.png \  # PNG alpha preserved
  -i aroll-segment.mp4 \
  -filter_complex "[1:v]scale=324:-1[av];[0:v][av]overlay=W-w:H-h:shortest=1[out]" \
  -map "[out]" -map "2:a" \
  ...
```

## Grok API SSL Errors

**Problem:** Python 3.9's system LibreSSL (2.8.3) fails with `SSLV3_ALERT_BAD_RECORD_MAC` when sending large base64 image payloads via `requests.post()`.

**Fix:** Build the JSON payload with Python (to handle base64 encoding), write it to a file, then submit with curl:
```bash
curl -s --max-time 120 -X POST "https://api.x.ai/v1/videos/generations" \
  -H "Authorization: Bearer $XAI_KEY" \
  -H "Content-Type: application/json" \
  -d @payload.json
```
Polling with Python requests works fine (small response payloads).

## Gemini Safety Filter

**Problem:** Gemini blocked an image generation request due to `IMAGE_SAFETY` policy violation when the prompt referenced "spicy sites" (adult content context).

**Fix:** Rephrase prompts to avoid suggestive language. Use neutral descriptions like "person browsing a website late at night" instead of explicitly referencing adult content.

## Image Compression for Grok

**Problem:** Grok Imagine API accepts base64 images but large payloads cause SSL errors.

**Fix:** Compress images aggressively before upload: resize to 360x640, JPEG quality 40, target under 100KB. The video generation quality is not affected by input image compression.

## Creatify Audio Limit

**Problem:** Creatify Aurora has a strict 60-second audio limit. Files at exactly 60.000s or microseconds over get rejected.

**Fix:** Always use 59s or less per chunk. Re-encode (never stream copy) to ensure exact duration. Stream copy can produce files that are 60.003s due to MP3 frame boundaries.

## ffmpeg Filter Availability

**Problem:** The brew-installed ffmpeg on Mac Mini M4 does not include `subtitles`, `ass`, or `drawtext` filters.

**Fix:** For subtitle/caption rendering, use Python PIL to burn text onto frames directly, then reassemble with ffmpeg. This is slower but works regardless of ffmpeg build configuration.
