# Changelog

All notable changes to the transcribe plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [3.0.0] - 2026-05-15

### Changed
- **BREAKING**: Migrated all slash commands from legacy `commands/*.md` layout to `skills/<name>/SKILL.md` format
- Slash command invocation names are unchanged for end users (e.g. `/transcribe:<command>` still works)

### Removed
- Legacy `commands/` directory (replaced by per-command skill folders under `skills/`)

## [2.6.0] - 2025-01-15

### Changed
- **File naming convention for non-English videos**: Now uses language code suffixes
  - Original language: `<filename>-<lang>.md` (e.g., `video-fr.md` for French)
  - English translation: `<filename>-en.md` (e.g., `video-en.md`)
  - English videos unchanged: `<filename>.md` (no suffix)
- Both files are now REQUIRED for non-English content (original + translation)
- Clearer file naming makes it obvious which file contains which language

### Fixed
- Issue where only one output file was created for non-English videos
- Temporary files now use correct language suffixes before formatting
- Documentation now consistently shows both output files for non-English content

## [2.5.1] - 2025-01-15

### Fixed
- **CRITICAL**: Updated whisper.cpp binary name from `whisper-cpp` to `whisper-cli`
  - The Homebrew `whisper-cpp` package now provides `whisper-cli` as the binary
  - All command examples and detection logic updated
- **Model path handling**: Now uses full model path (`-m "$MODELS_DIR/ggml-medium.bin"`) instead of model name
  - Models are located at `$(brew --prefix whisper-cpp)/share/whisper-cpp/models/`
  - Falls back to `~/.cache/whisper/` if brew models not found
- **Audio file flag**: Added `-f` flag for input audio file (required by whisper-cli)
- Improved model directory detection with clearer output messages

### Changed
- Model directory preference: brew location first, then user cache
- Better documentation of whisper-cli flags and their purposes

## [2.5.0] - 2025-01-14

### Added
- **Two-file output for non-English videos**:
  - Primary file: Transcript in ORIGINAL language (e.g., French stays French)
  - Secondary file: English translation (`<filename>-english.md`)
- English videos still produce a single file (no translation needed)

### Fixed
- **CRITICAL**: Improved language detection parsing
  - Now correctly extracts language code from whisper-cpp output
  - Handles multiple output formats: `language: fr` and `auto-detected language: fr`
  - Fixed issue where French videos were transcribed in English

### Changed
- **Separate translation step**: Non-English videos now run whisper twice:
  1. First pass: `-l $DETECTED_LANG` to transcribe in original language
  2. Second pass: `--translate` flag to create English translation
- Clear distinction between `transcribe` task (keeps original language) and `translate` task (converts to English)

### Technical Details
- whisper.cpp `--translate` flag is now used ONLY for creating the English translation file
- Original language transcription uses `-l` flag without `--translate`
- Language detection runs before main transcription to determine correct workflow

## [2.4.0] - 2025-01-14

### Fixed
- **CRITICAL**: French (and other non-English) videos now keep original language text
  - Added language detection step BEFORE transcription
  - Using `-l $LANG` flag to prevent auto-translation to English
  - Transcript text now stays in the video's original language
- **Changed**: Default model is now `medium` for higher accuracy (user preference)

### Technical Details
- whisper.cpp without `-l` flag defaults to English translation
- Now runs quick language detection first with tiny model
- Then uses detected language code in full transcription with medium model
- Example: `-l fr` for French, `-l es` for Spanish, etc.

## [2.3.0] - 2025-01-14

### Added
- **Language-aware formatting**: Transcript metadata now uses the VIDEO'S LANGUAGE
  - French videos get French headers (Chaîne, Durée, Publié le, etc.)
  - Spanish videos get Spanish headers (Canal, Duración, Publicado, etc.)
  - English videos get English headers (Channel, Duration, Published, etc.)
- **Automatic English translation**: For non-English videos, an English translation section is added
- **Language detection**: Automatically detects the video language during transcription
- **Bilingual output examples**: Added French and Spanish output format examples

### Changed
- Transcript text always stays in the original language (not translated)
- Metadata section headers adapt to detected language
- Non-English transcripts include both original and translated versions

## [2.2.0] - 2025-01-14

### Added
- **Automatic dependency installation on Apple Silicon**:
  - Detects if running on Apple Silicon Mac (M1/M2/M3/M4)
  - Automatically installs Homebrew if not present
  - Automatically installs whisper.cpp via Homebrew for optimal performance
  - Eliminates manual setup steps for Mac users
- System detection commands to identify macOS and Apple Silicon

### Changed
- Command file now includes explicit dependency installation steps
- whisper.cpp is automatically installed on macOS for best performance

### Fixed
- **Command execution error**: Fixed issue where command tried to spawn non-existent "transcribe" agent
- Added explicit step-by-step execution instructions to command file
- Command now executes workflow directly using Bash instead of spawning agents

## [2.1.1] - 2025-01-14

### Fixed
- **Command execution error**: Fixed issue where command tried to spawn non-existent "transcribe" agent
- Added explicit step-by-step execution instructions to command file
- Command now executes workflow directly using Bash instead of spawning agents

## [2.1.0] - 2025-01-14

