# KBTD - Kanban TODO

A minimal Kanban board web app that uses a Markdown file as its data store.

## Overview

KBTD presents a Kanban board interface for managing tasks stored in a standard Markdown TODO file. The Markdown file is the canonical data store—the app provides a visual interface for viewing and manipulating it.

## Core Concepts

| Term | Description |
|------|-------------|
| **Project** | A working directory containing a TODO.md file (identified by absolute file path) |
| **Section** | An H1 (`# Name`) in the TODO file — rendered as a selectable Kanban board |
| **Column** | An H2 (`## Name`) under a section — a column within that board |
| **Card** | A `- [ ]` or `- [x]` item — draggable between columns |

## Markdown Format

```markdown
# Project Alpha

## Backlog

- [ ] Research competitors
- [ ] Write RFC

## In Progress

- [x] Design mockups
- [ ] Build prototype

## Done

- [x] Initial brainstorm

# Project Beta

## TODO

- [ ] Something else
```

### Parsing Rules

- H1 headers (`#`) define sections (boards)
- H2 headers (`##`) define columns within the preceding section
- `- [ ]` is an incomplete task
- `- [x]` is a complete task
- Task text is everything after the `] ` until end of line
- Content before the first H1 is ignored
- Content between H1 and first H2 is ignored
- Nested content under tasks (indented lines) is preserved but not displayed

## Architecture

### Components

1. **Shell script (`kbtd`)** - Cross-platform launcher
2. **Single HTML file (`index.html`)** - The entire web application

### Data Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  TODO.md    │◄───►│  Web App    │◄───►│ localStorage│
│  (file)     │     │  (browser)  │     │ (UI state)  │
└─────────────┘     └─────────────┘     └─────────────┘
      │                    │
      │   File System      │   IndexedDB
      │   Access API       │   (FileHandle)
      │                    │
      └────────────────────┘
```

### State Management

- **File state**: The TODO.md file is the source of truth for task data
- **UI state**: localStorage stores current section selection, keyed by file path
- **File handle**: IndexedDB stores the FileSystemFileHandle for persistence across sessions

#### localStorage Schema

Key: `kbtd:${filePath}`

```json
{
  "currentSection": "Project Alpha"
}
```

#### IndexedDB Schema

Database: `kbtd`
Object Store: `fileHandles`
Key: file path from URL query string
Value: FileSystemFileHandle

## User Interface

### States

1. **No file associated**: Shows file picker button and instructions
2. **Permission needed**: Shows button to re-grant permission (after page reload)
3. **File not found**: Shows error message if TODO.md doesn't exist
4. **Section picker**: Shows list of sections (H1s) to choose from
5. **Kanban board**: Shows columns and cards for selected section

### Kanban Board Features

- Columns displayed horizontally
- Cards displayed vertically within columns
- Drag and drop cards between columns
- Click card to toggle complete/incomplete
- Visual distinction for completed tasks (strikethrough or muted)

### Navigation

- Back button to return to section picker
- Current section name displayed in header

## File Synchronization

### Polling

- Poll file every 250ms using `file.lastModified`
- On change detected: reload file, rebuild state, re-render UI
- If currently selected section no longer exists: return to section picker

### Write Strategy

- On any card move or status toggle: serialize full state to Markdown and write
- Use `FileSystemFileHandle.createWritable()` for atomic writes

### Conflict Resolution

None. Last write wins. External changes blow away app state.

## Shell Script (`kbtd`)

### Usage

```bash
kbtd [OPTIONS] <path-to-markdown-file>

Options:
  --browser    Open in existing browser instead of app mode
  --help       Show help message
