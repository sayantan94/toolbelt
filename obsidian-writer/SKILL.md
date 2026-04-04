---
name: obsidian-writer
description: Manages an Obsidian vault — creates, reads, searches, lists, appends to, deletes, and organizes markdown notes and folders. Use when working with Obsidian or when the user mentions "create a note", "save this to obsidian", "write a note", "find my note about", "search notes", "list my notes", "append to note", "delete note", "move note to folder", "create a folder", or "obsidian".
---

# Obsidian Writer

Manages an Obsidian vault — create, read, search, list, append, delete, and organize markdown notes and folders using plain file I/O.

## Step 1: Load Config

Check if the config file exists and contains the vault path:

```bash
cat ~/.config/obsidian-skill/config 2>/dev/null
```

The config file should contain a single line: `OBSIDIAN_VAULT_PATH=/path/to/vault`

**If the config file is missing or does not contain `OBSIDIAN_VAULT_PATH`:**

1. Ask the user: "Where is your Obsidian vault? Please provide the full path (e.g. ~/Documents/MyVault)."
2. Expand `~` to `$HOME` in the path.
3. Write the config:

```bash
mkdir -p ~/.config/obsidian-skill
# Expand ~ to $HOME in the user-provided path
OBSIDIAN_VAULT_PATH="${USER_PATH/#\~/$HOME}"
echo "OBSIDIAN_VAULT_PATH=\"$OBSIDIAN_VAULT_PATH\"" > ~/.config/obsidian-skill/config
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
source ~/.config/obsidian-skill/config
if [ -z "$OBSIDIAN_VAULT_PATH" ] || [ ! -d "$OBSIDIAN_VAULT_PATH" ]; then
  # run first-use flow (ask user for path)
fi
VAULT="$OBSIDIAN_VAULT_PATH"
```

Do NOT proceed with any operation until `$VAULT` is set and the directory exists.

## Step 2: Determine the Operation

Based on what the user said, pick the matching operation:

| User trigger | Operation |
|---|---|
| "create a note", "save this to obsidian", "write a note" | Create Note |
| "read note X", "show me the note about X" | Read Note |
| "search notes for X", "find my note about X" | Search Notes |
| "list my notes", "show notes in folder X" | List Notes |
| "append to note X", "add this to my note" | Append to Note |
| "delete note X" | Delete Note |
| "list folders", "show obsidian folders" | List Folders |
| "create folder X" | Create Folder |
| "move note X to folder Y" | Move Note |

Follow the section below that matches the chosen operation.

## Create Note

**Title resolution** — determine the note title in this order:
1. Explicit title given by the user (e.g. "create a note called 'Meeting Notes'")
2. Inferred from the first heading in the content (e.g. a `# Heading` line)
3. If neither, ask the user: "What should I title this note?"

**Slugify the title to a filename** — convert to lowercase, replace spaces and special characters with hyphens, remove consecutive hyphens, and append `.md`:

```bash
FILENAME=$(echo "NOTE TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g; s/--*/-/g; s/^-//; s/-$//')".md"
```

**Determine the target folder.** Default is the vault root unless the user specifies a subfolder. If a subfolder is specified, ensure it exists:

```bash
TARGET_DIR="$VAULT"
# Or if user specified a folder:
TARGET_DIR="$VAULT/FOLDER_NAME"
mkdir -p "$TARGET_DIR"
```

**Write the note with YAML frontmatter:**

```bash
cat << 'EOF' > "$TARGET_DIR/$FILENAME"
---
created: YYYY-MM-DD
tags: []
---

# Note Title

CONTENT_HERE
EOF
```

Replace `YYYY-MM-DD` with today's date, populate the `tags` array if the user specified tags (e.g. `[meeting, project-x]`), replace `Note Title` with the actual title, and replace `CONTENT_HERE` with the note content in markdown.

**Confirm:** Tell the user the note was created at `$TARGET_DIR/$FILENAME`.

## Read Note

**Find the note** — search by filename first:

```bash
find "$VAULT" -name "*SEARCH_TERM*" -name "*.md" -not -path "*/.obsidian/*"
```

If no filename match, search by content:

```bash
grep -rl "SEARCH_TERM" "$VAULT" --include="*.md" | grep -v "/.obsidian/"
```

If exactly one match, read and display it using the Read tool.

If multiple matches, list them and ask the user which one they want to read.

If no matches, tell the user no note was found and offer to search by a different term.

## Search Notes

**Search by title first:**

```bash
find "$VAULT" -name "*SEARCH_TERM*" -name "*.md" -not -path "*/.obsidian/*" | head -20
```

**If no title matches, search by content:**

```bash
grep -rl "SEARCH_TERM" "$VAULT" --include="*.md" | grep -v "/.obsidian/" | head -20
```

Replace `SEARCH_TERM` with the user's query.

**Display results** — for each match, show the relative path and last modified date:

```bash
for f in $MATCHES; do
  REL_PATH="${f#$VAULT/}"
  MOD_DATE=$(stat -f "%Sm" -t "%Y-%m-%d %H:%M" "$f" 2>/dev/null || stat -c "%y" "$f" 2>/dev/null | cut -d. -f1)
  echo "$REL_PATH | modified: $MOD_DATE"
done
```

