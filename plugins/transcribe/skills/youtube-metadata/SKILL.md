---
description: Knowledge about extracting, formatting, and preserving YouTube video metadata
---

# YouTube Metadata Skill

## Purpose

Provides comprehensive knowledge about YouTube video metadata extraction, validation, formatting, and preservation for transcripts and content research workflows.

## Metadata

**Skill Type**: Data Processing Knowledge
**Applies To**: YouTube video information and metadata
**Context**: Content research, transcription, archival

## Layer 1: Essential Metadata Fields

### Core Fields (Always Extract)

1. **video_id** - Unique YouTube identifier (e.g., `dQw4w9WgXcQ`)
2. **title** - Video title
3. **channel** - Channel name
4. **channel_id** - Channel unique identifier
5. **url** - Full YouTube URL
6. **upload_date** - When video was published
7. **duration** - Video length in seconds
8. **duration_string** - Human-readable duration (HH:MM:SS)

### Extended Fields (Extract When Available)

9. **description** - Full video description
10. **view_count** - Number of views
11. **like_count** - Number of likes
12. **comment_count** - Number of comments
13. **tags** - Video tags/keywords
14. **categories** - YouTube category
15. **thumbnail** - Thumbnail URL

## Layer 2: URL Parsing & Validation

### Supported URL Formats

```
Standard:
  https://www.youtube.com/watch?v=VIDEO_ID
  https://youtube.com/watch?v=VIDEO_ID

Short:
  https://youtu.be/VIDEO_ID

Mobile:
  https://m.youtube.com/watch?v=VIDEO_ID

Embed:
  https://www.youtube.com/embed/VIDEO_ID

With playlist:
  https://youtube.com/watch?v=VIDEO_ID&list=PLAYLIST_ID
  (extract only VIDEO_ID, ignore playlist)

With timestamp:
  https://youtube.com/watch?v=VIDEO_ID&t=120s
  (preserve timestamp info)
```

### URL Validation Regex

```regex
^(?:https?:\/\/)?(?:www\.)?(?:youtube\.com\/(?:watch\?v=|embed\/)|youtu\.be\/)([a-zA-Z0-9_-]{11})
```

**Video ID Format**: Exactly 11 characters, alphanumeric plus `_` and `-`

### Extraction Function

```bash
# Extract video ID from any YouTube URL
extract_video_id() {
  echo "$1" | sed -E 's/.*[?&]v=([^&]*).*/\1/' | sed 's/.*youtu.be\///' | cut -c1-11
}
```

## Layer 3: Metadata Extraction with yt-dlp

### Get All Metadata (JSON)

```bash
yt-dlp --dump-json --no-download <youtube-url>
```

**Output**: Complete JSON object with all available metadata

### Get Specific Fields

```bash
# Title only
yt-dlp --get-title <url>

# Duration (seconds)
yt-dlp --get-duration <url>

# Upload date
yt-dlp --get-filename -o "%(upload_date)s" <url>

# Channel name
yt-dlp --get-filename -o "%(channel)s" <url>
```

### Batch Extraction

```bash
yt-dlp \
  --dump-json \
  --no-download \
  --playlist-items 1 \
  --write-info-json \
  -o "%(id)s" \
  <url>
```

Creates: `VIDEO_ID.info.json` with all metadata

## Layer 4: Metadata Formatting

### Markdown Header Format

```markdown
# [Video Title]

**Channel**: [Channel Name]
**Duration**: [HH:MM:SS]
**Published**: [YYYY-MM-DD]
**Views**: [Formatted view count]
**URL**: [Full YouTube URL]
**Transcribed**: [Timestamp of transcription]
**Transcription Backend**: [Tool used]

---
```

### Example

```markdown
# Building a CLI with Go in 2024

**Channel**: Venavi Tech
**Duration**: 15:42
**Published**: 2024-03-15
**Views**: 12,543
**URL**: https://youtube.com/watch?v=dQw4w9WgXcQ
**Transcribed**: 2024-03-20 10:30:00 UTC
**Transcription Backend**: whisper.cpp (base model)

---
```

### Duration Formatting

```bash
# Convert seconds to HH:MM:SS
format_duration() {
  local seconds=$1
  printf "%02d:%02d:%02d" $((seconds/3600)) $((seconds%3600/60)) $((seconds%60))
}

# Examples:
# 90 → 00:01:30
# 3665 → 01:01:05
# 12600 → 03:30:00
```

### Date Formatting

```bash
# yt-dlp format: 20240315
# Convert to: 2024-03-15

format_date() {
  local yt_date=$1  # e.g., 20240315
  echo "$yt_date" | sed 's/\([0-9]\{4\}\)\([0-9]\{2\}\)\([0-9]\{2\}\)/\1-\2-\3/'
}
```

