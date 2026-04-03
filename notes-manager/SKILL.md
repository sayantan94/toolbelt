---
name: notes-manager
description: Manages macOS Notes app — creates, reads, searches, lists, appends to, deletes, and organizes notes and folders. Use when working with Apple Notes or when the user mentions "create a note", "save this to notes", "jot this down", "find my note about", "search notes", "list my notes", "append to note", "delete note", "move note to folder", or "create a folder".
---

# Notes Manager

Manages macOS Notes.app — create, read, search, list, append, delete, and organize notes and folders.

## Step 1: Check Dependencies

Run both checks. If either is missing, stop and show install commands — do NOT proceed without both.

```bash
command -v osascript && command -v pandoc
```

If `osascript` is missing: osascript is a built-in macOS tool. If missing, the system installation may be corrupted.

If `pandoc` is missing:
```bash
brew install pandoc
```

Do NOT proceed until both tools are confirmed present.

## Step 2: Determine the Operation

Based on what the user said, pick the matching operation:

| What the user said | Operation |
|--------------------|-----------|
| "create a note", "save this to notes", "jot this down" | Create Note |
| "find my note about X", "search notes for X" | Search Notes |
| "read note X", "show me the note about X" | Read Note |
| "list my notes", "show notes in folder X" | List Notes |
| "add this to my note about X", "append to note X" | Append to Note |
| "delete the note called X" | Delete Note |
| "list folders", "show my folders" | List Folders |
| "create a folder called X" | Create Folder |
| "move note X to folder Y" | Move Note |

Follow the section below that matches the chosen operation.

## Create Note

**Title resolution** — determine the note title in this order:
1. Explicit title given by the user (e.g. "create a note called 'Meeting Notes'")
2. Inferred from the first heading in the content (e.g. a `# Heading` in markdown)
3. If neither, ask the user: "What should I title this note?"

**Ensure the folder exists** (see the Default Folder section at the end of this document).

**Convert content to HTML** (see the Markdown-to-HTML Conversion section at the end of this document). Make sure to escape double quotes in the HTML before embedding it in the AppleScript string.

**Create the note:**

```bash
osascript -e 'tell application "Notes"
  tell folder "FOLDER_NAME"
    make new note with properties {body:"<h1>NOTE_TITLE</h1>" & "HTML_CONTENT"}
  end tell
end tell'
```

Replace `FOLDER_NAME`, `NOTE_TITLE`, and `HTML_CONTENT` with actual values. The note's title in Notes.app derives from the first `<h1>` tag in the body.

**Confirm:** Tell the user the note was created with title "NOTE_TITLE" in folder "FOLDER_NAME".

## Search Notes

First search by title. If no notes match by title, search by body content instead.

**Search by title:**

```bash
osascript -e 'tell application "Notes"
  set matchingNotes to every note whose name contains "SEARCH_TERM"
  set results to {}
  set resultCount to 0
  repeat with n in matchingNotes
    if resultCount is less than 20 then
      set end of results to (name of n & " | modified: " & (modification date of n as string))
      set resultCount to resultCount + 1
    end if
  end repeat
  return results
end tell'
```

**If no results from title search, search by body (plaintext):**

```bash
osascript -e 'tell application "Notes"
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
end tell'
```

Replace `SEARCH_TERM` with the user's search query.

Show the user the title and modification date for each match. If multiple notes match, list them all and ask the user which one they want to open or work with.

## Read Note

