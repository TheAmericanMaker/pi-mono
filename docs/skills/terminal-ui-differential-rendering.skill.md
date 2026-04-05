# Terminal UI with Differential Rendering

## Purpose

Render a component tree to a terminal efficiently by comparing the current
frame's output with the previous frame and only rewriting lines that changed.
Use synchronized output escape sequences to prevent visible flicker during
redraws.  Support an overlay system for modal dialogs rendered on top of base
content, focus management, and IME cursor positioning via a zero-width marker.

## Dependencies

None.  This is a foundational terminal rendering layer.

## Core Concepts

### Component model

Every visual element implements a `Component` interface.  Components are
arranged in a tree rooted at the `TUI` object (which is itself a `Container`).
Rendering is pull-based: the TUI calls `render(width)` on each child and
concatenates the results.

### Differential rendering

Instead of clearing the screen and redrawing everything, the TUI keeps a
`previousLines` array.  On each frame, it compares the new lines against the
previous ones and writes only the changed lines using cursor-positioning escape
codes.  This drastically reduces terminal I/O and visual flicker.

### Synchronized output

Wrapping a batch of writes in `CSI ? 2026 h` (begin) and `CSI ? 2026 l` (end)
tells the terminal to buffer all output and display it atomically.  Terminals
that don't support the protocol simply ignore the sequences.

### Overlay system

Modal components (dialogs, menus, autocomplete) are rendered as overlays on
top of base content.  Overlays have configurable positioning (anchor, absolute,
percentage-based) and stack with focus management.

### IME / Hardware cursor

CJK input methods need the hardware cursor at the correct position for their
candidate window.  Focused components emit a zero-width `CURSOR_MARKER` escape
at the logical cursor position.  The TUI strips this marker and positions the
real terminal cursor there.

## Data Structures

```
// ---- Component interface ----

interface Component:
    render(width: integer) -> list<string>    // one string per line
    handleInput(data: string)?                // keyboard input handler
    wantsKeyRelease : bool?                   // opt-in to key release events
    invalidate()                              // clear cached render state

// ---- Focusable interface ----

interface Focusable:
    focused : bool                            // set by TUI on focus change

// ---- Container ----

class Container implements Component:
    children : list<Component>

    addChild(component)
    removeChild(component)
    clear()

    render(width):
        lines = []
        for child in children:
            lines.extend(child.render(width))
        return lines

    invalidate():
        for child in children:
            child.invalidate()

// ---- Overlay entry ----

record OverlayEntry:
    component  : Component
    options    : OverlayOptions?
    preFocus   : Component?            // component that had focus before overlay
    hidden     : bool
    focusOrder : integer               // monotonic counter for visual z-order

// ---- Overlay options ----

record OverlayOptions:
    // Sizing
    width       : integer | percentage?     // e.g. 80 or "50%"
    minWidth    : integer?
    maxHeight   : integer | percentage?

    // Positioning -- anchor-based
    anchor      : OverlayAnchor?            // "center", "top-left", etc.
    offsetX     : integer?
    offsetY     : integer?

    // Positioning -- absolute or percentage
    row         : integer | percentage?
    col         : integer | percentage?

    // Margins
    margin      : OverlayMargin | integer?

    // Visibility
    visible     : function(termWidth, termHeight) -> bool?
    nonCapturing : bool?                    // don't steal focus

// ---- Overlay handle ----

interface OverlayHandle:
    hide()                  // permanently remove overlay
    setHidden(hidden: bool) // temporarily show/hide
    isHidden() -> bool
    focus()                 // bring to front and focus
    unfocus()               // release focus to previous target
    isFocused() -> bool

// ---- OverlayAnchor enum ----

enum OverlayAnchor:
    center, top-left, top-right, bottom-left, bottom-right,
    top-center, bottom-center, left-center, right-center

// ---- TUI ----

class TUI extends Container:
    terminal          : Terminal
    previousLines     : list<string>
    previousWidth     : integer
    previousHeight    : integer
    focusedComponent  : Component?
    overlayStack      : list<OverlayEntry>
    renderRequested   : bool
    inputListeners    : set<InputListener>
    maxLinesRendered  : integer
    hardwareCursorRow : integer
    focusOrderCounter : integer
    stopped           : bool

// ---- Constants ----

CURSOR_MARKER = "\x1b_pi:c\x07"     // APC sequence, zero visual width
SEGMENT_RESET = "\x1b[0m\x1b]8;;\x07"  // reset colors + close any hyperlink
```

## Algorithm / Flow

### 1. Render cycle (doRender)

