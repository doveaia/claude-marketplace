---
description: Transcribe YouTube videos and local video/audio files with timestamps and metadata
---

# /transcribe Command

## Purpose

Transcribes YouTube videos and local video/audio files to text with timestamps and metadata. Supports multiple transcription backends (whisper.cpp, OpenAI Whisper, macOS Speech Framework) with automatic fallback and clear error guidance.

**Supports**:
- YouTube videos (any URL format)
- Local video files (MP4, MKV, AVI, MOV, WebM, etc.)
- Local audio files (MP3, M4A, WAV, FLAC, OGG, etc.)

## Execution Instructions

**IMPORTANT**: Do NOT spawn a "transcribe" agent - it does not exist. Instead, execute this workflow directly using Bash commands.

### Step-by-Step Execution

1. **Parse the input** - Determine if it's a YouTube URL or local file path

2. **Check system and install dependencies** (MUST run first on macOS):
   ```bash
   # Step 2a: Check if running on Apple Silicon Mac
   IS_APPLE_SILICON=$(uname -m | grep -q 'arm64' && echo "yes" || echo "no")
   IS_MACOS=$(uname -s | grep -q 'Darwin' && echo "yes" || echo "no")

   echo "macOS: $IS_MACOS, Apple Silicon: $IS_APPLE_SILICON"
   ```

   ```bash
   # Step 2b: Check if Homebrew is installed
   if ! command -v brew &>/dev/null; then
     echo "Homebrew not found. Installing Homebrew..."
     /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

     # Add Homebrew to PATH for Apple Silicon
     if [ -f /opt/homebrew/bin/brew ]; then
       eval "$(/opt/homebrew/bin/brew shellenv)"
     fi
   else
     echo "✓ Homebrew is installed"
   fi
   ```

   ```bash
   # Step 2c: On Apple Silicon Mac, install whisper.cpp if not present
   if [ "$IS_MACOS" = "yes" ]; then
     if ! command -v whisper-cli &>/dev/null && ! brew list whisper-cpp &>/dev/null 2>&1; then
       echo "Installing whisper.cpp for optimal Apple Silicon performance..."
       brew install whisper-cpp
       echo "✓ whisper.cpp installed with Metal GPU support"
     else
       echo "✓ whisper.cpp is already installed"
     fi
   fi
   ```

   ```bash
   # Step 2d: Check if required models exist and download if missing
   # Models are stored in $(brew --prefix whisper-cpp)/share/whisper-cpp/models/ or ~/.cache/whisper/

   BREW_MODELS_DIR="$(brew --prefix whisper-cpp 2>/dev/null)/share/whisper-cpp/models"
   WHISPER_MODELS_DIR="$HOME/.cache/whisper"

   # Determine which models directory to use (prefer brew location)
   if [ -d "$BREW_MODELS_DIR" ]; then
     MODELS_DIR="$BREW_MODELS_DIR"
   else
     MODELS_DIR="$WHISPER_MODELS_DIR"
     mkdir -p "$MODELS_DIR"
   fi

   # Check for base model (used for language detection)
   BASE_MODEL_EXISTS="no"
   if [ -f "$MODELS_DIR/ggml-base.bin" ]; then
     BASE_MODEL_EXISTS="yes"
     echo "✓ Base model available at $MODELS_DIR/ggml-base.bin"
   fi

   # Check for medium model (used for transcription)
   MEDIUM_MODEL_EXISTS="no"
   if [ -f "$MODELS_DIR/ggml-medium.bin" ]; then
     MEDIUM_MODEL_EXISTS="yes"
     echo "✓ Medium model available at $MODELS_DIR/ggml-medium.bin"
   fi

   # Download missing models to user cache if not in brew location
   if [ "$BASE_MODEL_EXISTS" = "no" ]; then
     echo "Downloading base model for language detection..."
     mkdir -p "$WHISPER_MODELS_DIR"
     curl -L -o "$WHISPER_MODELS_DIR/ggml-base.bin" \
       "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin"
     MODELS_DIR="$WHISPER_MODELS_DIR"
     echo "✓ Base model downloaded"
   fi

   if [ "$MEDIUM_MODEL_EXISTS" = "no" ]; then
     echo "Downloading medium model for transcription (~1.5GB, this may take a few minutes)..."
     mkdir -p "$WHISPER_MODELS_DIR"
     curl -L -o "$WHISPER_MODELS_DIR/ggml-medium.bin" \
       "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-medium.bin"
     MODELS_DIR="$WHISPER_MODELS_DIR"
     echo "✓ Medium model downloaded"
   fi

   echo "Using models directory: $MODELS_DIR"
   ```

   **Model sizes reference**:
   - `base` (~148 MB) - Used for quick language detection
   - `medium` (~1.5 GB) - Used for high-accuracy transcription

   **Alternative: Download models manually**:
   ```bash
   # Create models directory
   mkdir -p ~/.cache/whisper

   # Download base model (for language detection)
   curl -L -o ~/.cache/whisper/ggml-base.bin \
     https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin

   # Download medium model (for transcription)
   curl -L -o ~/.cache/whisper/ggml-medium.bin \
     https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-medium.bin
   ```

