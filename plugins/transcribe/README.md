# Transcribe Plugin

Transcribe YouTube videos and local video/audio files with multiple transcription backend support.

## Overview

The `transcribe` plugin provides a complete workflow for transcribing media to searchable text with timestamps. It supports two input sources:
1. **YouTube videos** - Downloads audio using yt-dlp
2. **Local files** - Processes video/audio files from your filesystem

Supports multiple transcription backends and handles errors gracefully with helpful guidance.

## Features

- **Dual Input Support**:
  - YouTube videos (any URL format)
  - Local video files (MP4, MKV, AVI, MOV, WebM, etc.)
  - Local audio files (MP3, M4A, WAV, FLAC, OGG, etc.)
- **Language Preservation & Translation** (v2.6.0):
  - Transcripts stay in the video's ORIGINAL language (French stays French)
  - Non-English videos get TWO files with language code suffixes:
    - `<filename>-<lang>.md` (e.g., `video-fr.md` for French)
    - `<filename>-en.md` (English translation)
  - English videos get ONE file: `<filename>.md` (no suffix)
  - Automatic language detection before transcription
- Automatically transcribes audio to text using available transcription tools
- Supports multiple transcription backends (whisper.cpp, OpenAI Whisper, Whisper API, macOS Speech)
- Includes timestamps and metadata in transcripts
- Automatic filename sanitization (no spaces, clean special characters)
- Automatic cleanup of temporary files
- Handles errors gracefully with helpful guidance
- Automatic backend detection and fallback
- Formats output in clean, readable markdown

## Installation

### Required Dependencies

#### 1. yt-dlp (Required for YouTube)

Downloads audio from YouTube videos. **Only needed if you plan to transcribe YouTube videos.**

```bash
# macOS
brew install yt-dlp

# Linux/Windows
pip install yt-dlp
```

Verify installation:
```bash
yt-dlp --version
```

#### 2. ffmpeg (Required for Local Video Files)

Extracts audio from local video files. **Only needed if you plan to transcribe local video files (not needed for audio files).**

```bash
# macOS
brew install ffmpeg

# Linux
apt install ffmpeg  # Debian/Ubuntu
yum install ffmpeg  # Red Hat/CentOS

# Windows
# Download from https://ffmpeg.org
```

Verify installation:
```bash
ffmpeg -version
```

#### 3. Transcription Backend (Choose One - Required for All)

You need at least one transcription backend installed.

---

### ⚠️ IMPORTANT: Apple Silicon Mac Users (M1/M2/M3/M4)

**If you have an Apple Silicon Mac, ONLY use whisper.cpp!**

OpenAI Whisper (Python) does NOT use GPU/Metal acceleration on Apple Silicon and will:
- Cause 100% CPU usage on all cores
- Make your fans spin loudly
- Take 5-10x longer than whisper.cpp
- Potentially trigger thermal throttling

**whisper.cpp uses Metal GPU acceleration** and is 3-10x faster with lower resource usage.

---

**Option A: whisper.cpp (STRONGLY RECOMMENDED for Apple Silicon)**

Fast, GPU-accelerated on Apple Silicon, efficient.

```bash
# macOS (includes Metal support automatically)
brew install whisper-cpp

# The model will be downloaded automatically on first use
# Or manually download a model:
whisper-cpp --download-model base
```

Verify:
```bash
which whisper-cpp || brew list whisper-cpp 2>/dev/null
```

**For even faster performance on Apple Silicon (CoreML support)**:
```bash
# Clone and build with CoreML for Neural Engine acceleration
git clone https://github.com/ggml-org/whisper.cpp
cd whisper.cpp

# Install dependencies
pip install ane_transformers openai-whisper coremltools

# Generate CoreML model (one-time, ~5 minutes)
./models/generate-coreml-model.sh base.en

# Build with CoreML
cmake -B build -DWHISPER_COREML=1
cmake --build build -j --config Release
```

**Option B: OpenAI Whisper (Python) - ⚠️ CAUTION ON APPLE SILICON**