### View Count Formatting

```bash
# Format large numbers with commas
format_number() {
  printf "%'d" $1
}

# Examples:
# 1234567 → 1,234,567
# 12543 → 12,543
```

## Layer 5: Metadata Preservation & Archival

### Full Metadata Block (Extended Format)

```markdown
---
video_id: dQw4w9WgXcQ
title: "Video Title Here"
channel: "Channel Name"
channel_id: UCxxxxxxxxx
url: https://youtube.com/watch?v=dQw4w9WgXcQ
upload_date: 2024-03-15
duration: 942
duration_string: "15:42"
view_count: 12543
like_count: 1024
comment_count: 87
categories: ["Education", "Technology"]
tags: ["go", "cli", "tutorial"]
transcribed_date: 2024-03-20T10:30:00Z
transcription_backend: whisper.cpp
transcription_model: base
---

# Video Title Here

[Transcript content follows...]
```

### JSON Metadata (For Programmatic Use)

```json
{
  "video_id": "dQw4w9WgXcQ",
  "title": "Video Title Here",
  "channel": "Channel Name",
  "channel_id": "UCxxxxxxxxx",
  "url": "https://youtube.com/watch?v=dQw4w9WgXcQ",
  "upload_date": "2024-03-15",
  "duration": 942,
  "duration_string": "15:42",
  "view_count": 12543,
  "like_count": 1024,
  "comment_count": 87,
  "categories": ["Education", "Technology"],
  "tags": ["go", "cli", "tutorial"],
  "description": "Full video description...",
  "thumbnail": "https://i.ytimg.com/vi/dQw4w9WgXcQ/maxresdefault.jpg",
  "transcription": {
    "date": "2024-03-20T10:30:00Z",
    "backend": "whisper.cpp",
    "model": "base",
    "language": "en",
    "confidence": 0.95
  }
}
```

## Layer 6: Special Cases & Error Handling

### Unavailable Metadata Fields

**Missing like_count** (disabled by creator):
```markdown
**Likes**: [Disabled]
```

**Missing description**:
```markdown
**Description**: [None provided]
```

**Private view count**:
```markdown
**Views**: [Private]
```

### Age-Restricted Content

- yt-dlp may fail without authentication
- Provide guidance: "Age-restricted. Sign in with cookies or use `--cookies-from-browser chrome`"

### Geographically Restricted

- Extract available metadata even if video can't be downloaded
- Note restriction in metadata: `**Availability**: [Restricted in some regions]`

### Deleted/Private Videos

```markdown
# [Video Unavailable]

**Video ID**: dQw4w9WgXcQ
**URL**: https://youtube.com/watch?v=dQw4w9WgXcQ
**Status**: Deleted or Private
**Checked**: 2024-03-20

Unable to retrieve video metadata or transcript.
```

### Live Streams

- Duration may be "0" or "unknown" during live stream
- After stream ends, yt-dlp can extract full video

```markdown
**Type**: Livestream (ended)
**Stream Date**: 2024-03-15
**Duration**: 02:15:30
```

## Layer 7: Integration with Transcript Workflows

### Filename Generation

```bash
# Safe filename from metadata
generate_filename() {
  local title="$1"
  local video_id="$2"

  # Sanitize title (remove special chars, limit length)
  safe_title=$(echo "$title" | tr -cd '[:alnum:] .-' | cut -c1-50 | sed 's/ /-/g')

  echo "${safe_title}-${video_id}.md"
}

# Example:
# "Building a CLI with Go" → "Building-a-CLI-with-Go-dQw4w9WgXcQ.md"
```

### Metadata in Frontmatter (YAML)

```yaml
---
type: youtube-transcript
video_id: dQw4w9WgXcQ
title: "Video Title"
channel: "Channel Name"
url: https://youtube.com/watch?v=dQw4w9WgXcQ
duration: "15:42"
published: 2024-03-15
transcribed: 2024-03-20
tags: [youtube, transcript, go, cli]
---
```

### Search Integration

Metadata enables full-text search across transcripts:
```bash
# Search by channel
grep -r "Channel: Venavi Tech" transcripts/

# Search by date range
grep -r "Published: 2024-03" transcripts/

# Search by duration (find long videos)
grep -r "Duration: [0-9][0-9]:[0-9][0-9]:" transcripts/
```

## Progressive Disclosure Application

**When to use which layer**:

- **Layer 1**: Always extract essential fields
- **Layer 2**: Validate URL before processing
- **Layer 3**: Extract metadata using yt-dlp
- **Layer 4**: Format for display in transcript
- **Layer 5**: Full archival and metadata preservation
- **Layer 6**: Handle error cases
- **Layer 7**: Integrate with file systems and workflows

**Default Behavior**: Extract Layer 1 fields, format with Layer 4, escalate as needed.
