# Terminal Text Editor Component

## Purpose

Provide a multi-line text editor component for terminal UIs that supports
cursor movement, selection, undo/redo, kill ring (emacs-style clipboard),
autocomplete, configurable keybindings, word wrapping, and paste detection.
The editor maintains a text buffer as an array of lines and renders with
word wrapping to fit any viewport width, emitting a cursor marker for IME
support.

## Dependencies

- **terminal-ui-differential-rendering.skill.md** -- the editor implements
  the `Component` interface and uses `CURSOR_MARKER` for cursor positioning.

## Core Concepts

### Text buffer

The editor's state is a list of strings (one per logical line) plus a cursor
position (row, column).  All editing operations mutate this buffer and cursor.

### Undo / redo

Every mutation saves an undo point containing the full buffer and cursor state.
Rapid edits within a coalescing threshold (e.g. 500ms) are merged into a single
undo point so that undo steps correspond to logical user actions rather than
individual keystrokes.

### Kill ring

An emacs-style circular buffer of deleted text.  `kill` pushes text;
`yank` inserts the most recent entry; `yank-pop` cycles through older entries.

### Autocomplete

A provider-based system where the editor requests completions for the current
word prefix.  Results are fuzzy-matched and ranked.  The autocomplete dropdown
renders as an overlay or inline component managed by the editor.

### Keybindings

Keys are represented as pattern strings like `"ctrl+a"`, `"shift+tab"`,
`"alt+enter"`.  A binding maps a key pattern to an action function.  Default
bindings cover emacs-style, standard, and navigation keys.  Bindings are
configurable and overridable.

### Bracketed paste

Terminals supporting bracketed paste wrap pasted text in `ESC[200~` ...
`ESC[201~` markers.  The editor detects these and handles the entire paste as a
single operation (one undo point, no autocomplete triggers per character).

## Data Structures

```
// ---- Text buffer ----

record TextBuffer:
    lines  : list<string>        // one entry per logical line
    cursor : CursorPosition

record CursorPosition:
    row : integer                // 0-based line index
    col : integer                // 0-based character offset within line

// ---- Undo / Redo ----

record UndoEntry:
    lines     : list<string>     // snapshot of buffer
    cursorRow : integer
    cursorCol : integer
    timestamp : integer          // epoch millis, for coalescing

class UndoStack:
    entries      : list<UndoEntry>
    currentIndex : integer       // points to last applied entry
    maxSize      : integer       // cap to prevent unbounded memory

    canUndo() -> bool
    canRedo() -> bool
    push(entry: UndoEntry)
    undo() -> UndoEntry
    redo() -> UndoEntry
    truncateAfterCurrent()       // discard redo history on new edit

// ---- Kill ring ----

class KillRing:
    entries      : list<string>
    currentIndex : integer       // circular index
    maxSize      : integer       // e.g. 30

    push(text: string)
    current() -> string
    advance()                    // move to next entry (for yank-pop)

// ---- Autocomplete ----

record AutocompleteItem:
    label       : string         // display text
    detail      : string?        // secondary info (e.g. type)
    description : string?        // longer description
    insertText  : string?        // text to insert (defaults to label)
    icon        : string?        // icon character

record AutocompleteState:
    items         : list<AutocompleteItem>
    selectedIndex : integer
    prefix        : string       // word being completed
    active        : bool

interface AutocompleteProvider:
    getCompletions(prefix: string, context: any) -> list<AutocompleteItem>

// ---- Keybinding ----

record KeyBinding:
    key         : string         // pattern: "ctrl+a", "shift+tab", "alt+enter"
    action      : function()     // handler
    description : string         // for help display

// ---- Editor component ----

interface EditorComponent extends Component:
    getText() -> string
    setText(text: string)
    handleInput(data: string)
    onSubmit    : function(text)?
    onChange    : function(text)?
    addToHistory(text)?
    insertTextAtCursor(text)?
    getExpandedText() -> string?
    setAutocompleteProvider(provider)?
    borderColor : function(str) -> str?
    setPaddingX(padding)?
    setAutocompleteMaxVisible(maxVisible)?
```

## Algorithm / Flow

### 1. Input handling (main dispatch)

