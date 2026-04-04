---
name: daily-journal
description: Sentinel skill that proactively maintains a daily self-journal in an Obsidian vault. Automatically logs what the user accomplished at the end of each conversation or task — no manual trigger needed. Writes timestamped first-person entries with tags and wikilinks for Obsidian graph view. Also responds to "what did I do today", "review my day", "review my week", "search journal".
---

# Daily Journal (Sentinel)

Proactively maintains a daily self-journal in an Obsidian vault using plain markdown file I/O. This skill runs automatically — at the end of every conversation where meaningful work was done, summarize what was accomplished and log it to today's journal file. No need for the user to ask.

## When to Log (Automatic)

**At the end of every conversation where work was done**, before signing off:

1. **Read today's existing journal first** (if one exists) to understand what's already been logged
2. **Read memory.md** to understand recent context and narrative thread
3. Write a new entry that flows naturally from the existing entries — don't repeat what's already there, don't contradict earlier entries, and build on the narrative of the day
4. Log it as a timestamped entry to today's journal file
5. Update memory.md with a one-line summary
6. Briefly confirm: "Logged to today's journal."

**Continuity matters.** If the morning entry says "I started building the auth system", an afternoon entry should say "I finished the auth system and added tests" — not re-explain from scratch. The journal should read like a coherent story of the day, not isolated fragments.

**What counts as "work":** code written, bugs fixed, features built, PRs created, files modified, research done, designs discussed, deploys, reviews. If you did something the user would want to remember tomorrow, log it.

**What to skip:** trivial questions answered, no-op conversations, pure chat with no output.

## Writing Style

- Always write in **first person** ("I built...", "I fixed...", "I reviewed...")
- Be concise but capture the substance — what was done, what repo/project, and the outcome
- Group related work into one entry, don't list every file touched
- Include key details: PR numbers, branch names, skill names, error types fixed
- One entry per conversation, not one per task
- Include **[[wikilinks]]** for projects, skills, repos, and tools referenced
- Include **#tags** inline for work type and project

**Good entries:**
- `I built the obsidian-writer skill for the [[toolbelt]] repo. It supports full CRUD on Obsidian vaults via direct file I/O. #feature #project/toolbelt`
- `I fixed a config path bug in [[obsidian-writer]] where tilde expansion wasn't working. #bugfix #project/toolbelt`

**Bad entries:**
- `Worked on stuff.`
- `Modified SKILL.md, README.md, added icon.svg, ran tests, committed, pushed.` (too mechanical)
- `The user asked me to create a skill and I created it.` (third person, no substance)

## Step 1: Load Config

Check if the config file exists and contains the vault path:

```bash
cat ~/obsidian-skill/config 2>/dev/null
```

The config file should contain a single line: `OBSIDIAN_VAULT_PATH=/path/to/vault`

**If the config file is missing or does not contain `OBSIDIAN_VAULT_PATH`:**

1. Ask the user: "Where is your Obsidian vault? Please provide the full path (e.g. ~/Documents/MyVault)."
2. Expand `~` to `$HOME` in the path.
3. Write the config:

```bash
mkdir -p ~/obsidian-skill
# Expand ~ to $HOME in the user-provided path
OBSIDIAN_VAULT_PATH="${USER_PATH/#\~/$HOME}"
echo "OBSIDIAN_VAULT_PATH=\"$OBSIDIAN_VAULT_PATH\"" > ~/obsidian-skill/config
```

4. Verify the vault directory exists:

```bash
test -d "$OBSIDIAN_VAULT_PATH"
```

**If the vault directory does not exist**, ask the user: "That directory doesn't exist. Would you like me to create it as a new Obsidian vault?"

If they confirm:

```bash
mkdir -p "$OBSIDIAN_VAULT_PATH/.obsidian"
```

This creates the vault directory with the `.obsidian/` config folder that Obsidian expects.

**Load the config at the start of every operation:**

```bash
source ~/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (ask user for path)
fi
```

Do NOT proceed with any operation until `$OBSIDIAN_VAULT_PATH` is set and the directory exists.

**Ensure the daily-journal subfolder exists:**