3. **Validate the file exists** (for local files):
   ```bash
   ls -la "<file-path>"
   ```

4. **Create working directory**:
   ```bash
   mkdir -p "$HOME/venavi/.transcripts/downloads"
   ```

5. **Extract audio from video** (if video file):
   ```bash
   ffmpeg -i "<input-file>" -vn -acodec pcm_s16le -ar 16000 -ac 1 "$HOME/venavi/.transcripts/downloads/<filename>.wav"
   ```

6. **Check for transcription backends** (in priority order):
   ```bash
   # Check for whisper-cli (PREFERRED - from whisper.cpp)
   WHISPER_CLI=$(which whisper-cli 2>/dev/null || echo "")

   # Check for Homebrew whisper-cli
   if [ -z "$WHISPER_CLI" ]; then
     WHISPER_CLI=$(brew --prefix whisper-cpp 2>/dev/null)/bin/whisper-cli
     [ ! -f "$WHISPER_CLI" ] && WHISPER_CLI=""
   fi

   # Check for OpenAI Whisper Python (FALLBACK - slow on Apple Silicon!)
   WHISPER_PY=$(which whisper 2>/dev/null || echo "")
   ```

7. **Detect language FIRST** (CRITICAL - determines output language):
   ```bash
   # AUDIO_FILE should be the path to the audio file
   # MODELS_DIR should be set from step 2d
   # Run whisper-cli with --detect-language flag for quick language detection
   # The format is: "whisper_full_with_state: auto-detected language: fr (prob = 0.95)"
   WHISPER_OUTPUT=$(whisper-cli -m "$MODELS_DIR/ggml-base.bin" -l auto --detect-language -f "$AUDIO_FILE" 2>&1 | head -50)

   # Extract language code from whisper output
   # Look for pattern like "language: fr" or "detected language: fr"
   DETECTED_LANG=$(echo "$WHISPER_OUTPUT" | grep -oE 'language: [a-z]{2}' | head -1 | awk '{print $2}')

   # If detection failed, try alternate pattern
   if [ -z "$DETECTED_LANG" ]; then
     DETECTED_LANG=$(echo "$WHISPER_OUTPUT" | grep -oE 'auto-detected language: [a-z]{2}' | head -1 | awk '{print $NF}')
   fi

   # Default to 'auto' if still not detected
   if [ -z "$DETECTED_LANG" ]; then
     DETECTED_LANG="auto"
   fi

   echo "Detected language: $DETECTED_LANG"

   # Check if video is in English
   IS_ENGLISH="no"
   if [ "$DETECTED_LANG" = "en" ]; then
     IS_ENGLISH="yes"
   fi
   ```