```
function handleInput(editor, data):
    // 1. Bracketed paste detection
    if data starts with PASTE_START ("\x1b[200~"):
        content = extractBetween(data, PASTE_START, PASTE_END)
        handlePaste(editor, content)
        return

    // 2. If autocomplete is active, handle autocomplete keys first
    if editor.autocomplete.active:
        handled = handleAutocompleteInput(editor, data)
        if handled: return

    // 3. Match keybinding
    binding = findKeybinding(editor.keybindings, data)
    if binding is not null:
        binding.action()
        notifyChange(editor)
        return

    // 4. Insert printable character
    if isPrintable(data):
        saveUndoPoint(editor)
        insertAtCursor(editor, data)
        triggerAutocomplete(editor)
        notifyChange(editor)
        return

    // 5. Multi-byte character (e.g. emoji, CJK)
    if isMultiByteChar(data):
        saveUndoPoint(editor)
        insertAtCursor(editor, data)
        triggerAutocomplete(editor)
        notifyChange(editor)
```

### 2. Autocomplete input handling

```
function handleAutocompleteInput(editor, data) -> bool:
    ac = editor.autocomplete

    if data == TAB or data == DOWN_ARROW:
        ac.selectedIndex = (ac.selectedIndex + 1) % len(ac.items)
        return true

    if data == SHIFT_TAB or data == UP_ARROW:
        ac.selectedIndex = (ac.selectedIndex - 1 + len(ac.items)) % len(ac.items)
        return true

    if data == ENTER:
        acceptCompletion(editor)
        return true

    if data == ESCAPE:
        ac.active = false
        return true

    // For other keys, dismiss autocomplete and let normal handling proceed
    // (or update autocomplete after normal handling)
    return false
```

### 3. Autocomplete trigger and accept

```
function triggerAutocomplete(editor):
    prefix = getWordBeforeCursor(editor)
    if len(prefix) < editor.minTriggerLength:
        editor.autocomplete.active = false
        return

    provider = editor.autocompleteProvider
    if provider is null:
        return

    items = provider.getCompletions(prefix, getContext(editor))
    if len(items) == 0:
        editor.autocomplete.active = false
        return

    // Fuzzy match and rank
    scored = []
    for item in items:
        score = fuzzyMatch(prefix, item.label)
        if score > 0:
            scored.append({ item, score })

    sort(scored, by: score, descending)

    editor.autocomplete = AutocompleteState(
        items = scored.map(s => s.item),
        selectedIndex = 0,
        prefix = prefix,
        active = true
    )

function acceptCompletion(editor):
    ac = editor.autocomplete
    item = ac.items[ac.selectedIndex]
    text = item.insertText or item.label

    // Delete the prefix that triggered autocomplete
    deleteWordBeforeCursor(editor, len(ac.prefix))

    // Insert completion text
    saveUndoPoint(editor)
    insertAtCursor(editor, text)

    ac.active = false
    notifyChange(editor)
```

### 4. Undo / Redo

```
COALESCE_MS = 500

function saveUndoPoint(editor):
    stack = editor.undoStack
    now = currentTimeMillis()

    // Coalesce rapid changes
    if stack.entries is not empty:
        last = stack.entries[stack.currentIndex]
        if now - last.timestamp < COALESCE_MS:
            // Update the existing entry instead of creating a new one
            last.lines = copy(editor.buffer.lines)
            last.cursorRow = editor.buffer.cursor.row
            last.cursorCol = editor.buffer.cursor.col
            last.timestamp = now
            return

    // Truncate any redo history
    stack.truncateAfterCurrent()

    // Push new entry
    entry = UndoEntry(
        lines = copy(editor.buffer.lines),
        cursorRow = editor.buffer.cursor.row,
        cursorCol = editor.buffer.cursor.col,
        timestamp = now
    )
    stack.push(entry)

    // Enforce max size
    while len(stack.entries) > stack.maxSize:
        stack.entries.removeFirst()
        stack.currentIndex -= 1

function undo(editor):
    stack = editor.undoStack
    if not stack.canUndo(): return

    // Save current state as redo point if we're at the top
    if stack.currentIndex == len(stack.entries) - 1:
        stack.push(UndoEntry(
            lines = copy(editor.buffer.lines),
            cursorRow = editor.buffer.cursor.row,
            cursorCol = editor.buffer.cursor.col,
            timestamp = currentTimeMillis()
        ))
        stack.currentIndex -= 1   // point to the entry before the one we just pushed

    entry = stack.undo()
    editor.buffer.lines = copy(entry.lines)
    editor.buffer.cursor.row = entry.cursorRow
    editor.buffer.cursor.col = entry.cursorCol
    notifyChange(editor)

function redo(editor):
    stack = editor.undoStack
    if not stack.canRedo(): return

    entry = stack.redo()
    editor.buffer.lines = copy(entry.lines)
    editor.buffer.cursor.row = entry.cursorRow
    editor.buffer.cursor.col = entry.cursorCol
    notifyChange(editor)
```

