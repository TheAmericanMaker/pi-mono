# Extension & Plugin Architecture

## Purpose

Provide a typed, event-driven extension system that lets third-party modules
hook into every phase of an LLM agent's lifecycle.  Extensions can subscribe to
events, register new tools for the LLM, add slash commands and keyboard
shortcuts, and interact with the user through UI primitives.  The design
isolates extension failures, enforces ordering, and gives "before_" hooks the
power to cancel, block, or transform operations.

## Dependencies

- **tool-system-design.skill.md** -- extensions register tools using the same
  `ToolDefinition` schema that built-in tools use.
- **agent-loop-with-tool-execution.skill.md** -- the agent loop fires the
  lifecycle events that extensions subscribe to.

## Core Concepts

### Extension lifecycle

1. Extensions are discovered at startup (from a configured directory or
   manifest).
2. Each extension exports an `activate(api)` function that receives the
   registration API.
3. During activation, the extension calls `api.on(eventType, handler)` to
   subscribe to events.
4. At shutdown, a `session_shutdown` event is fired so extensions can clean up.

### Hook categories

Events fall into two categories:

| Category       | Naming convention   | Can cancel/modify? |
|----------------|---------------------|--------------------|
| **Before hooks** | `before_*`, `session_before_*` | Yes -- return `{ cancel: true }`, modify payload, or block |
| **Notification hooks** | All others | No -- inform only |

### Handler ordering

Handlers run in **registration order** (the order `api.on()` was called).
Within a single event, each handler runs sequentially.  For `before_` hooks,
later handlers see modifications made by earlier ones.

### Error isolation

Every handler invocation is wrapped in a try/catch.  If one handler throws, the
error is logged but remaining handlers still execute.  This prevents a buggy
extension from breaking the entire agent.

## Data Structures

### Event types -- Resource Discovery

```
record ResourcesDiscoverEvent:
    type   = "resources_discover"
    cwd    : string
    reason : "startup" | "reload"

record ResourcesDiscoverResult:
    skillPaths  : list<string>?
    promptPaths : list<string>?
    themePaths  : list<string>?
```

### Event types -- Session Lifecycle

```
record SessionStartEvent:
    type                : "session_start"
    reason              : "startup" | "reload" | "new" | "resume" | "fork"
    previousSessionFile : string?

record SessionBeforeSwitchEvent:
    type              : "session_before_switch"
    reason            : "new" | "resume"
    targetSessionFile : string?
    // Handler can return { cancel: true }

record SessionBeforeForkEvent:
    type    : "session_before_fork"
    entryId : string
    // Handler can return { cancel: true }

record SessionBeforeCompactEvent:
    type               : "session_before_compact"
    preparation        : CompactionPreparation
    branchEntries      : list<SessionEntry>
    customInstructions : string?
    signal             : AbortSignal

record SessionCompactEvent:
    type            : "session_compact"
    compactionEntry : CompactionEntry
    fromExtension   : bool

record SessionBeforeTreeEvent:
    type        : "session_before_tree"
    preparation : TreePreparation
    signal      : AbortSignal

record SessionTreeEvent:
    type         : "session_tree"
    newLeafId    : string?
    oldLeafId    : string?
    summaryEntry : BranchSummaryEntry?

record SessionShutdownEvent:
    type : "session_shutdown"
```

### Event types -- Agent Lifecycle

```
record BeforeAgentStartEvent:
    type         : "before_agent_start"
    prompt       : string
    images       : list<ImageContent>?
    systemPrompt : string           // mutable: handler can replace

record AgentStartEvent:
    type : "agent_start"

record AgentEndEvent:
    type     : "agent_end"
    messages : list<AgentMessage>

record ContextEvent:
    type     : "context"
    messages : list<AgentMessage>   // mutable: handler can modify before LLM call

record BeforeProviderRequestEvent:
    type    : "before_provider_request"
    payload : any                   // raw API payload; handler can replace
```

### Event types -- Turn Lifecycle

```
record TurnStartEvent:
    type      : "turn_start"
    turnIndex : integer
    timestamp : integer             // epoch millis

record TurnEndEvent:
    type        : "turn_end"
    turnIndex   : integer
    message     : AgentMessage
    toolResults : list<ToolResultMessage>
```

### Event types -- Message Lifecycle

```
record MessageStartEvent:
    type    : "message_start"
    message : AgentMessage

record MessageUpdateEvent:
    type                  : "message_update"
    message               : AgentMessage
    assistantMessageEvent : AssistantMessageEvent   // streaming delta

record MessageEndEvent:
    type    : "message_end"
    message : AgentMessage
```

