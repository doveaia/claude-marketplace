---
description: Knowledge about transcription tool selection, installation, and usage
---

# Transcription Backends Skill

## Purpose

Provides progressive disclosure of transcription tool knowledge, helping agents select, configure, and use the optimal transcription backend based on availability, performance requirements, and user preferences.

## Metadata

**Skill Type**: Technical Knowledge
**Applies To**: Audio-to-text transcription
**Context**: Local and cloud-based speech recognition tools

## Layer 1: Quick Reference

### Available Backends (Priority Order)

1. **whisper.cpp** - Fast, GPU-accelerated on Apple Silicon (STRONGLY RECOMMENDED)
2. **OpenAI Whisper (Python)** - Accurate but CPU-only, causes high load on Apple Silicon
3. **Whisper API** - Cloud, accurate, requires API key
4. **macOS Speech** - Built-in macOS, basic accuracy

### Quick Selection Decision Tree

```
On Apple Silicon (M1/M2/M3/M4)?
  ↓ Yes
  Have whisper.cpp installed? → Use it (base model) ✅ BEST CHOICE
    ↓ No
    Install whisper.cpp: brew install whisper-cpp → Use it

  ↓ No (Intel Mac or other)
Have whisper.cpp installed? → Use it (base model)
  ↓ No
Have Python Whisper installed? → Use it (base model ONLY, NOT medium/large)
  ↓ No
Have OPENAI_API_KEY set? → Use Whisper API
  ↓ No
On macOS? → Use macOS Speech (warn: no timestamps)
  ↓ No
ERROR: No backend available → Show installation guide
```

### ⚠️ Apple Silicon Warning

**DO NOT use OpenAI Whisper Python with medium/large models on Apple Silicon Macs!**
- Python Whisper does NOT use Metal/GPU acceleration
- Will cause 100% CPU usage and loud fan noise
- May trigger thermal throttling
- whisper.cpp is 3-10x faster on Apple Silicon

## Layer 2: Installation & Setup

### whisper.cpp (RECOMMENDED for Apple Silicon)

**Installation on macOS (Apple Silicon M1/M2/M3/M4)**:
```bash
# Install via Homebrew (includes Metal GPU support automatically)
brew install whisper-cpp

# The Homebrew version includes Metal support out of the box
# No additional configuration needed for GPU acceleration
```

**Installation on macOS (with CoreML for Neural Engine - fastest)**:
```bash
# Clone and build with CoreML support
git clone https://github.com/ggml-org/whisper.cpp
cd whisper.cpp

# Install Python dependencies for CoreML
pip install ane_transformers openai-whisper coremltools

# Generate CoreML model (one-time, ~5 minutes)
./models/generate-coreml-model.sh base.en

# Build with CoreML enabled
cmake -B build -DWHISPER_COREML=1
cmake --build build -j --config Release
```

**Installation on Linux**:
```bash
git clone https://github.com/ggml-org/whisper.cpp
cd whisper.cpp
cmake -B build
cmake --build build -j --config Release

# Download model
./models/download-ggml-model.sh base
```

**Download Models**:
```bash
# Using whisper.cpp built-in downloader
whisper-cpp --download-model base

# Or manually download to cache directory
mkdir -p ~/.cache/whisper
cd ~/.cache/whisper
curl -LO https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin
```

**Verification**:
```bash
# Check installation
which whisper-cpp || brew list whisper-cpp 2>/dev/null
echo "✓ Installed"

# Verify Metal support (Apple Silicon)
whisper-cpp --help | grep -i metal && echo "✓ Metal GPU support available"
```

**Basic Usage**:
```bash
# With Homebrew installation (model auto-downloaded)
whisper-cpp -m base -f <audio-file> --output-vtt

# With manual model path
whisper-cpp -m ~/.cache/whisper/ggml-base.bin -f <audio-file> --output-vtt

# Multi-threaded for faster processing
whisper-cpp -m base -f <audio-file> --output-vtt --threads 4
```

### OpenAI Whisper (Python) - ⚠️ CAUTION ON APPLE SILICON

**⚠️ WARNING for Apple Silicon (M1/M2/M3/M4) Users**:
- OpenAI Whisper Python does NOT use GPU/Metal acceleration
- Will cause 100% CPU usage and loud fan noise
- Use whisper.cpp instead for 3-10x better performance
- If you must use Python Whisper, use `base` model only (NOT medium/large)

