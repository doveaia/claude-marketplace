# Doveaia — Claude Code Marketplace

Marketplace de plugins [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) maintenu par [James K. GAGLO](https://github.com/doveaia).

## Plugins disponibles

| Plugin | Version | Description |
|--------|---------|-------------|
| [`transcribe`](./plugins/transcribe) | 3.0.0 | Transcrit des vidéos YouTube et des fichiers vidéo/audio locaux avec support de plusieurs backends de transcription (whisper.cpp, OpenAI Whisper, Whisper API, macOS Speech). |

## Installation

### 1. Ajouter le marketplace

Trois façons d'ajouter ce marketplace, au choix :

#### a. Depuis Claude Code (slash command)

```
/plugin marketplace add doveaia/claude-marketplace
```

#### b. Depuis le terminal (CLI Claude Code)

```bash
claude plugin marketplace add doveaia/claude-marketplace
```

Tu peux aussi pointer vers le dépôt Git complet :

```bash
claude plugin marketplace add git@github.com:doveaia/claude-marketplace.git
# ou en HTTPS
claude plugin marketplace add https://github.com/doveaia/claude-marketplace.git
```

#### c. Via `git clone` (chemin local)

Utile pour développer/modifier les plugins localement :

```bash
git clone git@github.com:doveaia/claude-marketplace.git ~/claude-marketplace
claude plugin marketplace add ~/claude-marketplace
```

Ou depuis Claude Code une fois le repo cloné :

```
/plugin marketplace add ~/claude-marketplace
```

### 2. Installer un plugin

Une fois le marketplace ajouté, installe le plugin de ton choix :

```
/plugin install transcribe@doveaia
```

La syntaxe générale est `/plugin install <nom-du-plugin>@doveaia`.

### 3. Vérifier l'installation

```
/plugin list
```

Le plugin installé apparaît dans la liste et ses skills/agents/commandes deviennent disponibles dans Claude Code.

## Commandes utiles

| Commande | Effet |
|----------|-------|
| `/plugin marketplace list` | Liste les marketplaces ajoutés |
| `/plugin marketplace update doveaia` | Met à jour la liste des plugins du marketplace |
| `/plugin marketplace remove doveaia` | Retire le marketplace |
| `/plugin install <name>@doveaia` | Installe un plugin |
| `/plugin uninstall <name>` | Désinstalle un plugin |
| `/plugin list` | Liste les plugins installés |

## Slash commands fournis par les plugins

Une fois installés, les plugins exposent des slash commands utilisables directement dans Claude Code.

### Plugin `transcribe`

| Commande | Description |
|----------|-------------|
| `/transcribe <url-youtube>` | Transcrit une vidéo YouTube vers un fichier markdown avec timestamps et métadonnées. |
| `/transcribe <chemin-fichier>` | Transcrit un fichier vidéo ou audio local (MP4, MKV, MOV, MP3, M4A, WAV, FLAC, etc.). |

Comportement automatique inclus :
- Détection de la langue d'origine (les vidéos françaises restent en français).
- Pour une vidéo non-anglophone : génère deux fichiers (`<nom>-<lang>.md` + `<nom>-en.md` traduit).
- Sélection automatique du backend de transcription disponible (whisper.cpp → OpenAI Whisper → Whisper API → macOS Speech).
- Nettoyage des fichiers temporaires.

> Les skills internes (`transcription-backends`, `youtube-metadata`) et les sous-agents (`audio-transcriber`, `media-processor`) ne s'invoquent pas directement : ils sont orchestrés par `/transcribe`.

## Dépendances par plugin

Chaque plugin peut avoir ses propres dépendances externes (binaires, services). Consulte le README du plugin pour la liste détaillée :

- **transcribe** → `yt-dlp` (YouTube), `ffmpeg` (vidéos locales) et au moins un backend de transcription. Voir [plugins/transcribe/README.md](./plugins/transcribe/README.md).

## Structure du marketplace

```
.
├── .claude-plugin/
│   └── marketplace.json        # Manifeste du marketplace
└── plugins/
    └── transcribe/
        ├── .claude-plugin/
        │   └── plugin.json     # Manifeste du plugin
        ├── agents/             # Sous-agents Claude
        ├── skills/             # Skills invocables
        ├── README.md
        └── CHANGELOG.md
```

## Contribuer

Pour ajouter un nouveau plugin :

1. Crée un dossier sous `plugins/<nom-du-plugin>/`.
2. Ajoute un manifeste `plugins/<nom-du-plugin>/.claude-plugin/plugin.json` (champs : `name`, `description`, `version`, `author`).
3. Référence le plugin dans `.claude-plugin/marketplace.json` :
   ```json
   {
     "name": "<nom-du-plugin>",
     "source": "./plugins/<nom-du-plugin>",
     "description": "..."
   }
   ```
4. Ouvre une Pull Request.

## Licence

Voir chaque plugin pour sa licence propre.
