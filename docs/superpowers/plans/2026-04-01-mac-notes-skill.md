# mac-notes Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Claude Code skill that provides full CRUD, search, and folder management for macOS Notes.app via AppleScript and pandoc.

**Architecture:** Single `SKILL.md` file containing all instructions. Claude reads the skill at invocation and follows the step-by-step guide, executing `osascript` and `pandoc` commands via Bash. The skill teaches Claude how to handle each operation — it's a markdown instruction file, not executable code.

**Tech Stack:** AppleScript (`osascript`), pandoc, macOS Notes.app

---

## File Structure

```
notes-manager/
  SKILL.md       # Complete skill definition — all AppleScript commands and workflow instructions
```

Single file. No helper scripts. Matches the existing `video-downloader/SKILL.md` pattern.

---

### Task 1: Create SKILL.md with Frontmatter and Dependency Check

**Files:**
- Create: `notes-manager/SKILL.md`

- [ ] **Step 1: Create the skill file with frontmatter and dependency check section**

Create `notes-manager/SKILL.md` with this content:

```markdown
---
name: notes-manager
description: Interact with macOS Notes app. Use when user says "create a note", "save this to notes", "jot this down", "find my note about", "search notes", "list my notes", "show notes in folder", "append to note", "add this to my note", "delete note", "move note to folder", "create a folder", or wants to read, search, organize, or manage their Apple Notes.
---

# Notes Manager

Interact with macOS Notes.app — create, read, search, list, append, delete, and organize notes and folders.

## Step 1: Check Dependencies

Run both checks. If `pandoc` is missing, stop and show the install command — do NOT proceed.

\```bash
command -v osascript && command -v pandoc
\```

If `pandoc` is missing:
\```bash
brew install pandoc
\```

## Step 2: Determine the Operation

Based on what the user asked, identify which operation to perform:

| User intent | Operation |
|-------------|-----------|
| "create a note", "save this to notes", "jot this down" | **Create Note** |
| "find my note about X", "search notes for X" | **Search Notes** |
| "read note X", "show me the note about X" | **Read Note** |
| "list my notes", "show notes in folder X" | **List Notes** |
| "add this to my note about X", "append to note X" | **Append to Note** |
| "delete the note called X" | **Delete Note** |
| "list folders", "show my folders" | **List Folders** |
| "create a folder called X" | **Create Folder** |
| "move note X to folder Y" | **Move Note** |

Then follow the corresponding section below.
```

- [ ] **Step 2: Verify the file exists and frontmatter is valid**

Run: `head -5 notes-manager/SKILL.md`
Expected: The YAML frontmatter with `name: notes-manager` and `description: ...`

---

### Task 2: Add Markdown-to-HTML Pipeline Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add the markdown conversion helper section**

Append this section to `notes-manager/SKILL.md` after the operation table:

```markdown
## Markdown-to-HTML Conversion

When creating or appending notes with markdown-formatted content, convert to HTML first:

\```bash
TMPFILE=$(mktemp /tmp/claude-note-XXXXX.md)
cat << 'MDEOF' > "$TMPFILE"
<markdown content here>
MDEOF
HTML=$(pandoc "$TMPFILE" -f markdown -t html)
rm "$TMPFILE"
\```

For plain text with no markdown formatting, skip pandoc and wrap in `<p>` tags:

\```bash
HTML="<p>plain text content here</p>"
\```

Use the `$HTML` variable in the AppleScript commands below.
```

- [ ] **Step 2: Verify the section was appended**

Run: `tail -20 notes-manager/SKILL.md`
Expected: The Markdown-to-HTML Conversion section appears at the end.

---

### Task 3: Add Default Folder Behavior Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add default folder section**

Append this section to `notes-manager/SKILL.md`:

```markdown
## Default Folder

New notes go to a folder called **"Claude Notes"** by default. If the user specifies a different folder (e.g., "save this note in Work Ideas"), use that instead.

Before creating a note, ensure the target folder exists:

\```bash
osascript -e 'tell application "Notes"
  try
    get folder "FOLDER_NAME"
  on error
    make new folder with properties {name:"FOLDER_NAME"}
  end try
end tell'
\```

Replace `FOLDER_NAME` with "Claude Notes" (default) or the user-specified folder name.
```

- [ ] **Step 2: Verify the section was appended**

Run: `tail -15 notes-manager/SKILL.md`
Expected: The Default Folder section with the AppleScript for folder creation.

---

### Task 4: Add Create Note Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add the create note operation**

Append this section to `notes-manager/SKILL.md`:

```markdown
## Create Note

### Title Resolution

1. If the user provided a title ("create a note called Meeting Notes"), use that
2. If not, infer from the first heading or first line of the content
3. If neither is clear, ask the user for a title

### Creating the Note

First, convert content to HTML using the Markdown-to-HTML Conversion steps above. Then:

\```bash
osascript -e 'tell application "Notes"
  tell folder "FOLDER_NAME"
    make new note with properties {body:"<h1>NOTE_TITLE</h1>" & "HTML_CONTENT"}
  end tell
end tell'
\```

Replace:
- `FOLDER_NAME` with the target folder (default: "Claude Notes")
- `NOTE_TITLE` with the note title
- `HTML_CONTENT` with the converted HTML

**Important:** The note title in Notes.app is derived from the first `<h1>` tag in the body. Always include `<h1>NOTE_TITLE</h1>` at the start.

### Confirm Result

After creating, tell the user:
- Note title
- Folder it was saved to
```

- [ ] **Step 2: Verify the section was appended**

Run: `grep -c "Create Note" notes-manager/SKILL.md`
Expected: At least 1 match.

---

### Task 5: Add Search and Read Notes Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add the search/read operation**

Append this section to `notes-manager/SKILL.md`:

```markdown
## Search Notes

Search by title:

\```bash
osascript -e 'tell application "Notes"
  set matchingNotes to every note whose name contains "SEARCH_TERM"
  set output to ""
  repeat with n in matchingNotes
    set output to output & name of n & " | " & modification date of n & linefeed
  end repeat
  return output
end tell'
\```

Search by body content:

\```bash
osascript -e 'tell application "Notes"
  set matchingNotes to every note whose plaintext contains "SEARCH_TERM"
  set output to ""
  repeat with n in matchingNotes
    set output to output & name of n & " | " & modification date of n & linefeed
  end repeat
  return output
end tell'
\```

Replace `SEARCH_TERM` with the user's search query.

If multiple notes match, show the list and let the user pick which one to read.

## Read Note

After the user selects a note (or if only one matched), read its content:

\```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "NOTE_TITLE"
  return plaintext of theNote
end tell'
\```

Display the plaintext content in the terminal. Use `plaintext` (not `body`) for readable terminal output.
```

- [ ] **Step 2: Verify the sections were appended**

Run: `grep -c "Search Notes\|Read Note" notes-manager/SKILL.md`
Expected: 2 matches.

---

### Task 6: Add List Notes Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add the list notes operation**

Append this section to `notes-manager/SKILL.md`:

```markdown
## List Notes

List all notes (default limit 20, most recent first):

\```bash
osascript -e 'tell application "Notes"
  set allNotes to every note
  set output to ""
  set noteCount to 0
  repeat with n in allNotes
    if noteCount is less than 20 then
      set output to output & name of n & " | " & modification date of n & linefeed
      set noteCount to noteCount + 1
    end if
  end repeat
  return output
end tell'
\```

List notes in a specific folder:

\```bash
osascript -e 'tell application "Notes"
  set folderNotes to every note of folder "FOLDER_NAME"
  set output to ""
  set noteCount to 0
  repeat with n in folderNotes
    if noteCount is less than 20 then
      set output to output & name of n & " | " & modification date of n & linefeed
      set noteCount to noteCount + 1
    end if
  end repeat
  return output
end tell'
\```

Replace `FOLDER_NAME` with the user's specified folder. If the user asks for more than 20, adjust the limit accordingly.

Show the results as a numbered list with title and last modified date.
```

