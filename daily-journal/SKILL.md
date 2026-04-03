---
name: daily-journal
description: Sentinel skill that proactively maintains a daily self-journal in macOS Notes app. Automatically logs what the user accomplished at the end of each conversation or task — no manual trigger needed. Writes timestamped first-person entries. Also responds to "what did I do today", "review my day", "review my week", "search journal".
---

# Daily Journal (Sentinel)

Proactively maintains a daily self-journal in macOS Notes.app. This skill runs automatically — at the end of every conversation where meaningful work was done, summarize what was accomplished and log it to today's journal note. No need for the user to ask.

## When to Log (Automatic)

**At the end of every conversation where work was done**, before signing off:

1. **Read today's existing journal first** (if one exists) to understand what's already been logged
2. Write a new entry that flows naturally from the existing entries — don't repeat what's already there, don't contradict earlier entries, and build on the narrative of the day
3. Log it as a timestamped entry to today's journal note
4. Briefly confirm: "Logged to today's journal."

**Continuity matters.** If the morning entry says "I started building the auth system", an afternoon entry should say "I finished the auth system and added tests" — not re-explain from scratch. The journal should read like a coherent story of the day, not isolated fragments.

**What counts as "work":** code written, bugs fixed, features built, PRs created, files modified, research done, designs discussed, deploys, reviews. If you did something the user would want to remember tomorrow, log it.

**What to skip:** trivial questions answered, no-op conversations, pure chat with no output.

## Writing Style

- Always write in **first person** ("I built...", "I fixed...", "I reviewed...")
- Be concise but capture the substance — what was done, what repo/project, and the outcome
- Group related work into one entry, don't list every file touched
- Include key details: PR numbers, branch names, skill names, error types fixed
- One entry per conversation, not one per task

**Good entries:**
- `I built the notes-manager skill for the toolbelt repo. It supports full CRUD on Notes.app via AppleScript and pandoc.`
- `I fixed a quoting bug in notes-manager where pandoc HTML broke AppleScript strings. Added sed escaping to the markdown pipeline.`
- `I set up the daily-journal sentinel skill and pushed it to toolbelt. It auto-logs what I do each day.`

**Bad entries:**
- `Worked on stuff.`
- `Modified SKILL.md, README.md, added icon.svg, ran tests, committed, pushed.` (too mechanical)
- `The user asked me to create a skill and I created it.` (third person, no substance)

## Step 1: Check Dependencies

```bash
command -v osascript && command -v pandoc
```

If `pandoc` is missing:
```bash
brew install pandoc
```

## Step 2: Ensure Journal Folder Exists

```bash
osascript -e 'tell application "Notes"
  try
    get folder "Daily Journal"
  on error
    make new folder with properties {name:"Daily Journal"}
  end try
end tell'
```

## Step 3: Log Entry

### Get Today's Date and Time

```bash
TODAY=$(date "+%Y-%m-%d")
DAY_DISPLAY=$(date "+%A, %B %-d, %Y")
TIMESTAMP=$(date "+%-I:%M %p")
```

### Check If Today's Note Exists

```bash
osascript -e 'tell application "Notes"
  tell folder "Daily Journal"
    set matchingNotes to every note whose name contains "'"$TODAY"'"
    if (count of matchingNotes) > 0 then
      return "exists"
    else
      return "missing"
    end if
  end tell
end tell'
```

### If Missing — Create Today's Note

```bash
TMPFILE=$(mktemp /tmp/claude-journal-XXXXX.md)
cat << MDEOF > "$TMPFILE"
**$TIMESTAMP** — ENTRY_TEXT_HERE
MDEOF
HTML=$(pandoc "$TMPFILE" -f markdown -t html)
rm "$TMPFILE"
HTML=$(echo "$HTML" | sed 's/"/\\"/g')
osascript -e "tell application \"Notes\"
  tell folder \"Daily Journal\"
    make new note with properties {body:\"<h1>$TODAY</h1><h2>$DAY_DISPLAY</h2>\" & \"$HTML\"}
  end tell
end tell"
```

### If Exists — Read First, Then Append

**Read today's existing entries first** so the new entry is coherent with what's already logged:

```bash
osascript -e 'tell application "Notes"
  tell folder "Daily Journal"
    set theNote to first note whose name contains "'"$TODAY"'"
    return plaintext of theNote
  end tell
end tell'
```

Review the existing content. Then write a new entry that builds on the day's narrative — don't repeat earlier entries.

**Append the new entry:**

```bash
TMPFILE=$(mktemp /tmp/claude-journal-XXXXX.md)
cat << MDEOF > "$TMPFILE"
**$TIMESTAMP** — ENTRY_TEXT_HERE
MDEOF
HTML=$(pandoc "$TMPFILE" -f markdown -t html)
rm "$TMPFILE"
HTML=$(echo "$HTML" | sed 's/"/\\"/g')
osascript -e "tell application \"Notes\"
  tell folder \"Daily Journal\"
    set theNote to first note whose name contains \"$TODAY\"
    set body of theNote to (body of theNote) & \"$HTML\"
  end tell
end tell"
```

Replace `ENTRY_TEXT_HERE` with the first-person journal entry that flows from existing entries.

Confirm briefly: "Logged to today's journal."

## User-Triggered Operations

These run only when the user explicitly asks.

### View Today

```bash
TODAY=$(date "+%Y-%m-%d")
osascript -e 'tell application "Notes"
  tell folder "Daily Journal"
    set matchingNotes to every note whose name contains "'"$TODAY"'"
    if (count of matchingNotes) > 0 then
      return plaintext of first item of matchingNotes
    else
      return "No journal entry for today yet."
    end if
  end tell
end tell'
```

### View Date

Parse the user's date to `YYYY-MM-DD` format, then:

```bash
osascript -e 'tell application "Notes"
  tell folder "Daily Journal"
    set matchingNotes to every note whose name contains "TARGET_DATE"
    if (count of matchingNotes) > 0 then
      return plaintext of first item of matchingNotes
    else
      return "No journal entry for that date."
    end if
  end tell
end tell'
```

### View Week

```bash
osascript -e 'tell application "Notes"
  tell folder "Daily Journal"
    set allNotes to every note
    set output to ""
    set noteCount to 0
    repeat with n in allNotes
      if noteCount is less than 7 then
        set output to output & name of n & " | " & modification date of n & linefeed
        set noteCount to noteCount + 1
      end if
    end repeat
    return output
  end tell
end tell'
```

For a **weekly summary**, read the last 7 notes and produce a concise first-person summary grouped by theme (features, fixes, reviews, learning).

### Search Journal

```bash
osascript -e 'tell application "Notes"
  tell folder "Daily Journal"
    set matchingNotes to every note whose plaintext contains "SEARCH_TERM"
    set results to {}
    set resultCount to 0
    repeat with n in matchingNotes
      if resultCount is less than 20 then
        set end of results to (name of n & " | modified: " & (modification date of n as string))
        set resultCount to resultCount + 1
      end if
    end repeat
    return results
  end tell
end tell'
```

## Errors

| Error | What to do |
|-------|------------|
| No journal for today | Create today's note with the current entry |
| No journal for that date | Tell the user no entry exists for that date |
| Folder not found | Auto-create "Daily Journal" folder (Step 2) |
| Permission denied | System Settings → Privacy & Security → Automation → enable Terminal to control Notes |
| pandoc not found | Install via `brew install pandoc` |
