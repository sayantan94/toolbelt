# Toolbelt

Shareable skills. Install a skill, then just ask Claude — it handles the rest.

## Skills

| | Skill | What it does                                                                                                              |
|-|-------|---------------------------------------------------------------------------------------------------------------------------|
| <img src="assets/video-downloader-icon.svg" width="20"/> | [**video-downloader**](video-downloader/) | Say "download this video" + paste a URL. Claude downloads it via yt-dlp. Supports YouTube, Reddit, Twitter/X, and others. |
| <img src="assets/notes-manager-icon.svg" width="20"/> | [**notes-manager**](notes-manager/) | Say "create a note", "search notes", or "list my notes". Claude manages your macOS Notes app — create, read, search, append, delete, and organize notes and folders. |

## Install

### Step 1: Clone this repo

```bash
git clone https://github.com/sayantan94/toolbelt.git
cd toolbelt
```

### Step 2: Pick a skill and symlink it

Each folder is a skill. To add one to your Claude, symlink it into `~/.claude/skills/`:

```bash
# Example: install the video-downloader skill
ln -s "$(pwd)/video-downloader" ~/.claude/skills/video-downloader
```

This creates a link so Claude can find the skill. Any time you `git pull` to update the repo, the skill updates too.

### Step 3: Restart Claude Code

Skills are loaded when a conversation starts. Open a new conversation for Claude to pick it up.

### Step 4: Verify

Type `/skills` in Claude Code. You should see your skill in the list. If it's there, you're good — just start using it.


### How it works

Claude reads the skill's description at conversation start. When you say something that matches — like "download this video" — Claude loads the full `SKILL.md` and follows the step-by-step instructions inside it. No server, no plugins, no config files. Just a markdown file that teaches Claude what to do.

## Usage

Just talk to Claude:

```
> Download this video https://youtube.com/watch?v=dQw4w9WgXcQ

> Grab the audio from https://soundcloud.com/artist/track

> Download this in 720p https://vimeo.com/123456789

> Create a note called "Shopping List" with milk, eggs, bread

> Search my notes for "meeting"

> List my notes in the Work folder

> Append "buy flowers" to my Shopping List note

> Delete the note called "Old Draft"
```

Claude recognizes the intent, the skill activates, and handles the rest.

## License

MIT