- [ ] **Step 2: Verify the section was appended**

Run: `grep -c "List Notes" notes-manager/SKILL.md`
Expected: At least 1 match.

---

### Task 7: Add Append to Note Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add the append operation**

Append this section to `notes-manager/SKILL.md`:

```markdown
## Append to Note

### Find the Target Note

\```bash
osascript -e 'tell application "Notes"
  set matchingNotes to every note whose name contains "SEARCH_TERM"
  set output to ""
  repeat with n in matchingNotes
    set output to output & name of n & " | " & modification date of n & linefeed
  end repeat
  return output
end tell'
\```

If multiple notes match, show the list and ask the user which one to append to. **Always confirm the matched note before appending.**

### Append Content

First, convert new content to HTML using the Markdown-to-HTML Conversion steps. Then:

\```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "NOTE_TITLE"
  set body of theNote to (body of theNote) & "HTML_CONTENT"
end tell'
\```

Replace:
- `NOTE_TITLE` with the exact name of the matched note
- `HTML_CONTENT` with the converted HTML

### Confirm Result

Tell the user what was appended and to which note.
```

- [ ] **Step 2: Verify the section was appended**

Run: `grep -c "Append to Note\|Append Content" notes-manager/SKILL.md`
Expected: 2 matches.

---

### Task 8: Add Delete Note Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add the delete operation**

Append this section to `notes-manager/SKILL.md`:

```markdown
## Delete Note

### Find the Note

\```bash
osascript -e 'tell application "Notes"
  set matchingNotes to every note whose name contains "SEARCH_TERM"
  set output to ""
  repeat with n in matchingNotes
    set output to output & name of n & " | " & modification date of n & linefeed
  end repeat
  return output
end tell'
\```

If multiple notes match, show the list and ask the user which one to delete.

### Confirm Before Deleting

**Always ask the user:** "Are you sure you want to delete the note '[NOTE_TITLE]'?" Do NOT delete without confirmation.

### Delete

\```bash
osascript -e 'tell application "Notes"
  delete (first note whose name is "NOTE_TITLE")
end tell'
\```

Tell the user the note was deleted.

**Important:** Never support bulk deletion. Only delete one note at a time.
```

- [ ] **Step 2: Verify the section was appended**

Run: `grep -c "Delete Note" notes-manager/SKILL.md`
Expected: At least 1 match.

---

### Task 9: Add Folder Management Section

**Files:**
- Modify: `notes-manager/SKILL.md`

- [ ] **Step 1: Add folder management operations**

Append this section to `notes-manager/SKILL.md`:

```markdown
## List Folders

\```bash
osascript -e 'tell application "Notes" to get name of every folder'
\```

Show the folder names as a list.

## Create Folder

\```bash
osascript -e 'tell application "Notes"
  make new folder with properties {name:"FOLDER_NAME"}
end tell'
\```

Replace `FOLDER_NAME` with the user's specified name. Confirm the folder was created.

## Move Note

### Find the Note

\```bash
osascript -e 'tell application "Notes"
  set matchingNotes to every note whose name contains "SEARCH_TERM"
  set output to ""
  repeat with n in matchingNotes
    set output to output & name of n & " | " & modification date of n & linefeed
  end repeat
  return output
end tell'
\```

If multiple notes match, show the list and ask the user which one to move.

### Move to Folder

Ensure the target folder exists (create it if not — see Default Folder section), then move:

\```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "NOTE_TITLE"
  move theNote to folder "FOLDER_NAME"
end tell'
\```

Replace:
- `NOTE_TITLE` with the exact name of the note
- `FOLDER_NAME` with the target folder

Confirm the note was moved.
```

