# mac-notes Skill Design

A Claude Code skill that interacts with macOS Notes.app via AppleScript and pandoc. Provides full CRUD operations, search, and folder organization for notes — all from the terminal.

## Dependencies

- `osascript` — built-in on macOS, used for all Notes.app interactions
- `pandoc` — markdown-to-HTML conversion (`brew install pandoc`)

The skill checks for both at invocation and shows install instructions if `pandoc` is missing.

## Skill Metadata

- **Location:** `notes-manager/SKILL.md` in the toolbelt repo
- **Trigger phrases:** "create a note", "save this to notes", "jot this down", "find my note about X", "search notes for X", "list my notes", "show notes in folder X", "add this to my note about X", "append to note X", "delete the note called X", "move note X to folder Y", "create a folder called X"

## Operations

### Create Note

1. User provides content (text, markdown, or asks Claude to draft it)
2. If content contains markdown formatting, convert to HTML via pandoc (see Markdown Pipeline below)
3. If plain text, wrap in `<p>` tags directly
4. Create the note in the target folder via AppleScript (`make new note with properties {name:title, body:html}`)
5. Default folder: "Claude Notes" (auto-created if it doesn't exist)
6. User can override: "save this note in Work Ideas" uses/creates "Work Ideas" folder
7. Confirm: show note title and folder

### Note Title Resolution

- Explicitly provided by the user ("create a note called Meeting Notes")
- Inferred from the first heading or first line of the content
- Asked for if neither is clear

### Read / Search Notes

1. Search by title keyword or body content via AppleScript `whose name contains` / `whose body contains`
2. Search spans all folders unless user specifies one
3. If multiple matches, list titles and let the user pick
4. Display the selected note's body as plain text in the terminal

### List Notes

1. List all notes, or notes in a specific folder
2. Show: title + modification date
3. Sort by most recent first
4. Default limit: 20 notes (to avoid flooding the terminal)

### Append to Note

1. Find the target note by title (search with `whose name contains`)
2. If multiple matches, show them and let the user pick
3. Convert new content from markdown to HTML via pandoc
4. Append the HTML to the existing note body
5. Confirm what was appended and to which note

### Delete Note

1. Find by title
2. If multiple matches, show them and let the user pick
3. Show the note title and ask "Are you sure?" before deleting
4. No bulk delete supported — single note at a time only

### Organize (Folders)

1. **List folders:** Show all folders in the default account
2. **Create folder:** Create a new folder by name
3. **Move note:** Find a note by title, move it to a specified folder (create folder if it doesn't exist)

## Markdown-to-HTML Pipeline

Used when creating or appending notes with markdown content:

1. Write markdown content to a temp file via `mktemp`: `TMPFILE=$(mktemp /tmp/claude-note-XXXXX.md)`
2. Convert: `pandoc "$TMPFILE" -f markdown -t html`
3. Capture HTML output and pass into AppleScript as the note body
4. Clean up: `rm "$TMPFILE"`

For plain text (no markdown formatting), skip pandoc and wrap in `<p>` tags directly.

## Default Folder Behavior

- New notes default to a folder called **"Claude Notes"**
- If "Claude Notes" doesn't exist, the skill creates it automatically on first use
- User can override per-command: "save this note in Work Ideas"
- Read, search, list, and delete operate across **all folders** by default unless user specifies one

## Safety & Confirmation

| Operation | Confirmation required? |
|-----------|----------------------|
| Create    | No — just confirm what was created |
| Read      | No |
| Search    | No — but disambiguate if multiple matches |
| List      | No |
| Append    | Yes — show matched note before appending |
| Delete    | Yes — show note title and ask "Are you sure?" |
| Move      | No — confirm after move |

- No "delete all notes" or "delete folder" operations
- Ambiguous title matches always presented to user for selection

## Approach

All Notes.app interactions use AppleScript executed via `osascript -e '...'`. This is the most reliable and well-documented method for programmatic Notes.app access on macOS. JXA and direct API access were considered but rejected in favor of AppleScript's broader documentation and community support.

Pandoc handles markdown-to-HTML conversion because it correctly handles edge cases that hand-rolled solutions miss. The trade-off is an additional dependency, but `pandoc` is lightweight and commonly installed via Homebrew.

## File Structure

```
notes-manager/
  SKILL.md       # Complete skill definition with all AppleScript commands
```

Single file, matching the existing video-downloader pattern.