8. **Transcribe audio in ORIGINAL LANGUAGE**:

   **Option A: whisper-cli** (RECOMMENDED - uses Metal GPU):
   ```bash
   # CRITICAL: Use -l with detected language to KEEP ORIGINAL LANGUAGE
   # Do NOT use --translate which would convert to English
   # Using medium model for higher accuracy
   # MODELS_DIR should be set from step 2d
   # For non-English: output file uses language suffix (e.g., <filename>-fr for French)
   # For English: output file has no suffix (e.g., <filename>)
   if [ "$IS_ENGLISH" = "no" ]; then
     OUTPUT_SUFFIX="-$DETECTED_LANG"
   else
     OUTPUT_SUFFIX=""
   fi
   whisper-cli -m "$MODELS_DIR/ggml-medium.bin" -l "$DETECTED_LANG" -f "$AUDIO_FILE" --output-txt -of "$HOME/venavi/.transcripts/downloads/<filename>${OUTPUT_SUFFIX}"
   ```

   **Key flags explained**:
   - `-m "$MODELS_DIR/ggml-medium.bin"` : Use medium model for higher accuracy (full path required)
   - `-l fr` (or detected language) : Transcribe in ORIGINAL language (French stays French)
   - `-f "$AUDIO_FILE"` : Input audio file
   - `--output-txt` : Output as text file
   - `-of <path>` : Output file path (without extension)
   - Do NOT use `--translate` flag - that converts to English

   **Output file naming**:
   - Non-English videos: `<filename>-<lang>.txt` (e.g., `video-fr.txt`)
   - English videos: `<filename>.txt` (no suffix)

   **Option B: OpenAI Whisper Python** (FALLBACK - CPU only, use base model!):
   ```bash
   # Use --task transcribe to keep original language (NOT --task translate)
   whisper "$AUDIO_FILE" --model base --language "$DETECTED_LANG" --output_format txt --output_dir "$HOME/venavi/.transcripts/downloads" --task transcribe
   ```

   **⚠️ WARNING**: If only OpenAI Whisper Python is available on Apple Silicon, warn the user:
   ```
   WARNING: Using OpenAI Whisper Python which is CPU-only on Apple Silicon.
   This may cause high CPU usage and fan noise.

   For better performance, install whisper.cpp (provides whisper-cli):
     brew install whisper-cpp

   Proceeding with base model (smallest) to minimize resource usage...
   ```

9. **Create English translation file for non-English videos**:

   **IMPORTANT**: If the video is NOT in English, create a SECOND file with English translation.

   ```bash
   if [ "$IS_ENGLISH" = "no" ]; then
     echo "Creating English translation..."

     # whisper-cli: Use --translate flag to translate to English
     # Output file uses -en suffix for English translation
     whisper-cli -m "$MODELS_DIR/ggml-medium.bin" -l "$DETECTED_LANG" --translate -f "$AUDIO_FILE" --output-txt -of "$HOME/venavi/.transcripts/downloads/<filename>-en"

     # For Python Whisper: Use --task translate
     # whisper "$AUDIO_FILE" --model base --language "$DETECTED_LANG" --output_format txt --output_dir "$HOME/venavi/.transcripts/downloads" --task translate
   fi
   ```

   **Output files for non-English videos** (BOTH files required):
   - `<filename>-<lang>.md` - Transcript in ORIGINAL language (e.g., `video-fr.md` for French)
   - `<filename>-en.md` - English translation

   **Output files for English videos**:
   - `<filename>.md` - English transcript only (no language suffix needed)

10. **Format the transcript(s)**:
   - **CRITICAL**: The markdown metadata headers should be in the VIDEO'S LANGUAGE
   - The transcript text remains in the original language
   - **For non-English videos**: Create TWO separate files with language code suffixes
   - **For English videos**: Create ONE file without language suffix

   **For non-English videos - TWO files required:**

   **FILE 1: Original Language Transcript** (`<filename>-<lang>.md`, e.g., `video-fr.md`):
   ```markdown
   # [Titre original de la vidéo]

   **Chaîne**: [Nom de la chaîne]
   **Durée**: [HH:MM:SS]
   **Publié le**: [Date]
   **URL**: [URL originale]
   **Langue**: Français (fr)
   **Transcrit le**: [Timestamp]
   **Backend de transcription**: [Outil utilisé]

   ---

   ## Transcription

   [00:00:00] Texte en français ici...
   [00:00:15] Suite du texte en français...
   ...

   ---

   ## Métadonnées

   - ID de la vidéo: [ID]
   ```

   **FILE 2: English Translation** (`<filename>-en.md`, e.g., `video-en.md`):
   ```markdown
   # [Video Title - English Translation]

   **Original Language**: French (fr)
   **Channel**: [Channel Name]
   **Duration**: [HH:MM:SS]
   **URL**: [Original URL]
   **Translated**: [Timestamp]
   **Translation Backend**: whisper.cpp (translate mode)

   ---

   ## English Translation

   [00:00:00] English translation of the French text...
   [00:00:15] Translation continues...
   ...

   ---

   ## Metadata

   - Video ID: [ID]
   - Original Language: French (fr)
   - This is an automatic translation
   ```

   **For English videos - single file only** (`<filename>.md`):
   ```markdown
   # [Title]

   **Channel**: [Channel Name]
   **Duration**: [HH:MM:SS]
   **Published**: [Date]
   **URL**: [Original URL]
   **Language**: English
   **Transcribed**: [Timestamp]
   **Transcription Backend**: [Tool Used]

   ---

   ## Transcript

   [00:00:00] English transcript here...
   [00:00:15] More English text...

   ---

   ## Metadata

   - Video ID: [ID]
   ```