### 5. Kill ring

```
function killLine(editor):
    // Kill from cursor to end of line (Ctrl+K)
    line = editor.buffer.lines[editor.buffer.cursor.row]
    col = editor.buffer.cursor.col
    killed = line[col:]
    editor.buffer.lines[editor.buffer.cursor.row] = line[0:col]

    if len(killed) == 0:
        // At end of line -- kill the newline (join with next line)
        if editor.buffer.cursor.row < len(editor.buffer.lines) - 1:
            nextLine = editor.buffer.lines[editor.buffer.cursor.row + 1]
            editor.buffer.lines[editor.buffer.cursor.row] += nextLine
            editor.buffer.lines.remove(editor.buffer.cursor.row + 1)
            killed = "\n"

    editor.killRing.push(killed)
    notifyChange(editor)

function killWord(editor):
    // Kill from cursor to end of word (Alt+D)
    { start, end } = findWordBoundaryForward(editor)
    killed = extractText(editor, start, end)
    deleteRange(editor, start, end)
    editor.killRing.push(killed)
    notifyChange(editor)

function killWordBackward(editor):
    // Kill from cursor backward to start of word (Alt+Backspace)
    { start, end } = findWordBoundaryBackward(editor)
    killed = extractText(editor, start, end)
    deleteRange(editor, start, end)
    editor.killRing.push(killed)
    notifyChange(editor)

function yank(editor):
    // Paste from kill ring (Ctrl+Y)
    text = editor.killRing.current()
    if text is null: return
    saveUndoPoint(editor)
    editor.lastYankStart = copy(editor.buffer.cursor)
    insertAtCursor(editor, text)
    editor.lastYankEnd = copy(editor.buffer.cursor)
    notifyChange(editor)

function yankPop(editor):
    // Cycle through kill ring (Alt+Y, only after yank)
    if editor.lastYankStart is null: return

    // Delete the last yanked text
    deleteRange(editor, editor.lastYankStart, editor.lastYankEnd)
    editor.buffer.cursor = copy(editor.lastYankStart)

    // Advance kill ring and yank next entry
    editor.killRing.advance()
    text = editor.killRing.current()
    if text is null: return

    editor.lastYankStart = copy(editor.buffer.cursor)
    insertAtCursor(editor, text)
    editor.lastYankEnd = copy(editor.buffer.cursor)
    notifyChange(editor)
```

### 6. Paste handling

```
PASTE_START = "\x1b[200~"
PASTE_END   = "\x1b[201~"

function handlePaste(editor, content):
    saveUndoPoint(editor)   // single undo point for entire paste
    insertAtCursor(editor, content)
    // Do NOT trigger autocomplete for pasted content
    editor.autocomplete.active = false
    notifyChange(editor)
```

### 7. Cursor movement