```
function doRender(tui):
    if tui.stopped: return

    width  = tui.terminal.columns
    height = tui.terminal.rows
    widthChanged  = tui.previousWidth != 0 and tui.previousWidth != width
    heightChanged = tui.previousHeight != 0 and tui.previousHeight != height

    // ---- Phase 1: Render component tree ----
    newLines = tui.render(width)       // inherited from Container

    // ---- Phase 2: Composite overlays ----
    if tui.overlayStack is not empty:
        newLines = compositeOverlays(tui, newLines, width, height)

    // ---- Phase 3: Extract cursor marker ----
    cursorPos = extractCursorPosition(newLines, height)

    // ---- Phase 4: Append reset sequences ----
    for i in range(len(newLines)):
        if not isImageLine(newLines[i]):
            newLines[i] = newLines[i] + SEGMENT_RESET

    // ---- Phase 5: Decide render strategy ----

    // First render -- no previous state
    if tui.previousLines is empty and not widthChanged and not heightChanged:
        fullRender(tui, newLines, cursorPos, clear=false)
        return

    // Width changed -- wrapping invalidated, must clear and redraw
    if widthChanged:
        fullRender(tui, newLines, cursorPos, clear=true)
        return

    // Height changed (except Termux workaround)
    if heightChanged and not isTermuxSession():
        fullRender(tui, newLines, cursorPos, clear=true)
        return

    // Content shrunk below working area (no overlays active)
    if tui.clearOnShrink and len(newLines) < tui.maxLinesRendered
       and tui.overlayStack is empty:
        fullRender(tui, newLines, cursorPos, clear=true)
        return

    // ---- Phase 6: Differential update ----
    differentialRender(tui, newLines, cursorPos, width, height)
```

### 2. Full render

```
function fullRender(tui, newLines, cursorPos, clear):
    buffer = "\x1b[?2026h"                        // begin synchronized output
    if clear:
        buffer += "\x1b[2J\x1b[H\x1b[3J"         // clear screen + scrollback
    for i in range(len(newLines)):
        if i > 0: buffer += "\r\n"
        buffer += newLines[i]
    buffer += "\x1b[?2026l"                        // end synchronized output
    tui.terminal.write(buffer)

    tui.cursorRow = max(0, len(newLines) - 1)
    tui.hardwareCursorRow = tui.cursorRow
    if clear:
        tui.maxLinesRendered = len(newLines)
    else:
        tui.maxLinesRendered = max(tui.maxLinesRendered, len(newLines))

    positionHardwareCursor(tui, cursorPos, len(newLines))
    tui.previousLines = newLines
    tui.previousWidth = tui.terminal.columns
    tui.previousHeight = tui.terminal.rows
```

### 3. Differential render

```
function differentialRender(tui, newLines, cursorPos, width, height):
    // Find range of changed lines
    firstChanged = -1
    lastChanged  = -1
    maxLen = max(len(newLines), len(tui.previousLines))

    for i in range(maxLen):
        oldLine = tui.previousLines[i] if i < len(tui.previousLines) else ""
        newLine = newLines[i] if i < len(newLines) else ""
        if oldLine != newLine:
            if firstChanged == -1: firstChanged = i
            lastChanged = i

    // Handle appended lines
    if len(newLines) > len(tui.previousLines):
        if firstChanged == -1: firstChanged = len(tui.previousLines)
        lastChanged = len(newLines) - 1

    // No changes at all
    if firstChanged == -1:
        positionHardwareCursor(tui, cursorPos, len(newLines))
        return

    // Build output buffer
    buffer = "\x1b[?2026h"     // begin synchronized output

    for i in range(firstChanged, lastChanged + 1):
        if i < len(newLines):
            // Move cursor to row i
            lineDiff = computeLineDiff(tui, i)
            if lineDiff > 0: buffer += "\x1b[{lineDiff}B"
            elif lineDiff < 0: buffer += "\x1b[{-lineDiff}A"
            buffer += "\r"         // go to column 0
            buffer += "\x1b[2K"    // clear entire line
            buffer += newLines[i]
            tui.hardwareCursorRow = i

    // Handle deleted lines (content shrunk)
    if len(newLines) < len(tui.previousLines):
        for i in range(len(newLines), len(tui.previousLines)):
            lineDiff = computeLineDiff(tui, i)
            if lineDiff > 0: buffer += "\x1b[{lineDiff}B"
            elif lineDiff < 0: buffer += "\x1b[{-lineDiff}A"
            buffer += "\r\x1b[2K"  // clear line
            tui.hardwareCursorRow = i

    buffer += "\x1b[?2026l"    // end synchronized output
    tui.terminal.write(buffer)

    positionHardwareCursor(tui, cursorPos, len(newLines))
    tui.previousLines = newLines
    tui.previousWidth = width
    tui.previousHeight = height
    tui.maxLinesRendered = max(tui.maxLinesRendered, len(newLines))
```