```bash
mkdir -p "$OBSIDIAN_VAULT_PATH/daily-journal"
```

## Step 2: Log Entry

### Compute Dates and Paths

```bash
TODAY=$(date "+%Y-%m-%d")
YEAR=$(date "+%Y")
MONTH=$(date "+%m")
DAY_DISPLAY=$(date "+%A, %B %-d, %Y")
TIMESTAMP=$(date "+%-I:%M %p")
JOURNAL_DIR="$OBSIDIAN_VAULT_PATH/daily-journal/$YEAR/$MONTH"
JOURNAL_FILE="$JOURNAL_DIR/$TODAY.md"
MEMORY_FILE="$OBSIDIAN_VAULT_PATH/daily-journal/memory.md"
```

Create the journal directory for this month:

```bash
mkdir -p "$JOURNAL_DIR"
```

### Read Memory for Context

If memory.md exists, read it first to understand recent context. This gives the narrative thread so entries connect across days.

```bash
cat "$MEMORY_FILE" 2>/dev/null
```

### If Today's Journal File Does Not Exist — Create It

Write the file with YAML frontmatter, an H1 header with the date, and the first timestamped entry. Extract tags from the entry content automatically.

```bash
cat << JOURNALEOF > "$JOURNAL_FILE"
---
created: $TODAY
tags:
  - daily-journal
  - TAG1
  - TAG2
---

# $TODAY — $DAY_DISPLAY

## $TIMESTAMP
ENTRY_TEXT_HERE
JOURNALEOF
```

Replace `YYYY-MM-DD` with today's date, `Day, Month D, Year` with the display date, `HH:MM AM/PM` with the current time, `TAG1`/`TAG2` with extracted work-type and project tags, and write the actual first-person journal entry with [[wikilinks]] and #tags inline.

### If Today's File Exists — Read First, Then Append

Read today's existing entries so the new entry is coherent with what's already logged:

```bash
cat "$JOURNAL_FILE"
```

Review the existing content. Then write a new entry that builds on the day's narrative — don't repeat earlier entries.

**Append a new timestamped section:**

```bash
cat << JOURNALEOF >> "$JOURNAL_FILE"

## $TIMESTAMP
ENTRY_TEXT_HERE
JOURNALEOF
```

Replace `HH:MM AM/PM` with the current time and write the actual entry.

**Update frontmatter tags if new tags are introduced.** Read the file, check if any new tags from this entry are missing from the frontmatter `tags` array, and update the frontmatter if needed.

### Update memory.md After Logging

After writing the journal entry, update memory.md to maintain the rolling context window.

**If memory.md does not exist, create it:**

```bash
cat << MEMEOF > "$MEMORY_FILE"
---
last-updated: $TODAY
---

## Recent Context

- **$TODAY:** ENTRY_TEXT_HERE
MEMEOF
```

**If memory.md exists, update it:**

1. Read the current contents
2. Update the `last-updated` field in frontmatter to today's date
3. Add a new one-line summary at the top of the "## Recent Context" list: `- **$TODAY:** Summary with [[wikilinks]] and #tags`
4. Remove entries older than 7 days
5. Write the updated file back

**7-day rotation — concrete approach:**

```bash
# Get cutoff date (7 days ago on macOS)
CUTOFF=$(date -v-7d "+%Y-%m-%d")

# Read memory.md, keep frontmatter and only entries where date >= CUTOFF
awk -v cutoff="$CUTOFF" '
  /^---$/ { front++; print; next }
  front < 2 { print; next }
  /^\- \*\*[0-9]{4}-[0-9]{2}-[0-9]{2}:\*\*/ {
    match($0, /[0-9]{4}-[0-9]{2}-[0-9]{2}/)
    d = substr($0, RSTART, RLENGTH)
    if (d >= cutoff) print
    next
  }
  { print }
' "$MEMORY_FILE" > "${MEMORY_FILE}.tmp" && mv "${MEMORY_FILE}.tmp" "$MEMORY_FILE"
```

memory.md format:

