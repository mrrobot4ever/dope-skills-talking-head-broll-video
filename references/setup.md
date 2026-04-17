# Setup Guide

## System Requirements

- macOS or Linux (tested on Mac Mini M4, 16GB RAM)
- Python 3.9+
- ffmpeg with libx264, libmp3lame, aac encoders
- Minimum 4GB free RAM for rembg background removal

## Tool Installation

### ffmpeg
```bash
brew install ffmpeg
```

### whisper.cpp (audio transcription)
```bash
brew install whisper-cpp
# Download model
mkdir -p models/whisper
curl -L -o models/whisper/ggml-base.en.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

### rembg (background removal)
```bash
pip3 install "rembg[cpu]"
# u2net model (~176MB) auto-downloads on first run
```

### PIL / Pillow (image processing)
```bash
pip3 install Pillow
```

## API Keys

### Creatify (Avatar Generation)
1. Sign up at https://creatify.ai
2. Get API ID and API Key from dashboard
3. Create credentials file:
```bash
cat > .env.creatify << EOF
CREATIFY_API_ID=your_api_id
CREATIFY_API_KEY=your_api_key
EOF
```

### Google Gemini (Image Generation)
1. Get API key from https://aistudio.google.com/apikey
2. Create credentials file:
```bash
cat > .env.google << EOF
GOOGLE_AI_API_KEY=your_api_key
EOF
```

### xAI Grok (Video Generation)
1. Get API key from https://console.x.ai
2. Create credentials file:
```bash
cat > .env.xai << EOF
XAI_API_KEY=your_api_key
EOF
```

## GitHub CLI (for temporary file hosting)

Creatify needs files at public URLs. GitHub Release assets work as temporary hosting.

```bash
brew install gh
gh auth login
```

You need at least one public repository to create temporary releases for file uploads. After avatar generation completes, delete the temporary release.

## Telegram Bot (for delivery)

If delivering via Telegram:
- Bot token in OpenClaw config: `~/.openclaw/openclaw.json` → `channels.telegram.botToken`
- Max send size: 50MB (compress with higher CRF if exceeded)
- Max receive size: 20MB

## Verification

Run these to verify everything is installed:
```bash
ffmpeg -version | head -1
whisper-cli --help 2>&1 | head -1
python3 -c "from rembg import remove; print('rembg OK')"
python3 -c "from PIL import Image; print('Pillow OK')"
gh --version
```

## Cost Estimates

Per video (assuming ~105s audio, 28 segments):
- **Creatify Aurora:** ~70-140 credits (2 chunks × 10-20 credits/15s)
- **Gemini image gen:** Free (Nano Banana 2 is free tier)
- **Grok video gen:** Free (no apparent rate limits)
- **Total API cost:** Creatify credits only