### 4. Overlay compositing

```
function compositeOverlays(tui, baseLines, termWidth, termHeight):
    result = copy(baseLines)

    // Collect visible overlays, sorted by focusOrder (lowest first = behind)
    visibleEntries = [e for e in tui.overlayStack if isOverlayVisible(e)]
    sort(visibleEntries, by: e.focusOrder, ascending)

    minLinesNeeded = len(result)

    // Pre-render each overlay
    rendered = []
    for entry in visibleEntries:
        { width, maxHeight } = resolveOverlayLayout(entry.options, 0, termWidth, termHeight)

        overlayLines = entry.component.render(width)
        if maxHeight is not undefined and len(overlayLines) > maxHeight:
            overlayLines = overlayLines[0:maxHeight]

        { row, col } = resolveOverlayLayout(entry.options, len(overlayLines), termWidth, termHeight)
        rendered.append({ overlayLines, row, col, width })
        minLinesNeeded = max(minLinesNeeded, row + len(overlayLines))

    // Pad result to fill viewport (overlays need screen-relative positions)
    workingHeight = max(len(result), termHeight, minLinesNeeded)
    while len(result) < workingHeight:
        result.append("")

    viewportStart = max(0, workingHeight - termHeight)

    // Composite each overlay onto result
    for { overlayLines, row, col, width } in rendered:
        for i in range(len(overlayLines)):
            idx = viewportStart + row + i
            if 0 <= idx < len(result):
                result[idx] = compositeLineAt(result[idx], overlayLines[i],
                                               col, width, termWidth)

    return result
```

### 5. Line compositing

```
function compositeLineAt(baseLine, overlayLine, startCol, overlayWidth, totalWidth):
    if isImageLine(baseLine): return baseLine

    // Extract base segments: before overlay and after overlay
    afterStart = startCol + overlayWidth
    base = extractSegments(baseLine, startCol, afterStart, totalWidth - afterStart)

    // Extract overlay with width tracking
    overlay = sliceWithWidth(overlayLine, 0, overlayWidth)

    // Pad to target widths
    beforePad  = max(0, startCol - base.beforeWidth)
    overlayPad = max(0, overlayWidth - overlay.width)
    afterTarget = max(0, totalWidth - startCol - overlayWidth)
    afterPad    = max(0, afterTarget - base.afterWidth)

    result = base.before + " " * beforePad
           + SEGMENT_RESET
           + overlay.text + " " * overlayPad
           + SEGMENT_RESET
           + base.after + " " * afterPad

    // Final safeguard: truncate to terminal width
    if visibleWidth(result) > totalWidth:
        result = sliceByColumn(result, 0, totalWidth)

    return result
```

### 6. Cursor position extraction

```
function extractCursorPosition(lines, viewportHeight):
    viewportTop = max(0, len(lines) - viewportHeight)

    // Scan from bottom up (cursor is usually near the bottom)
    for row in range(len(lines) - 1, viewportTop - 1, -1):
        idx = lines[row].indexOf(CURSOR_MARKER)
        if idx >= 0:
            beforeMarker = lines[row][0:idx]
            col = visibleWidth(beforeMarker)
            // Strip marker from line
            lines[row] = lines[row][0:idx] + lines[row][idx + len(CURSOR_MARKER):]
            return { row, col }

    return null
```

### 7. Input handling

```
function handleInput(tui, data):
    // 1. Run input listeners (can consume or transform)
    for listener in tui.inputListeners:
        result = listener(data)
        if result?.consume: return
        if result?.data is not undefined: data = result.data
    if len(data) == 0: return

    // 2. Consume terminal cell size responses (CSI 6;H;W t)
    if consumeCellSizeResponse(tui, data): return

    // 3. Global debug key
    if matchesKey(data, "shift+ctrl+d") and tui.onDebug:
        tui.onDebug()
        return

    // 4. Redirect focus if focused overlay is no longer visible
    focusedOverlay = findOverlayByComponent(tui, tui.focusedComponent)
    if focusedOverlay and not isOverlayVisible(focusedOverlay):
        topVisible = getTopmostVisibleOverlay(tui)
        tui.setFocus(topVisible?.component or focusedOverlay.preFocus)

    // 5. Forward to focused component
    if tui.focusedComponent?.handleInput:
        // Filter key release events unless component opts in
        if isKeyRelease(data) and not tui.focusedComponent.wantsKeyRelease:
            return
        tui.focusedComponent.handleInput(data)
        tui.requestRender()
```