11. **Save files to** `$HOME/venavi/transcripts/`:
    - **For non-English**: BOTH files required:
      - `<filename>-<lang>.md` (original language, e.g., `video-fr.md`)
      - `<filename>-en.md` (English translation)
    - **For English**: `<filename>.md` only (no suffix)

12. **Clean up** temporary files in downloads directory

### Apple Silicon Warning

**On M1/M2/M3/M4 Macs**: ALWAYS use whisper-cli (from whisper.cpp), NOT OpenAI Whisper Python.
- whisper-cli uses Metal GPU acceleration (fast, quiet)
- OpenAI Whisper Python is CPU-only (slow, loud fans, high CPU)

## High-level Flow

1. **Detect Input Type** - Determine if input is YouTube URL or local file path
2. **Validate Input** - Ensure URL format or file existence
3. **Check Dependencies & Install** - Verify and auto-install Homebrew + whisper.cpp on Apple Silicon
4. **Create Working Directory** - Create `$HOME/venavi/.transcripts/downloads/` for temporary files
5. **Prepare Audio** - Either download from YouTube OR extract/copy local media
6. **Detect Language FIRST** - Run whisper to identify audio language (CRITICAL step)
7. **Transcribe in ORIGINAL Language** - Use `-l $DETECTED_LANG` flag (NOT --translate) to keep text in original language
8. **Create English Translation** - For non-English videos ONLY: run whisper with `--translate` flag to get English version
9. **Format Output Files**:
   - **Non-English videos**: Create TWO files with language code suffixes
   - **English videos**: Create ONE file (no suffix)
10. **Sanitize Filename** - Remove spaces and special characters from filename
11. **Save Transcript(s)** - Write to `$HOME/venavi/transcripts/`:
    - **Non-English**: `<filename>-<lang>.md` (e.g., `-fr`) AND `<filename>-en.md`
    - **English**: `<filename>.md` only
12. **Clean Up** - Delete temporary audio files, metadata JSON, and intermediate transcripts

## Usage

### YouTube Videos

```bash
/transcribe <youtube-url> [options]
/transcribe https://youtube.com/watch?v=xxxxx
/transcribe https://youtu.be/xxxxx --backend whisper
```

### Local Files

```bash
/transcribe <file-path> [options]
/transcribe ./videos/meeting.mp4
/transcribe ~/Downloads/podcast.mp3
/transcribe /absolute/path/to/video.mkv --backend whisper.cpp
```

### Common Options

```bash
/transcribe <input> --output ./custom/location/
/transcribe <input> --language es --backend whisper
```

**Default Output**: Transcripts are saved to `$HOME/venavi/transcripts/` by default.

**Working Directory**: Temporary files (extracted audio, metadata) are stored in `$HOME/venavi/.transcripts/downloads/` and automatically cleaned up after transcription.

**Filename Sanitization**: Spaces are replaced with hyphens, special characters are removed. Examples:
- Video title: `"Full Tutorial: Build an AI Life Co-Pilot"` → `Full-Tutorial-Build-an-AI-Life-Co-Pilot-VIDEO_ID.md`
- Local file: `"My Meeting Recording.mp4"` → `My-Meeting-Recording.md`

## Arguments

- `<input>` - **Required**. Either:
  - YouTube video URL (youtube.com/watch?v=..., youtu.be/..., etc.)
  - Local file path (relative or absolute) to video/audio file

## Options

- `--backend <name>` - Specify transcription backend (whisper, whisper.cpp, macos-speech)
- `--output <path>` - Custom output directory for transcript
- `--no-timestamps` - Disable timestamp generation
- `--language <code>` - Specify audio language (e.g., en, es, fr)

## Output Format

### IMPORTANT: Two-File Approach for Non-English Videos