### CRITICAL FIX
- **Apple Silicon optimization**: Prioritize whisper.cpp over Python Whisper on M1/M2/M3/M4 Macs
  - Python Whisper does NOT use GPU/Metal acceleration on Apple Silicon
  - Causes 100% CPU usage, loud fan noise, and potential thermal throttling
  - whisper.cpp is 3-10x faster due to Metal GPU acceleration

### Added
- Metal GPU acceleration guidance for Apple Silicon Macs
- CoreML/Neural Engine setup instructions for fastest transcription (8-12x speedup)
- SSL certificate error handling and workarounds for Python Whisper
- Model selection recommendations based on hardware capabilities
- Performance comparison table showing Apple Silicon differences
- Troubleshooting section for Apple Silicon specific issues (fan noise, thermal throttling)

### Changed
- **Default model**: Now uses `base` instead of `medium` to prevent resource issues
- **Backend priority**: Strongly favors whisper.cpp on Apple Silicon
- Backend detection now checks multiple binary names (`whisper-cpp`, `whisper`, homebrew location)

### Improved
- Comprehensive warnings about Python Whisper performance on Apple Silicon
- Installation instructions with Apple Silicon specific guidance
- Documentation with performance comparisons and hardware recommendations
- Error handling for SSL certificate issues common with Python Whisper

### Fixed
- Addressed issue where using Python Whisper with medium model caused MacBook fans to spin loudly
- Added guidance for SSL errors when downloading Whisper models

## [2.0.0] - 2024-12-26

### BREAKING CHANGES
- **Plugin renamed** from `yt-transcribe` to `transcribe`
- **Command renamed** from `/yt-transcribe` to `/transcribe`
- Users must uninstall old plugin and reinstall new one

### Added
- **Local file support**: Transcribe video and audio files from your filesystem
  - Support for local video files (MP4, MKV, AVI, MOV, WebM, FLV, WMV)
  - Support for local audio files (MP3, M4A, WAV, FLAC, OGG, AAC, OPUS)
  - Automatic audio extraction from video files using ffmpeg
  - Direct transcription of audio files (no conversion needed)
- **Input detection**: Automatically detects if input is YouTube URL or local file path
- **Enhanced metadata**: Extract file size, format, codec, and bitrate for local files
- **ffmpeg integration**: Audio extraction from local video files
- Comprehensive error handling for local file scenarios (file not found, unsupported format, etc.)

### Changed
- Output directory changed to `./transcripts/` (unified location for all sources)
- Agent renamed from `video-downloader` to `media-processor` to reflect dual capabilities
- Documentation updated throughout to reflect YouTube + local file support
- Installation instructions clarify when yt-dlp/ffmpeg are needed

### Improved
- More flexible input handling (YouTube URLs OR local file paths)
- Better dependency management (yt-dlp only for YouTube, ffmpeg only for local video)
- Enhanced examples showing both YouTube and local file usage

## [1.1.0] - 2024-12-26

### Added
- Automatic cleanup of temporary files after successful transcription
- Filename sanitization to replace spaces with hyphens and remove special characters
- Working directory structure (`.youtube/downloads/` for temporary files)
- File management documentation with cleanup examples
- Detailed sanitization rules and examples

### Changed
- Output directory changed from `.youtube/transcripts/` to `./youtube/transcripts/` (visible folder)
- Final transcript filenames now sanitized to avoid filesystem issues
- Workflow now includes explicit cleanup step

### Improved
- Documentation with comprehensive file management section
- Agent instructions include filename sanitization and cleanup procedures
- Better organization of temporary vs permanent files

## [1.0.0] - 2024-12-25

### Added
- Initial release
- YouTube audio download via yt-dlp
- Multiple transcription backend support:
  - whisper.cpp (recommended)
  - OpenAI Whisper (Python)
  - Whisper API
  - macOS Speech Framework
- Automatic backend detection and selection
- Timestamp generation in `[HH:MM:SS]` format
- Metadata extraction (title, channel, duration, views, etc.)
- Comprehensive error handling with installation guidance
- Support for multiple languages
- Model selection (tiny, base, small, medium, large)
- Custom output location option

### Documentation
- Comprehensive README with installation instructions
- Usage examples and troubleshooting guide
- Agent and skill documentation
- Architecture overview

[2.6.0]: https://github.com/your-repo/claude-code-plugins/compare/v2.5.1...v2.6.0
[2.5.1]: https://github.com/your-repo/claude-code-plugins/compare/v2.5.0...v2.5.1
[2.5.0]: https://github.com/your-repo/claude-code-plugins/compare/v2.4.0...v2.5.0
[2.4.0]: https://github.com/your-repo/claude-code-plugins/compare/v2.3.0...v2.4.0
[2.3.0]: https://github.com/your-repo/claude-code-plugins/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/your-repo/claude-code-plugins/compare/v2.1.1...v2.2.0
[2.1.1]: https://github.com/your-repo/claude-code-plugins/compare/v2.1.0...v2.1.1
[2.1.0]: https://github.com/your-repo/claude-code-plugins/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/your-repo/claude-code-plugins/compare/v1.1.0...v2.0.0
[1.1.0]: https://github.com/your-repo/claude-code-plugins/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/your-repo/claude-code-plugins/releases/tag/v1.0.0