### 8. Overlay management

```
function showOverlay(tui, component, options) -> OverlayHandle:
    entry = OverlayEntry(component, options,
                          preFocus=tui.focusedComponent,
                          hidden=false,
                          focusOrder=++tui.focusOrderCounter)
    tui.overlayStack.append(entry)

    if not options?.nonCapturing and isOverlayVisible(entry):
        tui.setFocus(component)
    tui.terminal.hideCursor()
    tui.requestRender()

    return OverlayHandle:
        hide():
            remove entry from tui.overlayStack
            if tui.focusedComponent == component:
                tui.setFocus(getTopmostVisibleOverlay(tui)?.component or entry.preFocus)
            tui.requestRender()

        setHidden(hidden):
            if entry.hidden == hidden: return
            entry.hidden = hidden
            if hidden and tui.focusedComponent == component:
                tui.setFocus(getTopmostVisibleOverlay(tui)?.component or entry.preFocus)
            elif not hidden and not options?.nonCapturing and isOverlayVisible(entry):
                entry.focusOrder = ++tui.focusOrderCounter
                tui.setFocus(component)
            tui.requestRender()

        focus():
            if not isOverlayVisible(entry): return
            tui.setFocus(component)
            entry.focusOrder = ++tui.focusOrderCounter
            tui.requestRender()

        unfocus():
            if tui.focusedComponent != component: return
            top = getTopmostVisibleOverlay(tui)
            tui.setFocus(top?.component or entry.preFocus)
            tui.requestRender()

function hideOverlay(tui):
    entry = tui.overlayStack.pop()
    if entry is null: return
    if tui.focusedComponent == entry.component:
        tui.setFocus(getTopmostVisibleOverlay(tui)?.component or entry.preFocus)
    tui.requestRender()
```

### 9. Focus management

```
function setFocus(tui, component):
    if isFocusable(tui.focusedComponent):
        tui.focusedComponent.focused = false
    tui.focusedComponent = component
    if isFocusable(component):
        component.focused = true
```

### 10. Render request coalescing

```
function requestRender(tui, force=false):
    if force:
        // Reset all previous state to force a full redraw
        tui.previousLines = []
        tui.previousWidth = -1
        tui.previousHeight = -1
        tui.cursorRow = 0
        tui.hardwareCursorRow = 0
        tui.maxLinesRendered = 0

    if tui.renderRequested: return
    tui.renderRequested = true
    scheduleNextTick(() =>
        tui.renderRequested = false
        tui.doRender()
    )
```

## Key Design Decisions

1. **Line-based differential comparison.**  Comparing entire lines (as strings
   with embedded escape codes) is simple and fast.  It avoids the complexity of
   a cell-grid diffing algorithm while still achieving good efficiency for
   typical TUI workloads where most lines are unchanged between frames.

2. **Synchronized output protocol.**  The `CSI ? 2026 h/l` sequences ensure
   that even complex multi-line updates appear atomically.  Terminals that
   don't support the protocol simply ignore the sequences, so there is no
   compatibility cost.

3. **Overlay compositing in string space.**  Rather than maintaining a 2D cell
   grid, overlays are spliced into the line array at their target positions.
   This keeps the rendering pipeline simple and avoids double-buffering.

4. **Focus order via monotonic counter.**  Each overlay gets an incrementing
   `focusOrder` value.  Visual z-order (which overlay renders on top) follows
   this counter, ensuring the most recently focused overlay is always visible.

5. **IME cursor via zero-width marker.**  Using an APC escape sequence as a
   cursor position marker is invisible to the terminal but detectable by the
   TUI.  This avoids a separate cursor-tracking data structure and works with
   any component that can emit the marker in its render output.

6. **Content shrink handling is configurable.**  Clearing orphaned lines when
   content shrinks requires a full redraw, which can be expensive.  The
   `clearOnShrink` option lets slower terminals (e.g. Termux) opt out.

7. **Render coalescing via next-tick scheduling.**  Multiple `requestRender()`
   calls within the same tick are batched into a single render.  This prevents
   redundant work when multiple components invalidate simultaneously.

8. **Width-change forces full redraw.**  Terminal width changes invalidate all
   line wrapping.  Rather than trying to diff wrapped vs. unwrapped lines, the
   TUI simply clears and redraws.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| Terminal resize during render | Width/height changes are detected at the start of `doRender`; if dimensions differ from previous, a full redraw is triggered |
