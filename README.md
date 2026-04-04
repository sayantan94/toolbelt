# Toolbelt

Shareable skills. Install a skill, then just ask Claude — it handles the rest.

## Skills

| | Skill | What it does                                                                                                              |
|-|-------|---------------------------------------------------------------------------------------------------------------------------|
| <img src="assets/video-downloader-icon.svg" width="20"/> | [**video-downloader**](video-downloader/) | Say "download this video" + paste a URL. Claude downloads it via yt-dlp. Supports YouTube, Reddit, Twitter/X, and others. |
| <img src="assets/notes-manager-icon.svg" width="20"/> | [**notes-manager**](notes-manager/) | Say "create a note", "search notes", or "list my notes". Claude manages your macOS Notes app — create, read, search, append, delete, and organize notes and folders. |
| <img src="assets/daily-journal-icon.svg" width="20"/> | [**daily-journal**](daily-journal/) | Say "journal this" or "what did I do today". Claude keeps a daily self-journal in your Obsidian vault — timestamped first-person entries with tags and wikilinks for graph view, daily review, weekly summaries. |
| <img src="assets/obsidian-writer-icon.svg" width="20"/> | [**obsidian-writer**](obsidian-writer/) | Say "create a note in obsidian", "search my notes", or "list obsidian folders". Claude manages your Obsidian vault — create, read, search, append, delete, and organize markdown notes. |

## Install

### Step 1: Clone this repo

```bash
git clone https://github.com/sayantan94/toolbelt.git
cd toolbelt
```

### Step 2: Pick a skill and symlink it

Each folder is a skill. To add one to your Claude, symlink it into `~/.claude/skills/`:

```bash
# Install all skills
ln -s "$(pwd)/video-downloader" ~/.claude/skills/video-downloader
ln -s "$(pwd)/notes-manager" ~/.claude/skills/notes-manager
ln -s "$(pwd)/daily-journal" ~/.claude/skills/daily-journal
ln -s "$(pwd)/obsidian-writer" ~/.claude/skills/obsidian-writer
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

> Journal this — I built a notes manager skill today

> What did I do today?

> Review my week

> Create a note called "Project Ideas" in my obsidian vault

> Search my obsidian notes for "authentication"

> List notes in the projects folder
```

Claude recognizes the intent, the skill activates, and handles the rest.

### Daily Journal

The daily-journal is a **sentinel skill** — it runs automatically at the end of every work session where something meaningful was done. No need to ask.

**To enable automatic journaling**, add this to your `~/.claude/CLAUDE.md`:

```markdown
## Daily Journal (MANDATORY)

BEFORE completing any conversation where code was written, files were modified, features were built, bugs were fixed, or any meaningful work was done:

1. Check if `~/.claude/skills/daily-journal/SKILL.md` exists
2. If it does, invoke the daily-journal skill to log a first-person summary of what was accomplished
3. Confirm: "Logged to today's journal."

This is not optional. Every working session must be journaled. Do not skip this step.
```

This tells Claude to journal every session automatically. The skill handles the rest — it creates the Obsidian vault on first use, writes timestamped entries with tags and `[[wikilinks]]`, and maintains a rolling `memory.md` for continuity across sessions.

**To trigger it manually:**

```
> Journal this — I built the auth system today

> What did I do today?

> Review my week

> Search journal for "obsidian"
```

## License

MIT