```
function moveCursorLeft(editor):
    if editor.buffer.cursor.col > 0:
        editor.buffer.cursor.col -= 1
    elif editor.buffer.cursor.row > 0:
        editor.buffer.cursor.row -= 1
        editor.buffer.cursor.col = len(editor.buffer.lines[editor.buffer.cursor.row])

function moveCursorRight(editor):
    line = editor.buffer.lines[editor.buffer.cursor.row]
    if editor.buffer.cursor.col < len(line):
        editor.buffer.cursor.col += 1
    elif editor.buffer.cursor.row < len(editor.buffer.lines) - 1:
        editor.buffer.cursor.row += 1
        editor.buffer.cursor.col = 0

function moveCursorUp(editor):
    if editor.buffer.cursor.row > 0:
        editor.buffer.cursor.row -= 1
        editor.buffer.cursor.col = min(editor.buffer.cursor.col,
                                        len(editor.buffer.lines[editor.buffer.cursor.row]))

function moveCursorDown(editor):
    if editor.buffer.cursor.row < len(editor.buffer.lines) - 1:
        editor.buffer.cursor.row += 1
        editor.buffer.cursor.col = min(editor.buffer.cursor.col,
                                        len(editor.buffer.lines[editor.buffer.cursor.row]))

function moveToLineStart(editor):      // Ctrl+A or Home
    editor.buffer.cursor.col = 0

function moveToLineEnd(editor):        // Ctrl+E or End
    editor.buffer.cursor.col = len(editor.buffer.lines[editor.buffer.cursor.row])

function moveWordForward(editor):      // Alt+F
    { row, col } = findWordBoundaryForward(editor)
    editor.buffer.cursor = { row, col }

function moveWordBackward(editor):     // Alt+B
    { row, col } = findWordBoundaryBackward(editor)
    editor.buffer.cursor = { row, col }
```

### 8. Text insertion and deletion

```
function insertAtCursor(editor, text):
    lines = text.split("\n")
    row = editor.buffer.cursor.row
    col = editor.buffer.cursor.col
    currentLine = editor.buffer.lines[row]

    if len(lines) == 1:
        // Single line insert
        editor.buffer.lines[row] = currentLine[0:col] + lines[0] + currentLine[col:]
        editor.buffer.cursor.col = col + len(lines[0])
    else:
        // Multi-line insert
        before = currentLine[0:col]
        after  = currentLine[col:]
        editor.buffer.lines[row] = before + lines[0]
        // Insert middle lines
        for i in range(1, len(lines) - 1):
            editor.buffer.lines.insert(row + i, lines[i])
        // Insert last line with remainder
        editor.buffer.lines.insert(row + len(lines) - 1, lines[len(lines) - 1] + after)
        editor.buffer.cursor.row = row + len(lines) - 1
        editor.buffer.cursor.col = len(lines[len(lines) - 1])

function deleteCharBefore(editor):     // Backspace
    if editor.buffer.cursor.col > 0:
        line = editor.buffer.lines[editor.buffer.cursor.row]
        editor.buffer.lines[editor.buffer.cursor.row] = line[0:col-1] + line[col:]
        editor.buffer.cursor.col -= 1
    elif editor.buffer.cursor.row > 0:
        // Join with previous line
        prevRow = editor.buffer.cursor.row - 1
        prevLine = editor.buffer.lines[prevRow]
        editor.buffer.lines[prevRow] = prevLine + editor.buffer.lines[editor.buffer.cursor.row]
        editor.buffer.lines.remove(editor.buffer.cursor.row)
        editor.buffer.cursor.row = prevRow
        editor.buffer.cursor.col = len(prevLine)

function deleteCharAfter(editor):      // Delete or Ctrl+D
    line = editor.buffer.lines[editor.buffer.cursor.row]
    col = editor.buffer.cursor.col
    if col < len(line):
        editor.buffer.lines[editor.buffer.cursor.row] = line[0:col] + line[col+1:]
    elif editor.buffer.cursor.row < len(editor.buffer.lines) - 1:
        // Join with next line
        nextLine = editor.buffer.lines[editor.buffer.cursor.row + 1]
        editor.buffer.lines[editor.buffer.cursor.row] = line + nextLine
        editor.buffer.lines.remove(editor.buffer.cursor.row + 1)
```

### 9. Rendering with word wrap

```
function render(editor, width):
    outputLines = []
    cursorOutputRow = -1
    cursorOutputCol = -1

    for i in range(len(editor.buffer.lines)):
        wrappedLines = wordWrap(editor.buffer.lines[i], width)
        if len(wrappedLines) == 0:
            wrappedLines = [""]    // empty line still takes a row

        if i == editor.buffer.cursor.row:
            // Find which wrapped line contains the cursor column
            colOffset = 0
            for j in range(len(wrappedLines)):
                lineLen = len(wrappedLines[j])
                if editor.buffer.cursor.col <= colOffset + lineLen or j == len(wrappedLines) - 1:
                    cursorOutputRow = len(outputLines) + j
                    cursorOutputCol = editor.buffer.cursor.col - colOffset
                    break
                colOffset += lineLen

        outputLines.extend(wrappedLines)

    // Insert cursor marker if focused
    if editor.focused and cursorOutputRow >= 0:
        line = outputLines[cursorOutputRow]
        // Insert CURSOR_MARKER at the visual cursor column
        charIndex = columnToCharIndex(line, cursorOutputCol)
        outputLines[cursorOutputRow] = line[0:charIndex] + CURSOR_MARKER + line[charIndex:]

    return outputLines
```