### Event types -- Tool Lifecycle

```
record ToolExecutionStartEvent:
    type       : "tool_execution_start"
    toolCallId : string
    toolName   : string
    args       : any

record ToolExecutionUpdateEvent:
    type          : "tool_execution_update"
    toolCallId    : string
    toolName      : string
    args          : any
    partialResult : any

record ToolExecutionEndEvent:
    type       : "tool_execution_end"
    toolCallId : string
    toolName   : string
    result     : any
    isError    : bool

// Before tool execution -- can block or mutate args
record ToolCallEvent:
    type       : "tool_call"
    toolCallId : string
    toolName   : string             // "bash", "read", "edit", "write", etc.
    input      : map<string, any>   // mutable in place

// After tool execution -- can modify result
record ToolResultEvent:
    type       : "tool_result"
    toolCallId : string
    toolName   : string
    input      : map<string, any>
    content    : list<TextContent | ImageContent>  // mutable
    details    : any
    isError    : bool
```

### Event types -- Input & Misc

```
record InputEvent:
    type   : "input"
    text   : string
    images : list<ImageContent>?
    source : "interactive" | "rpc" | "extension"

union InputEventResult:
    { action: "continue" }
    | { action: "transform", text: string, images?: list<ImageContent> }
    | { action: "handled" }

record ModelSelectEvent:
    type          : "model_select"
    model         : Model
    previousModel : Model?
    source        : "set" | "cycle" | "restore"

record UserBashEvent:
    type               : "user_bash"
    command            : string
    excludeFromContext : bool
    cwd                : string
```

### Registration API

```
interface ExtensionAPI:
    // Subscribe to a lifecycle event
    on(eventType: string, handler: function) -> void

    // Register a tool the LLM can call
    registerTool(definition: ToolDefinition) -> void

    // Register a slash command (e.g. /deploy)
    registerCommand(name: string, options: CommandOptions) -> void

    // Register a keyboard shortcut
    registerShortcut(key: string, options: ShortcutOptions) -> void

    // Register a CLI flag (e.g. --verbose)
    registerFlag(flag: string, options: FlagOptions) -> void
```

### Tool Definition

```
record ToolDefinition:
    name             : string       // LLM-visible tool name
    label            : string       // human-readable label for UI
    description      : string       // description sent to LLM
    promptSnippet    : string?      // one-liner for "Available tools" system prompt section
    promptGuidelines : list<string>? // bullet points appended to Guidelines section
    parameters       : Schema       // JSON Schema / TypeBox schema for params

    // Optional argument pre-processing (before schema validation)
    prepareArguments : function(raw: any) -> params?

    // Execute the tool
    execute(toolCallId, params, signal, onUpdate, ctx) -> ToolResult

    // Custom rendering (optional)
    renderCall   : function(args, theme, context) -> Component?
    renderResult : function(result, options, theme, context) -> Component?
```

### Extension Context

```
record ExtensionContext:
    ui             : ExtensionUIContext
    hasUI          : bool               // false in headless/RPC mode
    cwd            : string
    sessionManager : ReadonlySessionManager
    modelRegistry  : ModelRegistry
    model          : Model?
    signal         : AbortSignal?       // current agent operation signal

    isIdle()            -> bool
    abort()             -> void
    hasPendingMessages() -> bool
    shutdown()          -> void
    getContextUsage()   -> ContextUsage?
    compact(options?)   -> void
    getSystemPrompt()   -> string
```

### Extended Context (for commands)

```
record ExtensionCommandContext extends ExtensionContext:
    waitForIdle()                           -> Promise<void>
    newSession(options?)                    -> Promise<{ cancelled: bool }>
    fork(entryId)                           -> Promise<{ cancelled: bool }>
    navigateTree(targetId, options?)         -> Promise<{ cancelled: bool }>
    switchSession(sessionPath)              -> Promise<{ cancelled: bool }>
    reload()                                -> Promise<void>
```

### UI Context