```

### Behavior

1. Resolve the markdown file path to absolute path
2. Locate a Chromium-based browser (Chrome, Chromium, Edge)
3. Determine mode:
   - Default: `--app` mode with isolated `--user-data-dir` in current directory
   - With `--browser`: open in existing browser instance with `--new-window`
4. Launch browser with URL: `{base_url}?file={absolute_path}`

### Browser Detection

Check in order:
1. `$BROWSER` environment variable (if Chromium-based)
2. `google-chrome` / `chromium` / `chromium-browser` / `microsoft-edge` in PATH
3. macOS: `/Applications/Google Chrome.app/...`, `/Applications/Chromium.app/...`, `/Applications/Microsoft Edge.app/...`
4. WSL: `/mnt/c/Program Files/Google/Chrome/Application/chrome.exe`, etc.

If no Chromium browser found: exit with error message explaining requirement.

### App Mode Launch

```bash
# Create isolated user data directory
USER_DATA_DIR="./.kbtd-chrome-data"

# Launch in app mode
google-chrome \
  --user-data-dir="$USER_DATA_DIR" \
  --app="http://localhost:8000/?file=/absolute/path/to/TODO.md"
```

### Browser Mode Launch

```bash
google-chrome --new-window "http://localhost:8000/?file=/absolute/path/to/TODO.md"
```

### URL Configuration

- Development: `http://localhost:8000`
- Production: Configurable via `$KBTD_URL` environment variable or hardcoded default

## Web App Behavior

### Startup Flow

```
1. Parse ?file= from URL query string
2. If no file param:
   → Show landing page with instructions and script download link
3. Check IndexedDB for stored FileHandle for this file path
4. If handle exists:
   → Check permission with queryPermission()
   → If "granted": proceed to load file
   → If "prompt": show "Grant Access" button
   → If "denied": show file picker
5. If no handle:
   → Show file picker button
6. On file picker success:
   → Store handle in IndexedDB
   → Load and parse file
   → Show section picker or board
```

### File Picker

```javascript
const [fileHandle] = await window.showOpenFilePicker({
  id: 'kbtd-todo-file',
  mode: 'readwrite',
  types: [{
    description: 'Markdown files',
    accept: { 'text/markdown': ['.md'] }
  }]
});
```

### Permission Re-grant

After page reload, stored handles return `"prompt"` for permission state. User must click a button to trigger:

```javascript
const permission = await fileHandle.requestPermission({ mode: 'readwrite' });
```

This requires a user gesture (button click).

## Browser Compatibility

**Required**: Chromium-based browser (Chrome 86+, Edge 86+, Opera 72+)

**Not supported**: Firefox, Safari (File System Access API not implemented)

The shell script should detect and warn if no compatible browser is found.

## File Structure

```
kbtd/
├── index.html      # Complete web application (single file)
├── kbtd            # Shell script launcher
└── SPEC.md         # This specification
```

## Landing Page

When loaded without a `?file=` parameter, the app displays:

1. App name and brief description
2. Instructions for use
3. Download link/instructions for the shell script
4. Note about Chromium browser requirement

## Architecture Principles

### Source of Truth

The Markdown file is the **only** source of truth. The Kanban UI is purely a renderer and editor for the file. There is no separate application state that needs to be "synced" with the file.

### Functional & Idempotent Design

- **Pure parsing**: `parseMarkdown(text) → {sections, columns, cards}` — same input always produces same output
- **Pure serialization**: `serializeToMarkdown(data) → text` — same input always produces same output
- **Idempotent renders**: `render(data)` can be called any number of times with the same data and produce the same DOM
- **No hidden state**: UI state (current section) is minimal and stored separately from task data
- **Stateless transformations**: operations like `moveCard(data, cardIndex, fromColumn, toColumn) → newData` return new data structures rather than mutating

### Data Flow

```
File Change → Read File → Parse → Render
User Action → Transform Data → Serialize → Write File → Read File → Parse → Render
```

Note: After every write, the app re-reads and re-parses the file. This ensures the file remains the source of truth and the UI always reflects exactly what's on disk.

## Non-Goals (v1)

- No task metadata (due dates, tags, assignees)
- No task IDs or UUIDs
- No multi-file support
- No task details/descriptions (subtasks, notes)
- No offline support beyond what the browser provides
- No collaborative editing
- No mobile support
- No server component (pure client-side)

## Future Considerations

- Task detail files in `.issues/` subdirectory
- UUID-based task tracking for stable identity
- Folder access instead of single file
- Multiple TODO files in project