### 10. Default keybinding table

```
DEFAULT_KEYBINDINGS = [
    // Navigation
    { key: "left",          action: moveCursorLeft },
    { key: "right",         action: moveCursorRight },
    { key: "up",            action: moveCursorUp },
    { key: "down",          action: moveCursorDown },
    { key: "home",          action: moveToLineStart },
    { key: "end",           action: moveToLineEnd },
    { key: "ctrl+a",        action: moveToLineStart },
    { key: "ctrl+e",        action: moveToLineEnd },
    { key: "alt+f",         action: moveWordForward },
    { key: "alt+b",         action: moveWordBackward },

    // Editing
    { key: "backspace",     action: deleteCharBefore },
    { key: "delete",        action: deleteCharAfter },
    { key: "ctrl+d",        action: deleteCharAfter },
    { key: "enter",         action: insertNewlineOrSubmit },
    { key: "alt+enter",     action: insertNewline },

    // Kill / Yank
    { key: "ctrl+k",        action: killLine },
    { key: "alt+d",         action: killWord },
    { key: "alt+backspace", action: killWordBackward },
    { key: "ctrl+y",        action: yank },
    { key: "alt+y",         action: yankPop },

    // Undo / Redo
    { key: "ctrl+z",        action: undo },
    { key: "ctrl+shift+z",  action: redo },

    // Autocomplete
    { key: "tab",           action: triggerOrAcceptAutocomplete },
    { key: "escape",        action: dismissAutocomplete },

    // History
    { key: "ctrl+up",       action: historyPrevious },
    { key: "ctrl+down",     action: historyNext },
]
```

## Key Design Decisions

1. **Array-of-lines buffer.**  Representing the buffer as `list<string>` makes
   line-oriented operations (insert newline, delete line, word wrap) natural.
   The tradeoff is that extremely long lines are a single string, but this is
   acceptable for an input editor (not a code editor for large files).

2. **Coalesced undo.**  Merging edits within 500ms into one undo point means
   that typing "hello" creates one undo entry, not five.  This matches user
   expectations from GUI editors.

3. **Kill ring over clipboard.**  Terminal environments don't have reliable
   clipboard access.  The kill ring provides an internal multi-entry clipboard
   that works everywhere.  System clipboard integration can be layered on top
   via OSC 52 paste sequences.

4. **Fuzzy matching for autocomplete.**  Rather than prefix-only matching,
   fuzzy matching lets users type partial fragments (e.g. "fcr" matches
   "fileCreate").  Items are scored so the best match appears first.

5. **Bracketed paste as atomic operation.**  Treating paste as a single undo
   step and suppressing autocomplete during paste prevents the jarring behavior
   of autocomplete popups flashing during a large paste.

6. **Configurable keybindings.**  The default table uses emacs-style bindings
   (common in terminals), but users or extensions can override any binding.
   This enables vim-mode or custom shortcut schemes without modifying the
   editor core.

7. **CURSOR_MARKER in render output.**  Rather than maintaining a separate
   cursor position data structure, the cursor is embedded as a zero-width
   marker in the rendered output.  The parent TUI extracts it and positions
   the hardware cursor.  This decouples the editor from cursor management.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| Empty buffer (no text) | Buffer always has at least one line (empty string); cursor at (0,0) |