```
interface ExtensionUIContext:
    // Modal dialogs
    select(title, options, opts?)   -> Promise<string?>
    confirm(title, message, opts?)  -> Promise<bool>
    input(title, placeholder?, opts?) -> Promise<string?>

    // Notifications
    notify(message, type?)          -> void     // type: "info" | "warning" | "error"

    // Persistent UI elements
    setStatus(key, text?)           -> void     // footer status
    setWorkingMessage(message?)     -> void     // streaming indicator
    setWidget(key, content?, opts?) -> void     // above/below editor
    setFooter(factory?)             -> void     // custom footer component
    setHeader(factory?)             -> void     // custom header component
    setTitle(title)                 -> void     // terminal window title

    // Editor interaction
    setEditorText(text)             -> void
    getEditorText()                 -> string
    pasteToEditor(text)             -> void
    editor(title, prefill?)         -> Promise<string?>
    setEditorComponent(factory?)    -> void     // replace entire editor

    // Advanced
    custom<T>(factory, options?)    -> Promise<T>   // focused custom component
    onTerminalInput(handler)        -> unsubscribe  // raw input listener

    // Theme
    theme                           -> Theme (readonly)
    getAllThemes()                   -> list<{ name, path? }>
    getTheme(name)                  -> Theme?
    setTheme(nameOrTheme)           -> { success: bool, error?: string }

    // Tool display
    getToolsExpanded()              -> bool
    setToolsExpanded(expanded)      -> void
```

## Algorithm / Flow

### Event dispatch

```
function dispatchEvent(runtime, event):
    handlers = runtime.handlersByType[event.type]
    results = []

    for handler in handlers:                // registration order
        try:
            result = await handler(event, runtime.context)
            if result is not undefined:
                results.append(result)
        catch error:
            logError("Extension handler failed", event.type, error)
            // Continue to next handler -- error isolation

    return results
```

### Before-hook cancellation

```
function dispatchBeforeHook(runtime, event):
    handlers = runtime.handlersByType[event.type]

    for handler in handlers:
        try:
            result = await handler(event, runtime.context)
            if result?.cancel == true:
                return { cancelled: true }
        catch error:
            logError("Extension before-hook failed", event.type, error)

    return { cancelled: false }
```

### Tool registration

```
function registerTool(runtime, definition):
    // Validate: name must be unique
    if runtime.tools.has(definition.name):
        throw "Tool name already registered: " + definition.name

    // Add to tool registry
    runtime.tools.set(definition.name, definition)

    // If promptSnippet provided, it will be injected into system prompt
    if definition.promptSnippet:
        runtime.toolSnippets.append(definition.name + ": " + definition.promptSnippet)

    // If promptGuidelines provided, they will be appended to Guidelines section
    if definition.promptGuidelines:
        runtime.toolGuidelines.appendAll(definition.promptGuidelines)
```

### Input event flow

```
function handleUserInput(runtime, text, images, source):
    event = InputEvent(text, images, source)
    results = dispatchEvent(runtime, event)

    for result in results:
        if result.action == "handled":
            return    // extension consumed the input

        if result.action == "transform":
            text = result.text
            images = result.images or images

    // Proceed with agent loop using (possibly transformed) input
    startAgentLoop(text, images)
```

### Tool call interception

```
function executeToolWithHooks(runtime, toolCallId, toolName, input):
    // 1. Fire tool_call event (can block or mutate input)
    callEvent = ToolCallEvent(toolCallId, toolName, input)
    for handler in runtime.handlersByType["tool_call"]:
        try:
            result = handler(callEvent, runtime.context)
            if result?.block == true:
                return blockedResult(result.message)
        catch error:
            logError("tool_call handler failed", error)
    // input may have been mutated in-place by handlers

    // 2. Execute tool
    toolResult = await tool.execute(toolCallId, input, ...)

    // 3. Fire tool_result event (can modify content)
    resultEvent = ToolResultEvent(toolCallId, toolName, input,
                                  toolResult.content, toolResult.isError)
    dispatchEvent(runtime, resultEvent)
    // content may have been mutated in-place by handlers

    return toolResult
```

## Key Design Decisions

1. **Registration order execution.**  Handlers run in the order they were
   registered (via `api.on()`).  This gives deterministic behavior when
   multiple extensions listen to the same event.

2. **Error isolation via try/catch per handler.**  A single buggy extension
   cannot crash the agent or prevent other extensions from running.

3. **Before-hooks can cancel; notification hooks cannot.**  This split makes it
   clear which events are control points vs. informational.

4. **Mutable event payloads for interception.**  `tool_call` events expose
   `input` as mutable.  Handlers mutate in-place rather than returning copies.
   This lets later handlers see earlier mutations without extra bookkeeping.
   No re-validation is performed after mutation -- extensions are trusted.

5. **System prompt injection via `promptSnippet` and `promptGuidelines`.**
   Registered tools can contribute to the system prompt, ensuring the LLM knows
   about extension-provided capabilities without the extension needing to hook
   `before_agent_start` manually.