Most accurate, but CPU-only on Apple Silicon (very slow).

```bash
pip install openai-whisper

# Optional: Install ffmpeg for audio conversion
brew install ffmpeg  # macOS
apt install ffmpeg   # Linux
```

Verify:
```bash
python3 -c "import whisper; print('Installed')"
```

**⚠️ WARNING for Apple Silicon**:
- Use `base` model only (NOT medium or large)
- `medium` model will take 30+ minutes for a 10-minute audio
- Consider switching to whisper.cpp instead

**SSL Certificate Issues** (common error):
```bash
# If you get SSL errors when downloading models:
pip install --upgrade certifi
export SSL_CERT_FILE=$(python3 -c "import certifi; print(certifi.where())")
```

**Option C: Whisper API**

Cloud-based, requires API key.

```bash
# Get API key from https://platform.openai.com/api-keys
export OPENAI_API_KEY='sk-...'

# Add to shell config for persistence
echo 'export OPENAI_API_KEY="sk-..."' >> ~/.zshrc
```

**Option D: macOS Speech Framework**

Built into macOS, no installation needed (limited features).

## Usage

### YouTube Videos

```bash
/transcribe <youtube-url>
```

**Example**:
```bash
/transcribe https://youtube.com/watch?v=dQw4w9WgXcQ
```

**What happens**:
1. Creates `.youtube/downloads/` for temporary files
2. Downloads audio from YouTube to working directory
3. Transcribes using the best available backend
4. Creates markdown file with transcript and metadata
5. Sanitizes filename (removes spaces, special characters)
6. Saves to `./transcripts/` with clean filename
7. Automatically deletes temporary files

### Local Video Files

```bash
/transcribe <path-to-video>
```

**Examples**:
```bash
/transcribe ./videos/meeting-recording.mp4
/transcribe ~/Downloads/conference-talk.mkv
```

**What happens**:
1. Creates working directory for temporary files
2. Extracts audio from video using ffmpeg
3. Transcribes the extracted audio
4. Creates markdown file with transcript
5. Saves to `./transcripts/`
6. Cleans up temporary audio

### Local Audio Files

```bash
/transcribe <path-to-audio>
```

**Examples**:
```bash
/transcribe ./audio/podcast-episode.mp3
/transcribe ~/Music/interview.m4a
```

**What happens**:
1. Copies audio to working directory
2. Transcribes directly (no extraction needed)
3. Creates markdown transcript
4. Saves to `./transcripts/`

### Advanced Options

#### Specify Transcription Backend

```bash
/transcribe <input> --backend whisper.cpp
/transcribe <input> --backend whisper
/transcribe <input> --backend api
```

#### Custom Output Location

```bash
/transcribe <url> --output ./youtube/research/
```

#### Specify Language

```bash
/transcribe <url> --language es  # Spanish
/transcribe <url> --language fr  # French
/transcribe <url> --language ja  # Japanese
```

#### Disable Timestamps

```bash
/transcribe <url> --no-timestamps
```

#### Choose Whisper Model Size

```bash
/transcribe <url> --model tiny    # Fastest, less accurate
/transcribe <url> --model base    # Default, balanced
/transcribe <url> --model small   # Slower, more accurate
/transcribe <url> --model medium  # Much slower, very accurate
```

## Output Format

Transcripts are saved as markdown files with this structure:

```markdown
# Video Title Here

**Channel**: Channel Name
**Duration**: 15:42
**Published**: 2024-03-15
**Views**: 12,543
**URL**: https://youtube.com/watch?v=dQw4w9WgXcQ
**Transcribed**: 2024-03-20 10:30:00 UTC
**Transcription Backend**: whisper.cpp (base model)

---

## Transcript

[00:00:00] Introduction and overview of the topic...
[00:00:15] Explaining the first main point here...
[00:01:30] Diving deeper into technical details...
[00:03:00] Demonstrating the practical example...

---

## Metadata

- Video ID: dQw4w9WgXcQ
- Categories: Education, Technology
- Tags: programming, tutorial, golang
- Upload Date: 2024-03-15
```

