# Doveaia — Claude Code Marketplace

[Claude Code](https://docs.claude.com/en/docs/claude-code/overview) plugin marketplace maintained by [James K. GAGLO](https://github.com/doveaia).

## Available plugins

| Plugin | Version | Description |
|--------|---------|-------------|
| [`transcribe`](./plugins/transcribe) | 3.0.0 | Transcribes YouTube videos and local video/audio files with support for several transcription backends (whisper.cpp, OpenAI Whisper, Whisper API, macOS Speech). |
| [`mattpocock-skills-addons`](./plugins/mattpocock-skills-addons) | 0.4.0 | Layers on top of Matt Pocock's skills through their per-repo config: a GitHub Projects board for `triage`, kept in sync with labels. Requires the `mattpocock-skills` plugin. |

## Installation

### 1. Add the marketplace

Three ways to add this marketplace, pick one:

#### a. From Claude Code (slash command)

```
/plugin marketplace add doveaia/claude-marketplace
```

#### b. From the terminal (Claude Code CLI)

```bash
claude plugin marketplace add doveaia/claude-marketplace
```

You can also point to the full Git repository:

```bash
claude plugin marketplace add git@github.com:doveaia/claude-marketplace.git
# or over HTTPS
claude plugin marketplace add https://github.com/doveaia/claude-marketplace.git
```

#### c. Via `git clone` (local path)

Useful for developing or modifying the plugins locally:

```bash
git clone git@github.com:doveaia/claude-marketplace.git ~/claude-marketplace
claude plugin marketplace add ~/claude-marketplace
```

Or from Claude Code once the repo is cloned:

```
/plugin marketplace add ~/claude-marketplace
```

### 2. Install a plugin

Once the marketplace is added, install the plugin of your choice:

```
/plugin install transcribe@doveaia
```

You should see:

```
✓ Installed transcribe. Run /reload-plugins to apply.
```

The general syntax is `/plugin install <plugin-name>@doveaia`.

### 3. Reload the plugins

To activate the plugin without restarting Claude Code:

```
/reload-plugins
```

### 4. Verify the installation

```
/plugin list
```

The installed plugin appears in the list and its skills/agents/commands become available in Claude Code.

## Useful commands

| Command | Effect |
|---------|--------|
| `/plugin marketplace list` | Lists the added marketplaces |
| `/plugin marketplace update doveaia` | Refreshes the marketplace's plugin list |
| `/plugin marketplace remove doveaia` | Removes the marketplace |
| `/plugin install <name>@doveaia` | Installs a plugin |
| `/plugin uninstall <name>` | Uninstalls a plugin |
| `/plugin list` | Lists the installed plugins |
| `/reload-plugins` | Reloads the plugins (after install/update) without restarting Claude Code |

## Slash commands provided by the plugins

Once installed, the plugins expose slash commands usable directly in Claude Code.

### `transcribe` plugin

| Command | Description |
|---------|-------------|
| `/transcribe <youtube-url>` | Transcribes a YouTube video to a markdown file with timestamps and metadata. |
| `/transcribe <file-path>` | Transcribes a local video or audio file (MP4, MKV, MOV, MP3, M4A, WAV, FLAC, etc.). |

Automatic behaviour included:
- Source language detection (French videos stay in French).
- For a non-English video: generates two files (`<name>-<lang>.md` + a translated `<name>-en.md`).
- Automatic selection of the available transcription backend (whisper.cpp → OpenAI Whisper → Whisper API → macOS Speech).
- Cleanup of temporary files.

> The internal skills (`transcription-backends`, `youtube-metadata`) and sub-agents (`audio-transcriber`, `media-processor`) are not invoked directly: they are orchestrated by `/transcribe`.

## Dependencies per plugin

Each plugin may have its own external dependencies (binaries, services). See the plugin's README for the detailed list:

- **transcribe** → `yt-dlp` (YouTube), `ffmpeg` (local videos) and at least one transcription backend. See [plugins/transcribe/README.md](./plugins/transcribe/README.md).

## Marketplace structure

```
.
├── .claude-plugin/
│   └── marketplace.json        # Marketplace manifest
└── plugins/
    └── transcribe/
        ├── .claude-plugin/
        │   └── plugin.json     # Plugin manifest
        ├── agents/             # Claude sub-agents
        ├── skills/             # Invocable skills
        ├── README.md
        └── CHANGELOG.md
```

## Contributing

To add a new plugin:

1. Create a folder under `plugins/<plugin-name>/`.
2. Add a manifest `plugins/<plugin-name>/.claude-plugin/plugin.json` (fields: `name`, `description`, `version`, `author`).
3. Reference the plugin in `.claude-plugin/marketplace.json`:
   ```json
   {
     "name": "<plugin-name>",
     "source": "./plugins/<plugin-name>",
     "description": "..."
   }
   ```
4. Open a Pull Request.

## License

See each plugin for its own license.