6. **UI abstraction.**  The `ExtensionUIContext` interface has multiple
   implementations (interactive terminal, RPC/headless, print mode).  Extensions
   use the same API regardless of the runtime mode; `hasUI` lets them
   conditionally skip interactive flows.

7. **Extended context for commands only.**  Session-mutating methods (`fork`,
   `newSession`, `navigateTree`) are only available in `ExtensionCommandContext`,
   not in event handlers.  This prevents event handlers from causing unexpected
   session switches mid-stream.

8. **Custom editor replacement.**  Extensions can replace the entire input
   editor (e.g. vim mode) via `setEditorComponent`.  The factory receives theme
   and keybindings so the custom editor can integrate with app-level shortcuts.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| Extension throws during `activate()` | Error logged; extension skipped; other extensions still load |
| Handler throws during event dispatch | Error logged; remaining handlers execute normally |
| Two extensions register tools with same name | Second registration throws; extension activation fails |
| `before_agent_start` handler modifies `systemPrompt` | LLM sees the modified prompt for that invocation only |
| `tool_call` handler blocks a tool | Agent receives a "blocked" result; LLM can try a different approach |
| `tool_call` handler mutates args to invalid values | Tool execution may fail; error surfaces as tool error to LLM |
| Extension calls `ctx.abort()` during event handler | Current agent streaming is aborted; agent returns to idle |
| Extension calls `ctx.shutdown()` | Graceful process exit; `session_shutdown` fires for cleanup |
| `custom()` called when `hasUI` is false | Depends on implementation; should throw or return a default |
| Overlay created during `session_shutdown` | May not display; shutdown proceeds regardless |
| `setEditorComponent` called mid-input | Current input text is preserved; new editor takes over |
| `resources_discover` returns paths that don't exist | Loader logs warnings; missing paths are skipped |
| `input` handler returns `{ action: "handled" }` | Agent loop is not started; input is consumed silently |
| Multiple handlers return `{ action: "transform" }` | Each transform applies cumulatively (last writer wins per field) |

## Testing Strategy

### Unit tests

```
test "handlers run in registration order":
    runtime = createRuntime()
    order = []
    runtime.on("agent_start", () => order.append("first"))
    runtime.on("agent_start", () => order.append("second"))
    dispatchEvent(runtime, AgentStartEvent())
    assert order == ["first", "second"]

test "error in handler does not break others":
    runtime = createRuntime()
    called = false
    runtime.on("agent_start", () => throw Error("boom"))
    runtime.on("agent_start", () => called = true)
    dispatchEvent(runtime, AgentStartEvent())
    assert called == true

test "before_hook cancellation stops processing":
    runtime = createRuntime()
    runtime.on("session_before_switch", () => { cancel: true })
    result = dispatchBeforeHook(runtime, SessionBeforeSwitchEvent())
    assert result.cancelled == true

test "tool_call handler can block execution":
    runtime = createRuntime()
    runtime.on("tool_call", (event) =>
        if event.toolName == "bash": return { block: true, message: "blocked" }
    )
    result = executeToolWithHooks(runtime, "id", "bash", { command: "rm -rf /" })
    assert result.isError == true

test "tool_call handler can mutate args":
    runtime = createRuntime()
    runtime.on("tool_call", (event) =>
        if event.toolName == "bash":
            event.input.command = "echo safe"
    )
    // After dispatch, input.command should be "echo safe"

test "registerTool adds to system prompt snippets":
    runtime = createRuntime()
    runtime.registerTool({
        name: "deploy",
        promptSnippet: "Deploy to production",
        ...
    })
    assert "deploy: Deploy to production" in runtime.toolSnippets
```

### Integration tests

```
test "extension activates and registers command":
    extension = loadExtension("./test-extension")
    runtime = createRuntime()
    extension.activate(runtime.api)
    assert runtime.commands.has("test-cmd")

test "input transform flows through to agent":
    runtime = createRuntime()
    runtime.on("input", (event) =>
        return { action: "transform", text: event.text.toUpperCase() }
    )
    handleUserInput(runtime, "hello", [], "interactive")
    // Agent should receive "HELLO"

test "session_shutdown fires on process exit":
    runtime = createRuntime()
    shutdownCalled = false
    runtime.on("session_shutdown", () => shutdownCalled = true)
    runtime.shutdown()
    assert shutdownCalled == true
```

## References

- VS Code Extension API (inspiration for lifecycle hooks):
  https://code.visualstudio.com/api
- Source: `packages/coding-agent/src/core/extensions/types.ts`