## Transcription Backends

The plugin automatically detects and uses the best available backend in this priority order:

1. **whisper.cpp** - Fast, GPU-accelerated on Apple Silicon (STRONGLY RECOMMENDED)
2. **OpenAI Whisper (Python)** - Accurate, but CPU-only on Apple Silicon (slow, high load)
3. **Whisper API** - Cloud-based, accurate, requires API key
4. **macOS Speech** - Built-in macOS, basic accuracy (fallback only)

### Backend Comparison

| Backend | Speed | Speed (Apple Silicon) | Accuracy | Privacy | Cost | GPU Support |
|---------|-------|----------------------|----------|---------|------|-------------|
| whisper.cpp | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Local | Free | ✅ Metal |
| Python Whisper | ⭐⭐⭐ | ⭐ (CPU only) | ⭐⭐⭐⭐⭐ | Local | Free | ❌ No |
| Whisper API | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Cloud | $0.006/min | N/A |
| macOS Speech | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | Local | Free | ✅ Native |

**Apple Silicon Note**: whisper.cpp is 3-10x faster than Python Whisper on M1/M2/M3/M4 Macs due to Metal GPU acceleration. Python Whisper does NOT use the GPU on Apple Silicon.

## Default Output Location

Transcripts are saved to `./youtube/transcripts/` by default.

The hidden folder (`.youtube`) keeps transcripts organized without cluttering your workspace. The directory is created automatically if it doesn't exist.

**Custom Location**: Use the `--output` option to save transcripts elsewhere:
```bash
/transcribe <url> --output ./newsletter/research/
```

## File Management

### Directory Structure

```
.youtube/
├── downloads/           # Temporary working directory
│   ├── *.m4a           # Audio files (auto-deleted)
│   ├── *.info.json     # Video metadata (auto-deleted)
│   └── transcript*.txt # Intermediate transcripts (auto-deleted)
└── transcripts/         # Final transcripts (permanent)
    └── Video-Title-VIDEO_ID.md
```

### Automatic Cleanup

After successful transcription, temporary files are **automatically deleted**:
- Audio files (`.m4a`, `.wav`, etc.)
- Metadata JSON files (`.info.json`)
- Intermediate transcript files (`transcript.txt`, `transcript_timestamped.txt`)

**Only the final formatted transcript** is kept in `./youtube/transcripts/`.

### Filename Sanitization

Filenames are automatically cleaned to avoid filesystem issues:

| Original | Sanitized |
|----------|-----------|
| `"Full Tutorial: Build AI ｜ Claude"` | `Full-Tutorial-Build-AI-Claude-VIDEO_ID.md` |
| `"React/Next.js Tutorial - Part 1"` | `React-Next-js-Tutorial-Part-1-VIDEO_ID.md` |
| `"Episode #42: Advanced TypeScript?"` | `Episode-42-Advanced-TypeScript-VIDEO_ID.md` |

