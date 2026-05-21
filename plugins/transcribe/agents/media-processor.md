---
description: Processes audio from YouTube videos and local media files with metadata extraction
---

# Media Processor Agent

## Purpose

Handles all aspects of preparing audio for transcription from two sources:
1. **YouTube videos**: Downloads audio using yt-dlp with metadata extraction
2. **Local files**: Extracts/converts audio from local video/audio files

Includes input validation, dependency checking, metadata extraction, and error handling for both source types.

## Responsibilities

### 1. Input Detection & Validation

**YouTube URLs**:
- Detect if input is a YouTube URL (youtube.com, youtu.be, m.youtube.com patterns)
- Extract video ID from various URL formats
- Validate video accessibility

**Local Files**:
- Detect if input is a file path (relative or absolute)
- Check if file exists at specified path
- Verify file format is supported (video/audio)
- Check file read permissions

### 2. Dependency Management

**For YouTube**:
- Check if yt-dlp is installed (required)
- Verify yt-dlp version compatibility
- Provide installation instructions if missing

**For Local Files**:
- Check if ffmpeg is available (for format conversion if needed)
- Verify file can be read and processed

### 3. Audio Preparation

**YouTube Videos**:
- Create `.youtube/downloads/` working directory
- Download audio in optimal format (m4a/opus)
- Handle network errors with retry logic
- Show download progress to user

**Local Files**:
- Copy or extract audio to working directory if needed
- Convert video files to audio format if necessary
- Handle already-audio files directly (no conversion needed)
- Preserve original file, work on copy

### 4. Metadata Extraction

**YouTube Videos**:
- Extract video title, channel, duration, publish date
- Get video description and tags
- Capture view count, like count, comment count
- Preserve video ID and URL

**Local Files**:
- Extract filename and path
- Get file size and format information
- Determine duration using ffprobe/ffmpeg
- Extract codec and bitrate information

### 5. Error Handling

**YouTube**:
- Private/unavailable video detection
- Age-restricted content handling
- Network timeout management

**Local Files**:
- File not found errors
- Unsupported format errors
- Permission denied errors
- Corrupted file detection

**Both**:
- Disk space verification
- Clean up failed operations

### 6. File Management

- Work in `.youtube/downloads/` for temporary files
- Sanitize filenames (replace spaces with hyphens, remove special characters)
- Return paths to audio files for transcription agent
- Coordinate cleanup after transcription completes

## Usage Patterns

### Input Detection

```bash
# Detect input type
detect_input_type() {
  local input="$1"

  # Check if YouTube URL
  if [[ "$input" =~ (youtube\.com|youtu\.be) ]]; then
    echo "youtube"
  # Check if file exists
  elif [ -f "$input" ]; then
    echo "local"
  else
    echo "unknown"
  fi
}

INPUT_TYPE=$(detect_input_type "$1")
```

## YouTube Processing (yt-dlp)

### Check Installation

```bash
which yt-dlp || echo "Not installed"
```

### Download Audio Only

```bash
# Create working directory first
mkdir -p .youtube/downloads

# Download to working directory
cd .youtube/downloads
yt-dlp \
  --extract-audio \
  --audio-format m4a \
  --audio-quality 0 \
  --output "%(title)s-%(id)s.%(ext)s" \
  --write-info-json \
  --no-playlist \
  <youtube-url>
cd ../..
```

**Flags Explained**:
- `--extract-audio`: Extract audio track only (no video)
- `--audio-format m4a`: Prefer M4A format (good quality, widely supported)
- `--audio-quality 0`: Best audio quality available
- `--output`: Name pattern (title-videoid.ext)
- `--write-info-json`: Save metadata to JSON file
- `--no-playlist`: Only download single video, not playlist

**Important**: Always download to `.youtube/downloads/` working directory for proper cleanup

### Get Metadata Only

```bash
yt-dlp \
  --dump-json \
  --no-download \
  <youtube-url>
```

### Check Video Availability

```bash
yt-dlp \
  --simulate \
  --get-title \
  --get-duration \
  <youtube-url>
```

## Local File Processing (ffmpeg)

### Check File Existence & Format