**Installation**:
```bash
pip install openai-whisper

# Optional: ffmpeg for audio conversion
brew install ffmpeg  # macOS
apt install ffmpeg   # Linux
```

**Verification**:
```bash
python3 -c "import whisper; print('✓ Installed')"
```

**Basic Usage** (base model only on Apple Silicon!):
```bash
# SAFE - base model
whisper <audio-file> --model base --output_format vtt

# ⚠️ AVOID on Apple Silicon - will cause high CPU load:
# whisper <audio-file> --model medium  ← SLOW, loud fans
# whisper <audio-file> --model large   ← EXTREMELY SLOW
```

**SSL Certificate Issues** (common error):
```bash
# If you get SSL errors when downloading models, try:
pip install --upgrade certifi
export SSL_CERT_FILE=$(python3 -c "import certifi; print(certifi.where())")

# Or use whisper.cpp instead (no SSL issues)
```

### Whisper API

**Setup**:
```bash
# Set API key (get from platform.openai.com)
export OPENAI_API_KEY='sk-...'

# Add to ~/.zshrc or ~/.bashrc for persistence
echo 'export OPENAI_API_KEY="sk-..."' >> ~/.zshrc
```

**Verification**:
```bash
test -n "$OPENAI_API_KEY" && echo "✓ API key set"
```

**Basic Usage**:
```bash
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F file="@<audio-file>" \
  -F model="whisper-1" \
  -F response_format="verbose_json"
```

### macOS Speech Framework

**Setup**: No installation required on macOS

**Verification**:
```bash
sw_vers | grep macOS && echo "✓ Available"
```

**Usage**: Limited to short clips, no direct timestamp support

## Layer 3: Advanced Configuration

### Model Selection (Whisper-based)

| Model | Size | Speed | Accuracy | Use Case | Apple Silicon Notes |
|-------|------|-------|----------|----------|---------------------|
| tiny | 75 MB | Very Fast | Fair | Quick previews, testing | ✅ Safe |
| base | 145 MB | Fast | Good | **Default choice** | ✅ Recommended |
| small | 466 MB | Medium | Very Good | High accuracy needed | ✅ OK with whisper.cpp |
| medium | 1.5 GB | Slow | Excellent | Professional transcription | ⚠️ Only with whisper.cpp |
| large | 3 GB | Very Slow | Best | Maximum accuracy | ⚠️ Only with whisper.cpp |

**Recommendation**: Use `base` for most cases. Only upgrade to `small` or larger if accuracy is critical and time is not a constraint.

**⚠️ Apple Silicon Warning**:
- `medium` and `large` models with Python Whisper will cause:
  - 100% CPU usage on all cores
  - Loud fan noise
  - 30+ minutes for a 10-minute audio file
  - Potential thermal throttling
- Use `base` model with Python Whisper, or switch to whisper.cpp for larger models

### Language Specification

```bash
# Auto-detect (default)
whisper-cpp --language auto <audio-file>

# Specify language for better accuracy
whisper-cpp --language en <audio-file>    # English
whisper-cpp --language es <audio-file>    # Spanish
whisper-cpp --language fr <audio-file>    # French
whisper-cpp --language de <audio-file>    # German
whisper-cpp --language ja <audio-file>    # Japanese
```

### Timestamp Options

```bash
# VTT format (subtitles with timestamps)
whisper-cpp --output-vtt <audio-file>

# SRT format
whisper-cpp --output-srt <audio-file>

# Plain text (no timestamps)
whisper-cpp --output-txt <audio-file>

# JSON with word-level timestamps
whisper <audio-file> --output_format json
```

### Audio Format Handling

**Supported Formats**: WAV, MP3, M4A, OGG, FLAC, OPUS

**Conversion** (if needed):
```bash
ffmpeg -i input.m4a -ar 16000 -ac 1 output.wav
```

**Optimal Format**: 16kHz mono WAV (best compatibility and quality)

## Layer 4: Error Handling & Optimization

### Common Errors & Solutions

#### Out of Memory

**Symptom**: Process killed or OOM error

**Solutions**:
1. Use smaller model (`tiny` or `base`)
2. Process in chunks (split audio into segments)
3. Reduce audio sample rate to 16kHz
4. Close other applications

```bash
# Chunk processing example
ffmpeg -i long-audio.m4a -f segment -segment_time 300 -c copy chunk%03d.m4a
for chunk in chunk*.m4a; do
  whisper-cpp --model base "$chunk"
done
```