**Output structure**:
- **English videos**: ONE file (`<filename>.md`) - no language suffix
- **Non-English videos**: TWO files with language code suffixes:
  - `<filename>-<lang>.md` (e.g., `video-fr.md` for French)
  - `<filename>-en.md` (English translation)

### For English Videos (Single File)

**File: `<filename>.md`**
```markdown
# [Video Title]

**Channel**: [Channel Name]
**Duration**: [HH:MM:SS]
**Published**: [Date]
**URL**: [Original URL]
**Language**: English
**Transcribed**: [Timestamp]
**Transcription Backend**: [Tool Used]

---

## Transcript

[00:00:00] Introduction text here...
[00:00:15] More transcript content...
[00:01:30] Continue with timestamps...

---

## Metadata

- Video ID: [ID]
- Views: [Count]
- Likes: [Count]
- Upload Date: [Date]
```

### For Non-English Videos (Two Files)

#### FILE 1: Original Language (`<filename>-<lang>.md`, e.g., `video-fr.md`)

```markdown
# [Titre de la vidéo]

**Chaîne**: [Nom de la chaîne]
**Durée**: [HH:MM:SS]
**Publié le**: [Date]
**URL**: [URL originale]
**Langue**: Français (fr)
**Transcrit le**: [Timestamp]
**Backend de transcription**: [Outil utilisé]

---

## Transcription

[00:00:00] Texte en français ici...
[00:00:15] Suite de la transcription...
[00:01:30] Continuation avec les horodatages...

---

## Métadonnées

- ID de la vidéo: [ID]
- Vues: [Nombre]
- J'aime: [Nombre]
- Date de mise en ligne: [Date]
```

#### FILE 2: English Translation (`<filename>-en.md`, e.g., `video-en.md`)

```markdown
# [Video Title] - English Translation

**Original Language**: French (fr)
**Channel**: [Channel Name]
**Duration**: [HH:MM:SS]
**URL**: [Original URL]
**Translated**: [Timestamp]
**Translation Backend**: whisper.cpp (translate mode)

---

## English Translation

[00:00:00] English translation of the French text...
[00:00:15] Translation continues here...
[00:01:30] More translated content...

---

## Metadata

- Video ID: [ID]
- Original Language: French (fr)
- Translation: Automatic (whisper.cpp)
```

### For Local Files (Non-English)

#### FILE 1: Original Language (`<filename>-<lang>.md`, e.g., `recording-fr.md`)

```markdown
# [Nom du fichier]

**Source**: Fichier local
**Chemin du fichier**: [Chemin original]
**Durée**: [HH:MM:SS]
**Taille**: [Taille en MB]
**Langue**: Français (fr)
**Transcrit le**: [Timestamp]
**Backend de transcription**: [Outil utilisé]

---

## Transcription

[00:00:00] Texte en français...
[00:00:15] Suite du texte...

---

## Informations sur le fichier

- Format: [Format vidéo/audio]
- Codec: [Infos du codec]
- Débit binaire: [Bitrate]
```

#### FILE 2: English Translation (`<filename>-en.md`, e.g., `recording-en.md`)

```markdown
# [Filename] - English Translation

**Original Language**: French (fr)
**Source**: Local file
**Translated**: [Timestamp]

---

## English Translation

[00:00:00] English translation here...
[00:00:15] Translation continues...

---

## File Info

- Original file: [Path]
- Translation: Automatic (whisper.cpp)
```

## Error Handling

### File Not Found (Local Files)

```
Error: File not found: ./videos/meeting.mp4

Please check:
  - File path is correct
  - File exists at the specified location
  - You have read permissions for the file
```

### Unsupported File Format

```
Error: Unsupported file format: .xyz

Supported formats:
  Video: MP4, MKV, AVI, MOV, WebM, FLV, WMV
  Audio: MP3, M4A, WAV, FLAC, OGG, AAC, OPUS
```

### Missing yt-dlp (YouTube Only)

```
Error: yt-dlp not found (required for YouTube videos)

To install:
  macOS:    brew install yt-dlp
  Linux:    pip install yt-dlp
  Windows:  pip install yt-dlp

Note: yt-dlp is only needed for YouTube videos, not local files.
```

### Missing Transcription Backend

```
Error: No transcription backend available

Available options:
1. whisper.cpp (recommended for speed)
   brew install whisper-cpp

2. OpenAI Whisper (most accurate)
   pip install openai-whisper

3. macOS Speech Framework (built-in, macOS only)
   No installation required
```