Cap results at 20. If more exist, tell the user there are additional matches and suggest refining the search.

## List Notes

**List notes in a folder (default: vault root), sorted by modification time, capped at 20:**

```bash
ls -lt "$TARGET_DIR"/*.md 2>/dev/null | head -20 | while read -r line; do
  FILE=$(echo "$line" | awk '{print $NF}')
  BASENAME=$(basename "$FILE")
  MOD_DATE=$(stat -f "%Sm" -t "%Y-%m-%d %H:%M" "$FILE" 2>/dev/null || stat -c "%y" "$FILE" 2>/dev/null | cut -d. -f1)
  echo "$BASENAME | modified: $MOD_DATE"
done
```

Where `TARGET_DIR` is `$VAULT` for the root or `$VAULT/FOLDER_NAME` for a specific folder.

To list all notes recursively across the vault:

```bash
find "$VAULT" -name "*.md" -not -path "*/.obsidian/*" | while read -r path; do
  stat -f "%m %N" "$path"
done | sort -rn | head -20 | while read -r ts path; do
  REL_PATH="${path#$VAULT/}"
  MOD_DATE=$(stat -f "%Sm" -t "%Y-%m-%d %H:%M" "$path" 2>/dev/null)
  echo "$REL_PATH | modified: $MOD_DATE"
done
```

Show filename and modification date for each note. Default limit is 20 notes. If the user asks for more, increase the limit.

## Append to Note

**Find the note** — use the Search Notes approach (filename first, then content). If multiple notes match, list them and ask the user to confirm which one before appending.

**Append markdown content to the end of the file:**

```bash
cat << 'EOF' >> "$VAULT/PATH_TO_NOTE"

CONTENT_TO_APPEND
EOF
```

The leading blank line ensures separation from existing content.

**Confirm:** Tell the user what was appended and to which note (showing the relative path).

## Delete Note

**Find the note** — use the Search Notes approach. Show the user the match before proceeding.

**Always confirm before deleting.** Ask:

> "Are you sure you want to delete 'NOTE_FILENAME'? This cannot be undone."

Only proceed if the user confirms. Never delete multiple notes in bulk — one at a time only.

**Delete the note:**

```bash
rm "$VAULT/PATH_TO_NOTE"
```

**Confirm:** Tell the user the note has been deleted.

## List Folders

List all folders in the vault, excluding `.obsidian` internals:

```bash
find "$VAULT" -type d -not -path "*/.obsidian/*" -not -path "*/.obsidian" | while read -r dir; do
  REL="${dir#$VAULT}"
  [ -z "$REL" ] && REL="/"
  echo "$REL"
done
```

Display the folder names (relative to vault root) to the user.

## Create Folder

```bash
mkdir -p "$VAULT/FOLDER_NAME"
```

Replace `FOLDER_NAME` with the name the user provided. Supports nested paths (e.g. `projects/work`).

**Confirm:** Tell the user the folder "FOLDER_NAME" was created.

## Move Note

**Find the note** — use the Search Notes approach. Confirm the match with the user if there are multiple results.

**Verify the target folder exists.** If it doesn't, create it:

```bash
mkdir -p "$VAULT/TARGET_FOLDER"
```

**Move the note:**

```bash
mv "$VAULT/CURRENT_PATH" "$VAULT/TARGET_FOLDER/$(basename "$VAULT/CURRENT_PATH")"
```

Replace `CURRENT_PATH` with the note's current relative path and `TARGET_FOLDER` with the destination folder.

**Confirm:** Tell the user that the note has been moved to the target folder.

## Obsidian Features

When creating or editing notes, use these Obsidian-native features where appropriate:

### YAML Frontmatter

Every note should start with YAML frontmatter between `---` delimiters:

```yaml
---
created: 2025-01-15
tags: [meeting, project-x]
aliases: [alt-name]
---
```

Common frontmatter fields: `created`, `tags`, `aliases`, `cssclass`, `publish`.

### Tags

Tags use the `#tag-name` syntax in note body text or the `tags` array in frontmatter. Tags support nesting with `/`:

- `#project` — simple tag
- `#project/work` — nested tag
- `#project/personal` — another nested tag

When the user mentions tags, add them both to the frontmatter `tags` array and inline in the content where relevant.

### Wikilinks

Obsidian uses `[[note-name]]` to link between notes. When the user references another note, use wikilinks:

- `[[meeting-notes]]` — link to a note
- `[[meeting-notes|Meeting Notes]]` — link with display text
- `[[meeting-notes#section]]` — link to a specific section

When creating notes that reference other notes, use wikilinks to connect them.

## Errors

| Error | What to do |
|---|---|
| Config file missing | Run the first-use flow from Step 1 — ask for vault path, write config. |
| Vault path doesn't exist | Warn the user the configured path no longer exists. Offer to reconfigure by re-running Step 1. |
| Note not found | Tell the user no note with that name was found. Offer to search by partial name or content. |
| Multiple matches | List all matching notes with their paths, ask the user to pick one. |
| File permission denied | Tell the user to check file permissions on the vault directory (e.g. `ls -la "$VAULT"`). |