| Content shrinks (fewer lines than before) | Orphaned lines are cleared by moving cursor to them and erasing; or full redraw if `clearOnShrink` is enabled |
| Overlay larger than viewport | Layout clamping ensures overlay row/col stay within margins; content is truncated by `maxHeight` |
| Multiple overlays with focus | Focus goes to overlay with highest `focusOrder`; lower overlays still render but don't receive input |
| Non-capturing overlay | Overlay renders but doesn't steal focus; input goes to the previously focused component |
| Overlay visibility changes due to resize | `visible()` callback is re-evaluated each render; if overlay becomes invisible, focus redirects to next visible overlay or preFocus |
| Component emits lines wider than `width` | Final safeguard truncates composited lines to terminal width |
| Wide character (CJK) at overlay boundary | `sliceByColumn` with `strict=true` omits the wide char rather than splitting it, preventing rendering artifacts |
| ANSI/OSC sequences affect visible width calculation | `visibleWidth` strips escape sequences; `extractSegments` preserves them correctly across slice boundaries |
| Image lines (Kitty/iTerm2 protocol) | Image lines are detected and excluded from compositing (overlay cannot overlay images) |
| Termux environment | Height changes are ignored to avoid full-redraw thrashing when the soft keyboard toggles |
| `requestRender(force=true)` | Resets all previous state, ensuring a complete redraw from scratch on next tick |

## Testing Strategy

### Unit tests

```
test "Container renders children in order":
    c = Container()
    c.addChild(MockComponent(renders: ["line1"]))
    c.addChild(MockComponent(renders: ["line2"]))
    assert c.render(80) == ["line1", "line2"]

test "differential render writes only changed lines":
    tui = createTUI(width=40, height=10)
    tui.previousLines = ["line0", "line1", "line2"]
    // Change line1 only
    newLines = ["line0", "CHANGED", "line2"]
    output = captureDifferentialOutput(tui, newLines)
    assert output contains "CHANGED"
    assert output does not contain "line0"
    assert output does not contain "line2"

test "overlay composites at correct position":
    tui = createTUI(width=20, height=10)
    base = ["hello world         "]
    overlay = showOverlay(tui, MockComponent(renders: ["BOX"]),
                          { anchor: "top-left", row: 0, col: 5 })
    result = compositeOverlays(tui, base, 20, 10)
    // "hello" + "BOX" should appear at column 5
    assert visibleContentAt(result[0], col=5, len=3) == "BOX"

test "cursor marker extraction":
    lines = ["abc" + CURSOR_MARKER + "def"]
    pos = extractCursorPosition(lines, 10)
    assert pos == { row: 0, col: 3 }
    assert lines[0] == "abcdef"     // marker stripped

test "focus management preserves preFocus":
    tui = createTUI()
    editor = MockFocusable()
    tui.setFocus(editor)
    overlay = showOverlay(tui, MockComponent(), {})
    assert tui.focusedComponent == overlay.component
    overlay.hide()
    assert tui.focusedComponent == editor

test "non-capturing overlay does not steal focus":
    tui = createTUI()
    editor = MockFocusable()
    tui.setFocus(editor)
    showOverlay(tui, MockComponent(), { nonCapturing: true })
    assert tui.focusedComponent == editor
```

### Visual regression tests

```
test "no flicker on rapid updates":
    // Render 100 frames in succession
    // Verify synchronized output sequences bracket every write
    for frame in frames:
        output = captureOutput(tui, frame)
        assert output starts with "\x1b[?2026h"
        assert output ends with "\x1b[?2026l"

test "width change triggers full redraw":
    tui = createTUI(width=80)
    tui.doRender()
    tui.terminal.columns = 120
    output = captureOutput(tui)
    assert output contains "\x1b[2J"    // clear screen sequence
```

### Overlay layout tests

```
test "center anchor":
    layout = resolveOverlayLayout({ anchor: "center" }, overlayHeight=5,
                                   termWidth=80, termHeight=24)
    assert layout.row == (24 - 5) / 2
    assert layout.col == (80 - layout.width) / 2

test "percentage-based sizing":
    layout = resolveOverlayLayout({ width: "50%", maxHeight: "30%" },
                                   overlayHeight=100, termWidth=80, termHeight=24)
    assert layout.width == 40
    assert layout.maxHeight == 7

test "margin clamping":
    layout = resolveOverlayLayout({ anchor: "top-left", margin: 2 },
                                   overlayHeight=5, termWidth=80, termHeight=24)
    assert layout.row >= 2
    assert layout.col >= 2
```

## References

- Synchronized Output protocol: https://gist.github.com/christianparpart/d8a62cc1ab659194571927c0b8e5a8d5
- Kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/
- APC (Application Program Command): ECMA-48, Section 8.3.2
- Source: `packages/tui/src/tui.ts`