#### Language Detection Failure

**Symptom**: Gibberish output or wrong language

**Solution**: Explicitly specify language
```bash
whisper-cpp --language en <audio-file>
```

#### Slow Processing

**Symptom**: Taking too long (> 2x audio duration)

**Solutions**:
1. Use `whisper.cpp` instead of Python version (3-5x faster)
2. Use smaller model (`tiny` or `base`)
3. Enable GPU acceleration (if available)
4. Use Whisper API (cloud processing)

#### Audio Format Error

**Symptom**: "Unsupported format" or decode error

**Solution**: Convert to WAV
```bash
ffmpeg -i input.m4a -ar 16000 -ac 1 output.wav
whisper-cpp --model base output.wav
```

### Performance Optimization

**whisper.cpp Optimizations**:
```bash
# Use threading (faster on multi-core)
whisper-cpp --threads 4 --model base <audio-file>

# Use GPU (if compiled with CUDA/Metal)
whisper-cpp --use-gpu --model base <audio-file>

# Faster processing (less accurate)
whisper-cpp --model tiny --beam-size 1 <audio-file>
```

**Python Whisper Optimizations**:
```bash
# Use GPU
whisper <audio-file> --device cuda --model base

# FP16 precision (faster, slight accuracy loss)
whisper <audio-file> --fp16 True --model base
```

### Quality Tuning

**Improve Accuracy**:
- Use larger model (`small`, `medium`, `large`)
- Specify language explicitly
- Use `--beam-size 5` for better results (slower)
- Pre-process audio (noise reduction, normalization)

**Improve Speed**:
- Use `whisper.cpp` with `tiny` or `base` model
- Reduce beam size: `--beam-size 1`
- Use FP16: `--fp16 True`
- Process on GPU if available

## Layer 5: Backend Comparison & Selection Logic

### Feature Matrix

| Feature | whisper.cpp | Python Whisper | Whisper API | macOS Speech |
|---------|-------------|----------------|-------------|--------------|
| Speed | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Speed (Apple Silicon) | ⭐⭐⭐⭐⭐ | ⭐ (CPU only) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Accuracy | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Privacy | ✅ Local | ✅ Local | ❌ Cloud | ✅ Local |
| Cost | Free | Free | $0.006/min | Free |
| Setup | Easy | Medium | Easy | Built-in |
| Timestamps | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| Languages | 99+ | 99+ | 99+ | ~50 |
| Offline | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| GPU (Apple Silicon) | ✅ Metal | ❌ No | N/A | ✅ Native |
| Neural Engine | ✅ CoreML | ❌ No | N/A | ✅ Native |
| Memory Usage | ~2GB | ~3-4GB | N/A | Low |

### Use Case Recommendations

**Apple Silicon Mac (M1/M2/M3/M4) - ALWAYS USE whisper.cpp**:
- **whisper.cpp** with `base` model is the ONLY recommended option
- 3-10x faster than Python Whisper due to Metal/CoreML acceleration
- Lower memory usage and no fan noise
- Install: `brew install whisper-cpp`

**Personal Use / Default**:
- **whisper.cpp** with `base` model
- Best balance of speed, accuracy, and ease of use

**Professional / High Accuracy**:
- **whisper.cpp** with `small` or `medium` model (with CoreML on Apple Silicon)
- Or **Whisper API** for maximum accuracy without local processing
- **AVOID Python Whisper** with large models on Apple Silicon

**Quick Testing / Previews**:
- **whisper.cpp** with `tiny` model
- Or **Whisper API** for instant results

**Privacy-Critical**:
- **whisper.cpp** (local only, GPU-accelerated)
- Never use Whisper API

**No Installation / Quick Start**:
- **Whisper API** (if API key available)
- Or **macOS Speech** (macOS only, limited features)

**Intel Mac / Linux with NVIDIA GPU**:
- **Python Whisper** can use CUDA acceleration (faster than CPU)
- Or **whisper.cpp** with CUDA build

## Progressive Disclosure Application

**When to use which layer**:

- **Layer 1**: Agent needs to quickly select backend
- **Layer 2**: Backend not installed, need installation instructions
- **Layer 3**: User wants to customize model or language
- **Layer 4**: Transcription failing or too slow
- **Layer 5**: User comparing options or making strategic choice

**Default Behavior**: Start with Layer 1, escalate only when needed.