| Cursor past end of line after line deletion | Cursor column is clamped to the new line's length |
| Very long paste (megabytes) | Single undo point; render may be slow but functional; consider truncation for extremely large pastes |
| Wide characters (emoji, CJK) | `visibleWidth` accounts for double-width chars; cursor column is in character units, converted to visual columns for rendering |
| Undo stack overflow | Oldest entries are evicted when max size is reached |
| Autocomplete with zero results | Dropdown dismissed; no visual change |
| Autocomplete item's `insertText` contains newlines | Multi-line insertion handled by `insertAtCursor`; autocomplete dismisses after accept |
| Kill ring empty when yank is invoked | No-op; no text inserted |
| `yankPop` without preceding `yank` | No-op; `lastYankStart` is null |
| Word boundary at start/end of buffer | Movement functions clamp to buffer bounds |
| Tab character in input | Treated as printable (inserted as-is) or expanded to spaces depending on configuration |
| Rapid repeated undo | Coalescing prevents "stuck" undo; each undo step represents a meaningful change |

## Testing Strategy

### Unit tests

```
test "insert single character":
    editor = createEditor("hello")
    moveCursor(editor, row=0, col=5)
    insertAtCursor(editor, "!")
    assert editor.getText() == "hello!"
    assert editor.buffer.cursor.col == 6

test "insert newline splits line":
    editor = createEditor("hello world")
    moveCursor(editor, row=0, col=5)
    insertAtCursor(editor, "\n")
    assert editor.buffer.lines == ["hello", " world"]
    assert editor.buffer.cursor == { row: 1, col: 0 }

test "backspace at line start joins lines":
    editor = createEditor("hello\nworld")
    moveCursor(editor, row=1, col=0)
    deleteCharBefore(editor)
    assert editor.buffer.lines == ["helloworld"]

test "undo restores previous state":
    editor = createEditor("hello")
    saveUndoPoint(editor)
    insertAtCursor(editor, " world")
    undo(editor)
    assert editor.getText() == "hello"

test "undo coalesces rapid edits":
    editor = createEditor("")
    insertAtCursor(editor, "a")  // t=0
    insertAtCursor(editor, "b")  // t=10ms (within 500ms threshold)
    insertAtCursor(editor, "c")  // t=20ms
    // All three should be one undo point
    undo(editor)
    assert editor.getText() == ""

test "kill and yank round-trip":
    editor = createEditor("hello world")
    moveCursor(editor, row=0, col=5)
    killLine(editor)
    assert editor.getText() == "hello"
    moveToLineEnd(editor)
    yank(editor)
    assert editor.getText() == "hello world"

test "yank-pop cycles kill ring":
    editor = createEditor("")
    editor.killRing.push("first")
    editor.killRing.push("second")
    yank(editor)
    assert editor.getText() == "second"
    yankPop(editor)
    assert editor.getText() == "first"

test "autocomplete accept inserts text":
    editor = createEditor("hel")
    moveCursor(editor, row=0, col=3)
    mockProvider(returns: [{ label: "hello", insertText: "hello" }])
    triggerAutocomplete(editor)
    assert editor.autocomplete.active == true
    acceptCompletion(editor)
    assert editor.getText() == "hello"

test "bracketed paste is single undo point":
    editor = createEditor("")
    handlePaste(editor, "line1\nline2\nline3")
    assert len(editor.buffer.lines) == 3
    undo(editor)
    assert editor.getText() == ""

test "word wrap preserves cursor position":
    editor = createEditor("a very long line that wraps")
    moveCursor(editor, row=0, col=20)
    lines = editor.render(width=15)
    // Cursor should appear in the correct wrapped line
    cursorLine = findLineWithCursorMarker(lines)
    assert cursorLine is not null
```

### Integration tests

```
test "editor component integrates with TUI":
    tui = createTUI()
    editor = createEditorComponent()
    tui.addChild(editor)
    tui.setFocus(editor)
    // Simulate key input
    tui.handleInput("h")
    tui.handleInput("i")
    assert editor.getText() == "hi"
    // Render should contain cursor marker
    lines = tui.render(80)
    assert any(line contains CURSOR_MARKER for line in lines)

test "custom keybindings override defaults":
    editor = createEditor("")
    editor.keybindings.override("ctrl+a", selectAll)
    simulateInput(editor, "ctrl+a")
    // Should call selectAll instead of moveToLineStart
```

## References

- Bracketed Paste Mode: https://invisible-island.net/xterm/ctlseqs/ctlseqs.html
- Kill ring concept: GNU Emacs Manual, "Yanking"
- Kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/
- Source: `packages/tui/src/editor-component.ts`
- Source: `packages/tui/src/autocomplete.ts`
- Source: `packages/tui/src/keys.ts`