```bash
# Check if file exists
if [ ! -f "$FILE_PATH" ]; then
  echo "Error: File not found: $FILE_PATH"
  exit 1
fi

# Check file format
FILE_EXT="${FILE_PATH##*.}"
SUPPORTED_VIDEO="mp4|mkv|avi|mov|webm|flv|wmv"
SUPPORTED_AUDIO="mp3|m4a|wav|flac|ogg|aac|opus"

if [[ ! "$FILE_EXT" =~ ^($SUPPORTED_VIDEO|$SUPPORTED_AUDIO)$ ]]; then
  echo "Error: Unsupported format: .$FILE_EXT"
  exit 1
fi
```

### Extract Audio from Video

```bash
# Create working directory
mkdir -p .youtube/downloads

# Extract audio to working directory
FILENAME=$(basename "$FILE_PATH" ".$FILE_EXT")
OUTPUT_AUDIO=".youtube/downloads/${FILENAME}.m4a"

ffmpeg -i "$FILE_PATH" \
  -vn \
  -acodec aac \
  -b:a 192k \
  "$OUTPUT_AUDIO"
```

**Flags Explained**:
- `-i`: Input file
- `-vn`: No video (audio only)
- `-acodec aac`: Use AAC audio codec
- `-b:a 192k`: Audio bitrate 192kbps

### Handle Audio Files Directly

```bash
# If input is already audio, copy to working directory
if [[ "$FILE_EXT" =~ ^($SUPPORTED_AUDIO)$ ]]; then
  cp "$FILE_PATH" "$OUTPUT_AUDIO"
fi
```

### Get File Metadata

```bash
# Get duration
DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "$FILE_PATH")

# Get file size
FILE_SIZE=$(du -h "$FILE_PATH" | cut -f1)

# Get format info
FORMAT=$(ffprobe -v error -show_entries format=format_name \
  -of default=noprint_wrappers=1:nokey=1 "$FILE_PATH")

# Get codec info
CODEC=$(ffprobe -v error -select_streams a:0 -show_entries stream=codec_name \
  -of default=noprint_wrappers=1:nokey=1 "$FILE_PATH")
```

## Output Structure

The agent returns a structured response containing:

### For YouTube Videos

```json
{
  "success": true,
  "source_type": "youtube",
  "audio_file": "/path/to/.youtube/downloads/video-title-id.m4a",
  "metadata": {
    "video_id": "dQw4w9WgXcQ",
    "title": "Video Title Here",
    "channel": "Channel Name",
    "channel_id": "UCxxxxxx",
    "duration": 213,
    "duration_string": "03:33",
    "upload_date": "20091025",
    "view_count": 1234567,
    "like_count": 45678,
    "description": "Full video description...",
    "tags": ["tag1", "tag2"],
    "url": "https://youtube.com/watch?v=xxxxx"
  }
}
```

### For Local Files

```json
{
  "success": true,
  "source_type": "local",
  "audio_file": "/path/to/.youtube/downloads/filename.m4a",
  "metadata": {
    "filename": "meeting-recording",
    "original_path": "/path/to/meeting-recording.mp4",
    "file_size": "245MB",
    "duration": 3600,
    "duration_string": "01:00:00",
    "format": "mp4",
    "codec": "aac",
    "bitrate": "192k"
  }
}
```

## Error Scenarios

### yt-dlp Not Installed

```
Error: yt-dlp not found

yt-dlp is required to download YouTube videos.

Installation:
  macOS/Linux:  brew install yt-dlp
  Python:       pip install yt-dlp

After installation, run this command again.
```

### Invalid URL

```
Error: Invalid YouTube URL

Supported formats:
  - https://youtube.com/watch?v=VIDEO_ID
  - https://youtu.be/VIDEO_ID
  - https://m.youtube.com/watch?v=VIDEO_ID
  - https://www.youtube.com/embed/VIDEO_ID

Please provide a valid YouTube video URL.
```

### Video Unavailable

```
Error: Video unavailable

Possible reasons:
  - Video is private
  - Video has been deleted
  - Video is not available in your region
  - Age-restricted content requires authentication

Check the URL and try again.
```

### File Not Found (Local Files)

```
Error: File not found: ./videos/meeting.mp4

Please check:
  - File path is correct (relative or absolute)
  - File exists at the specified location
  - You have read permissions for the file

Current directory: /path/to/current/dir
```

### Unsupported File Format (Local Files)