```markdown
---
last-updated: YYYY-MM-DD
---

## Recent Context

- **2026-04-04:** Built the obsidian-writer skill for [[toolbelt]]. Full CRUD via file I/O. #feature #project/toolbelt
- **2026-04-03:** Created notes-manager skill, pushed to [[toolbelt]]. #feature #project/toolbelt
```

**Confirm:** "Logged to today's journal."

## Tags and Wikilinks Guidance

Automatically extract and apply tags and wikilinks to every journal entry.

### Tags (in frontmatter AND inline)

**Work type tags:**
- `#bugfix` — bug fixed
- `#feature` — new feature built
- `#refactor` — code refactored
- `#review` — code review done
- `#docs` — documentation written
- `#deploy` — deployment performed
- `#research` — research or investigation done

**Project tags:**
- `#project/repo-name` — the repository worked on (e.g. `#project/toolbelt`)
- `#project/skill-name` — the skill worked on (e.g. `#project/daily-journal`)

Tags go in both places: the frontmatter `tags` array (without `#` prefix) and inline in the entry text (with `#` prefix).

### Wikilinks

- `[[project-name]]` for repos worked on (e.g. `[[toolbelt]]`)
- `[[skill-name]]` for skills created or modified (e.g. `[[obsidian-writer]]`, `[[daily-journal]]`)
- `[[tool-name]]` for tools referenced (e.g. `[[yt-dlp]]`, `[[ripgrep]]`)

Obsidian creates graph nodes even for notes that don't exist yet, so wikilinks are always useful for building the knowledge graph.

## User-Triggered Operations

These run only when the user explicitly asks.

**Always run Step 1 (Load Config) first before any operation below.** After sourcing the config, verify the vault path is valid:

```bash
source ~/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (Step 1) — ask user for vault path, write config
fi
```

### View Today

```bash
source ~/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (Step 1)
fi
TODAY=$(date "+%Y-%m-%d")
YEAR=$(date "+%Y")
MONTH=$(date "+%m")
JOURNAL_FILE="$OBSIDIAN_VAULT_PATH/daily-journal/$YEAR/$MONTH/$TODAY.md"
if [ -f "$JOURNAL_FILE" ]; then
  cat "$JOURNAL_FILE"
else
  echo "No journal entry for today yet."
fi
```

Read and display the contents of today's journal file.

### View Date

Parse the user's date to `YYYY-MM-DD` format, then compute the path and read:

```bash
source ~/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (Step 1)
fi
TARGET_DATE="YYYY-MM-DD"  # parsed from user input
TARGET_YEAR=$(echo "$TARGET_DATE" | cut -d- -f1)
TARGET_MONTH=$(echo "$TARGET_DATE" | cut -d- -f2)
JOURNAL_FILE="$OBSIDIAN_VAULT_PATH/daily-journal/$TARGET_YEAR/$TARGET_MONTH/$TARGET_DATE.md"
if [ -f "$JOURNAL_FILE" ]; then
  cat "$JOURNAL_FILE"
else
  echo "No journal entry for $TARGET_DATE."
fi
```

### View Week

```bash
source ~/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (Step 1)
fi
find "$OBSIDIAN_VAULT_PATH/daily-journal" -name "*.md" -not -name "memory.md" -type f | sort -r | head -7
```

Read each of the returned files and display their contents. For a **weekly summary**, read all entries from the last 7 days and produce a concise first-person summary grouped by theme (features, fixes, reviews, learning).

### Search Journal

```bash
source ~/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (Step 1)
fi
grep -rl "SEARCH_TERM" "$OBSIDIAN_VAULT_PATH/daily-journal" --include="*.md" | head -20
```

Replace `SEARCH_TERM` with the user's query. Cap results at 20. For each match, show the filename and a snippet of the matching line. If more than 20 results exist, suggest the user refine their search.

## Errors

| Error | What to do |
|---|---|
| Config file missing | Run the first-use flow from Step 1 — ask for vault path, write config |
| Vault path doesn't exist | Warn the user the configured path no longer exists. Offer to reconfigure by re-running Step 1 |
| No journal for today | Create today's journal file with the current entry |
| No journal for requested date | Tell the user no entry exists for that date |
| File write fails | Report the error and suggest checking file permissions on the vault directory |