**Rules**:
- Spaces → Hyphens (`-`)
- Special characters (`:`, `｜`, `/`, `\`, `*`, `?`, `"`, `<`, `>`) → Removed or replaced
- Multiple consecutive hyphens → Single hyphen
- Length limited to 200 characters

### Manual Cleanup

If transcription fails or is interrupted, temporary files may remain. To clean up manually:

```bash
rm -rf .youtube/downloads/*
```

## Architecture

### Command Structure

- **Command**: `/transcribe` - Entry point, argument parsing, orchestration
- **Agents**:
  - `video-downloader` - Handles yt-dlp operations, URL validation, audio download
  - `audio-transcriber` - Manages transcription backends, format conversion, output
- **Skills**:
  - `transcription-backends` - Progressive knowledge about transcription tools
  - `youtube-metadata` - Knowledge about YouTube metadata extraction and formatting

### Agent Orchestration

```
/transcribe <url>
       ↓
  [Parse arguments]
       ↓
  video-downloader agent
    • Validate URL
    • Check yt-dlp
    • Download audio
    • Extract metadata
       ↓
  audio-transcriber agent
    • Detect backends
    • Select optimal backend
    • Transcribe with timestamps
    • Format output
       ↓
  [Save transcript]
  [Display summary]
```

### Model Selection

- **Default Model**: Haiku (fast operations, efficient for orchestration)
- **video-downloader**: Haiku (simple bash operations)
- **audio-transcriber**: Haiku (straightforward tool invocation)

Haiku is sufficient for this workflow as it primarily involves system command orchestration rather than complex reasoning.

## Examples

### Example 1: Basic Transcription

```bash
/transcribe https://youtube.com/watch?v=dQw4w9WgXcQ
```

**Output**:
```
Downloading audio from YouTube...
✓ Downloaded: Rick Astley - Never Gonna Give You Up
✓ Duration: 03:33
✓ Size: 4.2 MB

Transcribing with whisper.cpp (base model)...
✓ Transcribed 213 seconds
✓ Generated 387 words

Saved transcript to: ./youtube/transcripts/Rick-Astley-Never-Gonna-Give-You-Up-dQw4w9WgXcQ.md
✓ Temporary files cleaned up
```

**Files created/deleted**:
- ✓ Created: `./youtube/transcripts/Rick-Astley-Never-Gonna-Give-You-Up-dQw4w9WgXcQ.md`
- 🗑️ Deleted: `.youtube/downloads/Rick Astley - Never Gonna Give You Up-dQw4w9WgXcQ.m4a`
- 🗑️ Deleted: `.youtube/downloads/Rick Astley - Never Gonna Give You Up-dQw4w9WgXcQ.info.json`
- 🗑️ Deleted: `.youtube/downloads/transcript_timestamped.txt`

### Example 2: Custom Output Location

```bash
/transcribe https://youtube.com/watch?v=xxxxx --output ./newsletter/research/
```

Saves to custom directory instead of default `./youtube/transcripts/`.

### Example 3: High-Accuracy Transcription

```bash
/transcribe https://youtube.com/watch?v=xxxxx --backend whisper --model medium
```

Uses Python Whisper with medium model for maximum accuracy.

### Example 4: Non-English Content

```bash
/transcribe https://youtube.com/watch?v=xxxxx --language es
```

Transcribes Spanish video with language-specific optimization.

## Troubleshooting

### Error: yt-dlp not found

```
Error: yt-dlp not found

To install:
  macOS:    brew install yt-dlp
  Linux:    pip install yt-dlp
  Windows:  pip install yt-dlp
```

**Solution**: Install yt-dlp following the instructions above.

### Error: No transcription backend available

```
Error: No transcription backend available

Please install one of the following:

1. whisper.cpp (recommended)
   macOS: brew install whisper-cpp

2. OpenAI Whisper
   pip install openai-whisper

3. Whisper API
   export OPENAI_API_KEY='sk-...'
```

**Solution**: Install at least one transcription backend.

### Error: Video unavailable

```
Error: Video unavailable

Possible reasons:
  - Video is private
  - Video has been deleted
  - Video is region-restricted
```

**Solution**: Verify the URL is correct and the video is publicly accessible.

### Transcription is too slow / Fan spinning loudly (Apple Silicon)

**Most likely cause**: You're using OpenAI Whisper Python, which does NOT use GPU on Apple Silicon.

**Solution 1**: Install and use whisper.cpp (RECOMMENDED for Apple Silicon)
```bash
# Install whisper.cpp
brew install whisper-cpp

# Use it for transcription
/transcribe <url> --backend whisper.cpp
```

**Solution 2**: Use smaller model if stuck with Python Whisper
```bash
# Use base model (NOT medium or large!)
/transcribe <url> --model base
```

**Solution 3**: Use Whisper API (cloud processing)
```bash
export OPENAI_API_KEY='sk-...'
/transcribe <url> --backend api
```

### SSL Certificate Error (Python Whisper)

```
Traceback (most recent call last):
  File ".../urllib/request.py", line 1348, in do_open
    h.request(...)
ssl.SSLCertVerificationError: ...
```

**Solution 1**: Update certificates
```bash
pip install --upgrade certifi
export SSL_CERT_FILE=$(python3 -c "import certifi; print(certifi.where())")
```

**Solution 2**: Use whisper.cpp instead (no SSL issues, models downloaded locally)
```bash
brew install whisper-cpp
/transcribe <url> --backend whisper.cpp
```

### Out of memory error

**Solution**: Use smaller model
```bash
/transcribe <url> --model tiny
```

or

```bash
/transcribe <url> --model base
```

### High CPU usage / Thermal throttling (Apple Silicon)

**Cause**: OpenAI Whisper Python uses CPU-only processing on Apple Silicon.

**Solution**: Switch to whisper.cpp which uses Metal GPU acceleration
```bash
brew install whisper-cpp
/transcribe <url> --backend whisper.cpp
```

## Development

### Plugin Structure

```
plugins/transcribe/
├── .claude-plugin/
│   └── plugin.json                       # Plugin metadata
├── agents/
│   ├── video-downloader.md               # Audio download agent
│   └── audio-transcriber.md              # Transcription agent
├── skills/
│   ├── transcribe/
│   │   └── SKILL.md                      # /transcribe:transcribe slash command
│   ├── transcription-backends/
│   │   └── SKILL.md                      # Transcription knowledge
│   └── youtube-metadata/
│       └── SKILL.md                      # YouTube metadata knowledge
└── README.md                             # This file
```

### Testing

Test with various video types:

```bash
# Short video (< 5 minutes)
/transcribe <short-url>

# Long video (> 30 minutes)
/transcribe <long-url>

# Non-English video
/transcribe <non-english-url> --language es

# Educational content
/transcribe <tutorial-url>

# Music video (test speech detection)
/transcribe <music-url>
```

## Contributing

Contributions are welcome! Please ensure:

1. All agents follow the standard agent structure
2. Skills use progressive disclosure pattern
3. Error messages are helpful and actionable
4. Documentation is comprehensive and clear

## License

Part of the claude-code-plugins marketplace (doveaia).

## Support

For issues or questions:
- Check troubleshooting section above
- Review agent and skill documentation
- Open an issue in the marketplace repository

## Updating the Plugin

If you've already installed this plugin and want to update to the latest version:

### Method 1: Reinstall from Marketplace

If installed via marketplace:
```bash
# Uninstall current version
claude plugin uninstall transcribe

# Reinstall latest version
claude plugin install
# Select transcribe from the list
```

### Method 2: Update Local Installation

If installed from local directory:
```bash
# Navigate to the marketplace directory
cd /path/to/claude-code-plugins

# Pull latest changes (if using git)
git pull

# Plugin will automatically use the updated version
```

### Method 3: Manual Update

```bash
# Remove old plugin
rm -rf ~/.claude/plugins/transcribe

# Install from updated local directory
claude plugin install /path/to/claude-code-plugins/plugins/transcribe
```

**Note**: After updating, verify the new version:
```bash
claude plugin list
```

## Version History

### 3.0.0 (Current)
- **BREAKING**: Migrated all slash commands from legacy `commands/*.md` to `skills/<name>/SKILL.md` format
- Command invocation names are unchanged for end users (e.g. `/transcribe:<command>`)
- Removed legacy `commands/` directory

### 2.6.0
- **NEW**: Language code suffixes for non-English video output files
  - Original language: `<filename>-<lang>.md` (e.g., `video-fr.md`)
  - English translation: `<filename>-en.md`
  - English videos unchanged: `<filename>.md` (no suffix)
- **FIXED**: Both files now properly created for non-English content

### 2.5.1
- **CRITICAL FIX**: Updated whisper.cpp binary name from `whisper-cpp` to `whisper-cli`
  - The Homebrew `whisper-cpp` package now provides `whisper-cli` as the binary
  - All command examples and detection logic updated
- **FIXED**: Model path handling - now uses full path (`-m "$MODELS_DIR/ggml-medium.bin"`)
  - Models located at `$(brew --prefix whisper-cpp)/share/whisper-cpp/models/`
  - Falls back to `~/.cache/whisper/` if brew models not found
- **FIXED**: Added `-f` flag for input audio file (required by whisper-cli)

### 2.5.0
- **NEW**: Two-file output for non-English videos
  - Primary file: Transcript in ORIGINAL language (French stays French)
  - Secondary file: English translation (`<filename>-english.md`)
- **FIXED**: Improved language detection parsing
  - Correctly extracts language code from whisper-cpp output
  - Handles multiple output formats
- **CHANGED**: Separate transcription and translation steps
  - First pass: Transcribe in original language (`-l $LANG`)
  - Second pass: Create English translation (`--translate`)

### 2.4.0
- **FIXED**: French and non-English videos now keep original language text
  - Added language detection BEFORE transcription
  - Using `-l $LANG` flag to prevent auto-translation to English
- **Changed**: Default model is now `medium` for higher accuracy
- whisper.cpp without language flag was defaulting to English translation

### 2.3.0
- **New**: Language-aware formatting
  - French videos get French metadata headers (Chaîne, Durée, Publié le, etc.)
  - Spanish videos get Spanish metadata headers (Canal, Duración, Publicado, etc.)
  - English videos get English metadata headers
- **New**: Automatic English translation for non-English videos
- **New**: Language detection during transcription
- Transcript text always stays in original language
- Non-English transcripts include both original and English translation

### 2.2.0
- **New**: Automatic dependency installation on Apple Silicon Macs
  - Detects macOS and Apple Silicon automatically
  - Installs Homebrew if not present
  - Installs whisper.cpp automatically for optimal performance
  - Zero manual setup required for Mac users
- **Fixed**: Command execution error - no longer tries to spawn non-existent agent

### 2.1.0
- **CRITICAL FIX**: Apple Silicon optimization - prioritize whisper.cpp over Python Whisper
- **New**: Metal GPU acceleration guidance for Apple Silicon Macs (M1/M2/M3/M4)
- **New**: CoreML/Neural Engine setup instructions for fastest transcription
- **New**: SSL certificate error handling and workarounds
- **New**: Model selection recommendations based on hardware
- **Changed**: Default model is now `base` (not `medium`) to prevent resource issues
- **Changed**: Backend priority strongly favors whisper.cpp on Apple Silicon
- **Improved**: Warnings about Python Whisper performance on Apple Silicon
- **Improved**: Troubleshooting section with Apple Silicon specific issues
- **Improved**: Documentation with performance comparisons

### 2.0.0
- Major refactor for local file support
- Support for local video files (MP4, MKV, AVI, MOV, etc.)
- Support for local audio files (MP3, M4A, WAV, FLAC, etc.)
- Unified output location (`$HOME/venavi/transcripts/`)
- Improved working directory structure

### 1.1.0
- **New**: Automatic cleanup of temporary files after transcription
- **New**: Filename sanitization (spaces → hyphens, special characters removed)
- **New**: Working directory structure (`.youtube/downloads/` for temp files)
- **Changed**: Output directory now `./youtube/transcripts/` (visible folder)
- **Improved**: Better file management and organization
- **Improved**: Documentation with file management examples

### 1.0.0
- Initial release
- Support for yt-dlp audio download
- Multiple transcription backend support (whisper.cpp, OpenAI Whisper, Whisper API, macOS Speech)
- Timestamp generation
- Metadata extraction and formatting
- Comprehensive error handling

## Future Enhancements

Potential future features:
- Playlist support (transcribe multiple videos)
- Subtitle download (use existing subtitles if available)
- ~~Translation support (transcribe in one language, translate to another)~~ ✅ Implemented in v2.5.0
- Speaker diarization (identify different speakers)
- Automatic summarization of transcripts
- Integration with note-taking systems