First, find the note using the Search Notes approach above. Use the **exact note title** returned by the search (not the user's original query) in the `name is` predicate below:

```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "NOTE_TITLE"
  return plaintext of theNote
end tell'
```

Use `plaintext` (not `body`) — plaintext strips HTML tags and is readable in the terminal.

Display the note's content to the user.

## List Notes

**List all notes (capped at 20, in Notes.app default order):**

```bash
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
```

**List notes in a specific folder (capped at 20, in Notes.app default order):**

```bash
osascript -e 'tell application "Notes"
  tell folder "FOLDER_NAME"
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
  end tell
end tell'
```

Notes.app returns notes in its default order. Show title and modification date for each note. Default limit is 20 notes. If the user asks for more, show all.

## Append to Note

**Find the note by title** — use the Search Notes approach to locate it. If multiple notes match, list them and ask the user to confirm which one before appending.

**Convert new content to HTML** (see the Markdown-to-HTML Conversion section at the end of this document). Make sure to escape double quotes in the HTML before embedding it in the AppleScript string.

**Append to the note:**

```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "NOTE_TITLE"
  set body of theNote to (body of theNote) & "HTML_CONTENT"
end tell'
```

Replace `NOTE_TITLE` and `HTML_CONTENT` with actual values.

**Confirm:** Tell the user what was appended and to which note.

## Delete Note

**Find the note by title** — use the Search Notes approach. Show the user the match before proceeding.

**Always confirm before deleting.** Ask:

> "Are you sure you want to delete 'NOTE_TITLE'? This cannot be undone."

Only proceed if the user confirms. Never delete multiple notes in bulk — one at a time only.

**Delete the note:**

```bash
osascript -e 'tell application "Notes"
  delete (first note whose name is "NOTE_TITLE")
end tell'
```

**Confirm:** Tell the user the note "NOTE_TITLE" has been deleted.

## List Folders

```bash
osascript -e 'tell application "Notes"
  return name of every folder
end tell'
```

Display the folder names to the user.

## Create Folder

```bash
osascript -e 'tell application "Notes"
  make new folder with properties {name:"FOLDER_NAME"}
end tell'
```

Replace `FOLDER_NAME` with the name the user provided.

**Confirm:** Tell the user the folder "FOLDER_NAME" was created.

## Move Note

**Find the note** — use the Search Notes approach to locate the note by title. Confirm the match with the user if there are multiple results.

**Ensure the target folder exists** (see the Default Folder section at the end of this document — same auto-create logic applies).

**Move the note:**

```bash
osascript -e 'tell application "Notes"
  set theNote to first note whose name is "NOTE_TITLE"
  move theNote to folder "FOLDER_NAME"
end tell'
```

Replace `NOTE_TITLE` and `FOLDER_NAME` with actual values.

**Confirm:** Tell the user that "NOTE_TITLE" has been moved to folder "FOLDER_NAME".

## Utilities

### Markdown-to-HTML Conversion

When the note content is in markdown (e.g. has headings, lists, bold, code blocks), convert it to HTML using pandoc before passing it to AppleScript:

```bash
TMPFILE=$(mktemp /tmp/claude-note-XXXXX.md)
cat << 'MDEOF' > "$TMPFILE"
<markdown content here>
MDEOF
HTML=$(pandoc "$TMPFILE" -f markdown -t html)
rm "$TMPFILE"
# Escape double quotes so they don't break the AppleScript string
HTML=$(echo "$HTML" | sed 's/"/\\"/g')
```

For plain text content (no formatting), skip pandoc and wrap the text in `<p>` tags directly:
```bash
HTML="<p>PLAIN_TEXT_CONTENT</p>"
```

The resulting `$HTML` variable is used as `HTML_CONTENT` in the AppleScript commands above.

### Default Folder

The default folder for all notes is **"Claude Notes"**. Always use this folder unless the user specifies a different one.

Before creating any note, ensure the folder exists. Auto-create it if missing:

```bash
osascript -e 'tell application "Notes"
  try
    get folder "FOLDER_NAME"
  on error
    make new folder with properties {name:"FOLDER_NAME"}
  end try
end tell'
```

Replace `FOLDER_NAME` with the actual folder name (default: `Claude Notes`).

## Errors

| Error | What to do |
|-------|------------|
| Note not found | Tell the user no note with that title was found. Offer to search by partial name or body text. |
| Folder not found | Auto-create the folder (see the Default Folder section at the end of this document) and retry. |
| AppleScript permission denied | Notes.app needs Automation permission. Tell the user: System Settings → Privacy & Security → Automation → enable Terminal (or Claude Code) to control Notes. |
| pandoc conversion fails | Fall back to wrapping content in `<p>` tags as plain text. Warn the user that formatting may be lost. |