### Download Failures

- Network errors: Retry with automatic exponential backoff
- Private/unavailable videos: Clear error message with video status
- Age-restricted content: Guidance on authentication if needed

### Transcription Failures

- Audio format issues: Automatic conversion to supported format
- Out of memory: Suggest using lighter model or chunking
- Language detection: Fallback to English or prompt for language code

## Default Output Location

Transcripts are saved to `$HOME/venavi/transcripts/` by default.

The directory is created automatically if it doesn't exist. All transcripts (YouTube and local files) are saved in one organized location.

**Custom Location**: Use `--output <path>` to override the default location.

## File Management & Cleanup

### Working Directory Structure

```
.youtube/
├── downloads/           # Temporary files (auto-cleaned)
│   ├── *.m4a           # Audio file (deleted after transcription)
│   ├── *.info.json     # Video metadata (deleted after extraction)
│   └── transcript*.txt # Intermediate transcripts (deleted after formatting)
└── transcripts/         # Final transcripts (permanent)
    └── Video-Title-VIDEO_ID.md
```

### Automatic Cleanup

After successful transcription, the following temporary files are **automatically deleted**:
- Audio files (`.m4a`, `.wav`, etc.)
- Metadata JSON files (`.info.json`)
- Intermediate transcript files (`transcript.txt`, `transcript_timestamped.txt`)

**Only the final formatted transcript** in `$HOME/venavi/youtube/transcripts/` is kept.

### Manual Cleanup

If transcription fails or is interrupted, temporary files may remain in `.youtube/downloads/`. To clean up:

```bash
rm -rf .youtube/downloads/*
```

### Filename Sanitization

Filenames are automatically sanitized to avoid filesystem issues:
- **Spaces** → Hyphens (`-`)
- **Special characters** (`:`, `｜`, `/`, `\`, etc.) → Removed or replaced
- **Multiple consecutive spaces/hyphens** → Single hyphen
- **Length limit** → Truncated to 200 characters (plus video ID)

**Examples**:
- `"Full Tutorial: Build AI ｜ Claude"` → `Full-Tutorial-Build-AI-Claude-VIDEO_ID.md`
- `"React/Next.js Tutorial - Part 1"` → `React-Next-js-Tutorial-Part-1-VIDEO_ID.md`
- `"Episode #42: Advanced TypeScript"` → `Episode-42-Advanced-TypeScript-VIDEO_ID.md`

## Agent Orchestration

1. **video-downloader** agent:
   - Validates YouTube URL
   - Checks yt-dlp availability
   - Downloads audio in optimal format
   - Extracts video metadata
   - Returns audio file path and metadata

2. **audio-transcriber** agent:
   - Detects available transcription backends
   - Selects optimal backend based on availability and user preference
   - Runs transcription with timestamps
   - Handles errors with fallback strategies
   - Returns formatted transcript

## Skills Applied

- **transcription-backends**: Progressive disclosure of transcription tool selection, installation, and usage
- **youtube-metadata**: Knowledge about extracting, formatting, and preserving YouTube video metadata

## Examples

### Example 1: YouTube Video

```bash
/transcribe https://youtube.com/watch?v=dQw4w9WgXcQ
```

Downloads and transcribes video, saves to `$HOME/venavi/transcripts/video-title-VIDEO_ID.md`

### Example 2: Local Video File

```bash
/transcribe ./videos/meeting-recording.mp4
```

Extracts audio and transcribes, saves to `$HOME/venavi/transcripts/meeting-recording.md`

### Example 3: Local Audio File

```bash
/transcribe ~/Downloads/podcast-episode.mp3 --backend whisper.cpp
```

Transcribes audio with specified backend

### Example 4: Custom Output Location

```bash
/transcribe ./interview.mkv --output ./newsletter/research/
```

Saves transcript to custom directory

### Example 5: Non-English Content

```bash
/transcribe https://youtube.com/watch?v=xxxxx --language es
```

Transcribes Spanish video with language-specific optimization

## Success Criteria

- Input validated (YouTube URL or local file path)
- Audio prepared successfully (downloaded from YouTube OR extracted from local file)
- Transcription completed with timestamps
- Metadata extracted and formatted
- Transcript saved to appropriate location
- Temporary files cleaned up automatically
- Clear error messages if anything fails
- Guidance provided for missing dependencies