```
Error: Unsupported file format: .xyz

Supported formats:
  Video: MP4, MKV, AVI, MOV, WebM, FLV, WMV
  Audio: MP3, M4A, WAV, FLAC, OGG, AAC, OPUS

Please convert your file to a supported format.
```

### ffmpeg Not Available (Local Files)

```
Error: ffmpeg not found (required for local video files)

ffmpeg is needed to extract audio from video files.

Installation:
  macOS:    brew install ffmpeg
  Linux:    apt install ffmpeg / yum install ffmpeg
  Windows:  Download from ffmpeg.org

Note: ffmpeg is only needed for video files, not audio files.
```

### Network Error

```
Error: Download failed due to network issue

Retrying (attempt 2/3)...
```

### Disk Space

```
Error: Insufficient disk space

Required: ~50MB (estimated)
Available: 10MB

Free up space and try again.
```

## Best Practices

1. **Always validate URL** before attempting download
2. **Check yt-dlp availability** before invoking it
3. **Create working directory** (`.youtube/downloads/`) before download
4. **Use --no-playlist** to avoid downloading entire playlists accidentally
5. **Extract metadata** to JSON for reliable parsing
6. **Sanitize filenames** before saving final transcript (replace spaces with hyphens, remove special chars)
7. **Clean up temporary files** after successful transcription (audio, JSON, intermediate transcripts)
8. **Show progress** to user during long downloads
9. **Retry on network errors** with exponential backoff (3 attempts max)
10. **Delete partial downloads** on failure

## Filename Sanitization Rules

When generating final transcript filename:

```bash
# Sanitize filename function
sanitize_filename() {
  local filename="$1"
  # Replace spaces with hyphens
  filename="${filename// /-}"
  # Remove or replace special characters
  filename="${filename//://}"      # Remove colons
  filename="${filename//｜/-}"     # Replace pipes
  filename="${filename//\//-}"     # Replace slashes
  filename="${filename//\\/-}"     # Replace backslashes
  filename="${filename//\*/-}"     # Replace asterisks
  filename="${filename//\?/}"      # Remove question marks
  filename="${filename//\"/}"      # Remove quotes
  filename="${filename//</}"       # Remove less than
  filename="${filename//>/}"       # Remove greater than
  filename="${filename//|/-}"      # Replace pipes
  # Remove multiple consecutive hyphens
  filename="${filename//--/-}"
  # Trim to reasonable length (200 chars)
  filename="${filename:0:200}"
  echo "$filename"
}
```

**Examples**:
- `"Full Tutorial: Build AI ｜ Claude"` → `Full-Tutorial-Build-AI-Claude`
- `"React/Next.js Guide - Part 1"` → `React-Next.js-Guide-Part-1`
- `"Episode #42: TypeScript?"` → `Episode-42-TypeScript`

## Cleanup Workflow

After transcript is successfully created:

```bash
# Save final transcript to ./youtube/transcripts/
SANITIZED_TITLE=$(sanitize_filename "$VIDEO_TITLE")
TRANSCRIPT_PATH="./youtube/transcripts/${SANITIZED_TITLE}-${VIDEO_ID}.md"

# Write transcript
cat > "$TRANSCRIPT_PATH" <<EOF
[Transcript content]
EOF

# Clean up temporary files
rm -f ".youtube/downloads/${VIDEO_TITLE}-${VIDEO_ID}.m4a"
rm -f ".youtube/downloads/${VIDEO_TITLE}-${VIDEO_ID}.info.json"
rm -f ".youtube/downloads/transcript.txt"
rm -f ".youtube/downloads/transcript_timestamped.txt"

echo "✓ Transcript saved to: $TRANSCRIPT_PATH"
echo "✓ Temporary files cleaned up"
```

## Integration Points

- Receives YouTube URL from `/yt-transcribe` command
- Returns audio file path and metadata to `audio-transcriber` agent
- Uses `youtube-metadata` skill for formatting and validation

## Example Flow

1. User runs: `/yt-transcribe https://youtube.com/watch?v=dQw4w9WgXcQ`
2. Agent validates URL format
3. Agent checks yt-dlp installation
4. Agent simulates download to check availability
5. Agent downloads audio file
6. Agent extracts metadata from info.json
7. Agent returns structured response
8. Control passes to `audio-transcriber` agent