- [ ] **Step 2: Verify the sections were appended**

Run: `grep -c "List Folders\|Create Folder\|Move Note" notes-manager/SKILL.md`
Expected: 3 matches.

---

### Task 10: Add Errors Section and Update README

**Files:**
- Modify: `notes-manager/SKILL.md`
- Modify: `README.md`

- [ ] **Step 1: Add errors section to SKILL.md**

Append this final section to `notes-manager/SKILL.md`:

```markdown
## Errors

| Error | What to do |
|-------|------------|
| Notes app not running | AppleScript will auto-launch Notes.app — no action needed |
| Folder not found | Create the folder first using the Create Folder steps |
| Note not found | The search term didn't match any notes — ask the user to refine their query |
| Multiple matches | Show all matches and let the user pick the right one |
| pandoc not found | Install via `brew install pandoc` |
| Permission denied | The user needs to grant Terminal/iTerm access to Notes in System Settings > Privacy & Security > Automation |
```

- [ ] **Step 2: Add notes-manager to the README skills table**

In `README.md`, add a new row to the skills table after the video-downloader row:

```markdown
| | [**notes-manager**](notes-manager/) | Say "create a note", "search notes", or "list my notes". Claude manages your macOS Notes app — create, read, search, append, delete, and organize notes and folders. |
```

- [ ] **Step 3: Verify both files are complete**

Run: `wc -l notes-manager/SKILL.md` — should be roughly 200-250 lines.
Run: `grep "notes-manager" README.md` — should show the new table row.

---

### Task 11: Manual QA Test

**Files:** None (verification only)

- [ ] **Step 1: Test create note**

Run the dependency check, then create a test note with markdown content:
```bash
command -v osascript && command -v pandoc
```

Then create a note:
```bash
TMPFILE=$(mktemp /tmp/claude-note-XXXXX.md)
cat << 'MDEOF' > "$TMPFILE"
## Test Note

This is a **test** note created by the notes-manager skill.

- Item 1
- Item 2
MDEOF
HTML=$(pandoc "$TMPFILE" -f markdown -t html)
rm "$TMPFILE"
osascript -e "tell application \"Notes\"
  try
    get folder \"Claude Notes\"
  on error
    make new folder with properties {name:\"Claude Notes\"}
  end try
  tell folder \"Claude Notes\"
    make new note with properties {body:\"<h1>Skill QA Test</h1>\" & \"$HTML\"}
  end tell
end tell"
```

Expected: Note "Skill QA Test" created in "Claude Notes" folder.

- [ ] **Step 2: Test list and search**

```bash
osascript -e 'tell application "Notes"
  set folderNotes to every note of folder "Claude Notes"
  set output to ""
  repeat with n in folderNotes
    set output to output & name of n & " | " & modification date of n & linefeed
  end repeat
  return output
end tell'
```

Expected: Shows "Skill QA Test" with its modification date.

- [ ] **Step 3: Test read note**

```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "Skill QA Test"
  return plaintext of theNote
end tell'
```

Expected: Shows the note content as plain text including "Test Note", "test", "Item 1", "Item 2".

- [ ] **Step 4: Test append**

```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "Skill QA Test"
  set body of theNote to (body of theNote) & "<p>Appended paragraph</p>"
end tell'
```

Then read again to verify the appended text appears.

- [ ] **Step 5: Test delete (cleanup)**

```bash
osascript -e 'tell application "Notes"
  delete (first note of folder "Claude Notes" whose name is "Skill QA Test")
end tell'
```

Expected: Note deleted. Verify by listing Claude Notes folder again.

- [ ] **Step 6: Clean up test folder if empty**

```bash
osascript -e 'tell application "Notes"
  if (count of notes of folder "Claude Notes") is 0 then
    delete folder "Claude Notes"
  end if
end tell'
```
