# Tool System Design

## Purpose

Define a declarative tool system where tools are registered with a name,
description, JSON Schema parameters, and an execute function. The agent loop
orchestrates the full lifecycle: schema validation, permission checks
(beforeToolCall), execution with streaming partial results, post-processing
(afterToolCall), and result emission. Tools can run sequentially or in parallel.

This design achieves:

- **Safety** -- schema validation catches hallucinated arguments before they
  reach tool code.
- **Permission control** -- beforeToolCall hooks enable approval flows for
  dangerous operations (file writes, shell commands).
- **Observability** -- streaming partial results let the UI show progress during
  long-running tools (large file reads, slow commands).
- **Separation of concerns** -- tools produce content (for the LLM) and details
  (for the UI) independently.

## Dependencies

- `message-transformation-pipeline.skill.md` -- tool results are emitted as
  `ToolResultMessage` instances that flow through the message pipeline.

## Core Concepts

### Declarative Tool Definition

Each tool is a struct with:

- **name**: identifier used in tool calls (e.g., `"read"`, `"bash"`, `"edit"`)
- **description**: natural language description sent to the LLM so it knows when
  and how to use the tool
- **parameters**: a JSON Schema that defines the expected arguments
- **execute**: an async function that receives validated arguments and returns a
  result

The LLM receives the name, description, and parameter schema. It generates tool
calls with arguments that (ideally) conform to the schema. The agent loop
validates before executing.

### AgentTool (Extended Tool)

The runtime tool type extends the base with:

- **label**: human-readable name for UI display
- **prepareArguments**: optional compatibility shim that normalizes raw
  arguments before schema validation (e.g., renaming legacy parameter names)
- **execute**: takes toolCallId, validated params, AbortSignal, and an update
  callback for streaming partial results
- **onUpdate**: callback for streaming intermediate results during execution

### Lifecycle Hooks

Two hooks wrap every tool execution:

- **beforeToolCall**: runs after validation, before execution. Can block the
  tool (e.g., require user approval). Returns `{ block: true, reason: "..." }`
  to prevent execution.
- **afterToolCall**: runs after execution, before result emission. Can override
  the content, details, or error flag of the result. Uses field-by-field merge
  (no deep merge).

### Execution Modes

- **Sequential**: each tool call is fully processed (validate, before, execute,
  after, emit) before the next one starts.
- **Parallel**: all tool calls are preflighted sequentially (validate, before),
  then non-blocked tools execute concurrently, then results are finalized
  (after, emit) in original source order.

## Data Structures

### Tool (Base)

```
Tool:
    name: String                    # unique identifier
    description: String             # natural language for LLM
    parameters: JSONSchema          # argument schema
```

### AgentTool (Runtime)

```
AgentTool<TParameters, TDetails> extends Tool:
    label: String                   # human-readable UI label
    prepareArguments?: Function(rawArgs: Any) -> TParameters
    execute: Function(
        toolCallId: String,
        params: TParameters,
        signal?: AbortSignal,
        onUpdate?: AgentToolUpdateCallback<TDetails>
    ) -> Promise<AgentToolResult<TDetails>>
```

### AgentToolResult

```
AgentToolResult<T>:
    content: List<TextContent | ImageContent>   # returned to the LLM
    details: T                                   # returned to the UI/logs only
```

### AgentToolUpdateCallback

```
AgentToolUpdateCallback<T> = Function(partialResult: AgentToolResult<T>) -> Void
```

Called by the tool during execution to stream intermediate results. The agent
loop forwards these as `tool_execution_update` events.

### BeforeToolCallContext

```
BeforeToolCallContext:
    assistantMessage: AssistantMessage   # the message that requested the tool call
    toolCall: ToolCall                   # the specific tool call block
    args: Any                            # validated arguments
    context: AgentContext                # current agent state
```

### BeforeToolCallResult

```
BeforeToolCallResult:
    block?: Boolean         # if true, prevent execution
    reason?: String         # text shown in the error tool result
```

### AfterToolCallContext

```
AfterToolCallContext:
    assistantMessage: AssistantMessage
    toolCall: ToolCall
    args: Any
    result: AgentToolResult     # the executed result before overrides
    isError: Boolean            # whether execution threw
    context: AgentContext
```

### AfterToolCallResult

```
AfterToolCallResult:
    content?: List<TextContent | ImageContent>   # replaces content array in full
    details?: Any                                 # replaces details in full
    isError?: Boolean                             # replaces error flag
```

Merge semantics: each field is independent. If provided, it fully replaces the
corresponding field in the result. If omitted, the original value is kept. There
is no deep merge within content or details.

### ToolExecutionMode

```
ToolExecutionMode = "sequential" | "parallel"
```

### ToolCall (from AssistantMessage)

```
ToolCall:
    type: "toolCall"
    id: String              # unique identifier for this call
    name: String            # tool name
    arguments: Map<String, Any>
```

### Agent Events (Tool-Related)

```
tool_execution_start:
    toolCallId: String
    toolName: String
    args: Any

tool_execution_update:
    toolCallId: String
    toolName: String
    args: Any
    partialResult: Any

tool_execution_end:
    toolCallId: String
    toolName: String
    result: Any
    isError: Boolean
```

## Algorithm / Flow

### Sequential Tool Execution

```
function executeToolsSequential(assistantMessage, tools, config, context, signal):
    toolCalls = extractToolCalls(assistantMessage.content)
    results = empty list

    for each toolCall in toolCalls:
        result = processSingleToolCall(toolCall, assistantMessage, tools, config, context, signal)
        results.append(result)

    return results
```

### Parallel Tool Execution

```
function executeToolsParallel(assistantMessage, tools, config, context, signal):
    toolCalls = extractToolCalls(assistantMessage.content)

    # Phase 1: Sequential preflight (validate + beforeToolCall)
    preflightResults = empty list
    for each toolCall in toolCalls:
        preflight = preflightToolCall(toolCall, assistantMessage, tools, config, context, signal)
        preflightResults.append(preflight)

    # Phase 2: Concurrent execution for non-blocked tools
    executionPromises = empty list
    for each (toolCall, preflight) in zip(toolCalls, preflightResults):
        if preflight.blocked:
            executionPromises.append(resolvedPromise(preflight.errorResult))
        else:
            executionPromises.append(
                executeToolAsync(toolCall, preflight.tool, preflight.args, signal, config)
            )

    executionResults = await allPromises(executionPromises)

    # Phase 3: Sequential finalization (afterToolCall + emit) in source order
    finalResults = empty list
    for each (toolCall, preflight, execResult) in zip(toolCalls, preflightResults, executionResults):
        if preflight.blocked:
            finalResults.append(execResult)     # already an error result
        else:
            final = finalizeToolCall(toolCall, assistantMessage, preflight.args,
                                     execResult, config, context, signal)
            finalResults.append(final)

    return finalResults
```

### Single Tool Call Processing (Detail)

```
function processSingleToolCall(toolCall, assistantMessage, tools, config, context, signal):
    # Step 1: Find tool by name
    tool = findTool(tools, toolCall.name)
    if tool is null:
        return createErrorResult(toolCall, "Unknown tool: " + toolCall.name)

    # Step 2: Prepare arguments (optional compatibility shim)
    rawArgs = toolCall.arguments
    if tool.prepareArguments is defined:
        rawArgs = tool.prepareArguments(rawArgs)

    # Step 3: Validate against JSON Schema
    validationResult = validateJsonSchema(tool.parameters, rawArgs)
    if validationResult.hasErrors:
        return createErrorResult(toolCall, formatValidationErrors(validationResult.errors))

    validatedArgs = validationResult.value

    # Step 4: Emit tool_execution_start event
    emit({ type: "tool_execution_start", toolCallId: toolCall.id,
           toolName: toolCall.name, args: validatedArgs })

    # Step 5: Run beforeToolCall hook
    if config.beforeToolCall is defined:
        hookResult = await config.beforeToolCall(
            { assistantMessage, toolCall, args: validatedArgs, context },
            signal
        )
        if hookResult is not null and hookResult.block == true:
            reason = hookResult.reason or "Tool call blocked"
            errorResult = {
                content: [{ type: "text", text: reason }],
                details: null,
            }
            emit({ type: "tool_execution_end", toolCallId: toolCall.id,
                   toolName: toolCall.name, result: errorResult, isError: true })
            return createToolResultMessage(toolCall, errorResult, isError = true)

    # Step 6: Execute tool
    isError = false
    try:
        updateCallback = function(partial):
            emit({ type: "tool_execution_update", toolCallId: toolCall.id,
                   toolName: toolCall.name, args: validatedArgs, partialResult: partial })

        result = await tool.execute(toolCall.id, validatedArgs, signal, updateCallback)
    catch error:
        isError = true
        result = {
            content: [{ type: "text", text: error.message }],
            details: null,
        }

    # Step 7: Run afterToolCall hook
    if config.afterToolCall is defined:
        hookResult = await config.afterToolCall(
            { assistantMessage, toolCall, args: validatedArgs,
              result, isError, context },
            signal
        )
        if hookResult is not null:
            if hookResult.content is defined:
                result.content = hookResult.content
            if hookResult.details is defined:
                result.details = hookResult.details
            if hookResult.isError is defined:
                isError = hookResult.isError

    # Step 8: Emit tool_execution_end event
    emit({ type: "tool_execution_end", toolCallId: toolCall.id,
           toolName: toolCall.name, result: result, isError: isError })

    # Step 9: Create ToolResultMessage
    return createToolResultMessage(toolCall, result, isError)


function createToolResultMessage(toolCall, result, isError):
    return {
        role: "toolResult",
        toolCallId: toolCall.id,
        toolName: toolCall.name,
        content: result.content,
        details: result.details,
        isError: isError,
        timestamp: now(),
    }
```

### Schema Validation

```
function validateJsonSchema(schema, args):
    # Standard JSON Schema validation
    # Returns { hasErrors: Boolean, errors: List<String>, value: Any }
    #
    # The value may be coerced (e.g., string "42" to integer 42) depending
    # on the schema library's configuration.
    #
    # Common validation errors from LLM-generated arguments:
    # - Missing required fields
    # - Wrong types (string where number expected)
    # - Extra properties not in schema
    # - String not matching pattern/format
```

### Tool Factory Pattern

```
function createReadTool(workingDirectory, options?):
    operations = options?.operations or defaultFileSystemOperations

    return {
        name: "read",
        label: "read",
        description: "Read the contents of a file. Supports text and images...",
        parameters: {
            type: "object",
            properties: {
                path: { type: "string", description: "Path to file" },
                offset: { type: "integer", description: "Start line (1-indexed)" },
                limit: { type: "integer", description: "Max lines to read" },
            },
            required: ["path"],
        },
        execute: async function(toolCallId, params, signal, onUpdate):
            absolutePath = resolve(workingDirectory, params.path)
            await operations.access(absolutePath)

            if isImage(absolutePath):
                return readImage(absolutePath, operations)
            else:
                return readTextFile(absolutePath, params.offset, params.limit, operations)
    }
```

## Key Design Decisions

1. **Schema validation before execution.** LLMs frequently hallucinate argument
   names, types, or structures. Validating against JSON Schema before executing
   prevents malformed arguments from reaching tool code. The validation error
   is returned as a tool result so the LLM can self-correct.

2. **beforeToolCall enables permission systems.** The hook runs after validation
   (so the args are known to be well-formed) but before execution. This is the
   right place for user approval prompts, rate limiting, or sandboxing checks.
   Blocking returns an error result, not an exception.

3. **afterToolCall enables result transformation.** Extensions can modify tool
   results without wrapping the tool itself. Field-by-field merge (not deep
   merge) keeps the semantics simple and predictable.

4. **content vs details separation.** `content` goes to the LLM as part of the
   `ToolResultMessage`. `details` is for the UI (diffs, syntax-highlighted
   output, structured metadata). This prevents verbose UI data from consuming
   the model's context window.

5. **Factory pattern for tool creation.** `createReadTool(cwd)`,
   `createBashTool(cwd)`, etc. Each factory closes over the working directory
   and optional configuration. This allows multiple tool instances with
   different configurations (e.g., different working directories for different
   projects).

6. **Pluggable operations.** Tools like read and edit accept an `operations`
   parameter with functions for file I/O. The default is local filesystem, but
   tests can inject mocks, and remote agents can delegate to SSH/API calls.

7. **prepareArguments for backward compatibility.** When tool schemas evolve
   (e.g., renaming `file_path` to `path`), the shim can normalize old argument
   names before validation. This avoids breaking existing cached tool calls.

8. **Parallel execution preserves source order.** Even when tools run
   concurrently, results are emitted in the order they appeared in the
   assistant message. This ensures deterministic conversation history regardless
   of execution timing.

9. **Tools throw on failure, loop catches.** The execute function signals errors
   by throwing. The agent loop catches and converts to an error tool result.
   This keeps tool implementations simple -- they do not need to construct error
   result structs.

10. **Streaming partial results via callback.** Long-running tools (bash
    commands, large file reads) call the `onUpdate` callback with intermediate
    results. The loop forwards these as `tool_execution_update` events so the
    UI can show progress.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|---|---|
| Unknown tool name | Error tool result: "Unknown tool: {name}". LLM can self-correct. |
| Schema validation fails | Error tool result with formatted validation errors. LLM can retry with correct args. |
| Tool throws exception | Caught by loop. Error tool result with exception message. |
| Tool hangs indefinitely | AbortSignal passed to execute. Caller can abort after timeout. Tool must honor the signal. |
| beforeToolCall blocks | Error tool result with the hook's reason text. Tool never executes. |
| afterToolCall overrides isError to false | A tool that threw can be "forgiven" by the hook. Result is treated as success. |
| afterToolCall overrides content | Full replacement. Original content is discarded. |
| Multiple tool calls, one blocked | In parallel mode: blocked tool gets error result, others execute normally. |
| Tool call with empty arguments | Validated against schema. If schema has no required fields, this is valid. |
| Duplicate tool call IDs | Each ID maps to a separate ToolResultMessage. Duplicates would cause LLM confusion. Provider should ensure unique IDs. |
| Tool returns image content | Images are included in ToolResultMessage content. LLM providers that support vision can process them. |
| prepareArguments throws | Treated as validation failure. Error tool result emitted. |
| Concurrent file mutations | File mutation queue serializes writes to the same file. Multiple edit/write tool calls to the same file are sequenced even in parallel mode. |
| AbortSignal fires during beforeToolCall | Hook receives the signal and should return promptly. |
| Tool calls reference nonexistent files | Tool-specific error (e.g., "file not found"). Returned as error tool result. |
| Schema allows additional properties | LLM's extra arguments are silently accepted. Set `additionalProperties: false` to reject them. |

## Built-In Tool Examples

### read

```
Parameters:
    path: String (required)         # file path, relative or absolute
    offset?: Integer                # start line, 1-indexed
    limit?: Integer                 # max lines to read

Behavior:
    - Resolves path relative to working directory
    - Detects image files (jpg, png, gif, webp) and returns as ImageContent
    - For text files: reads content, applies offset/limit, truncates if needed
    - Truncation limits: 2000 lines or 256KB (whichever hit first)
    - Returns continuation hint with next offset when truncated

Result content: text with line-numbered output, or image attachment
Result details: truncation metadata (was it truncated, by lines or bytes)
```

### edit

```
Parameters:
    path: String (required)         # file path
    edits: List<{ oldText, newText }> (required)
        oldText: String             # exact text to find (must be unique in file)
        newText: String             # replacement text

Behavior:
    - Reads current file content
    - For each edit: finds exact match of oldText, replaces with newText
    - All edits matched against original content (not incrementally)
    - Validates no overlapping edits
    - Writes modified content back
    - Preserves original line endings (CRLF/LF)
    - Generates unified diff for details

Result content: success message or error description
Result details: unified diff string, first changed line number
```

### write

```
Parameters:
    path: String (required)         # file path
    content: String (required)      # complete file content

Behavior:
    - Creates parent directories if needed
    - Writes content to file (creates or overwrites)
    - For new files: straightforward write
    - For existing files: generates diff against previous content

Result content: success message with file path
Result details: diff if file existed, file size
```

### bash

```
Parameters:
    command: String (required)      # shell command to execute
    timeout?: Integer               # timeout in milliseconds

Behavior:
    - Spawns shell process with working directory
    - Streams stdout/stderr to onUpdate callback
    - Applies timeout (kills process if exceeded)
    - Captures exit code
    - Truncates output if too large
    - Supports AbortSignal for cancellation

Result content: command output (stdout + stderr interleaved)
Result details: exit code, whether output was truncated, full output path
```

### grep

```
Parameters:
    pattern: String (required)      # regex pattern
    path?: String                   # directory or file to search
    glob?: String                   # file pattern filter (e.g., "*.ts")

Behavior:
    - Runs ripgrep (or equivalent) with the pattern
    - Respects .gitignore by default
    - Returns matching lines with file paths and line numbers
    - Truncates if too many matches

Result content: matching lines formatted as "file:line: content"
```

### find

```
Parameters:
    pattern: String (required)      # glob pattern for file names
    path?: String                   # directory to search in

Behavior:
    - Finds files matching the glob pattern
    - Respects .gitignore
    - Returns list of matching file paths
    - Sorted by modification time (most recent first)
```

### ls

```
Parameters:
    path?: String                   # directory to list (defaults to cwd)

Behavior:
    - Lists directory contents
    - Shows file/directory indicators
    - Respects .gitignore for large directories
```

## Testing Strategy

### Unit Tests for Tool Execution Pipeline

```
test "successful tool call lifecycle":
    tool = createMockTool("test", schema = { x: Integer })
    tool.execute = async (id, params, signal, onUpdate) =>
        return { content: [{ type: "text", text: "result:" + params.x }], details: null }

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "test", arguments: { x: 42 } }
    ])

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results.length == 1
    assert results[0].content[0].text == "result:42"
    assert results[0].isError == false

test "schema validation rejects bad arguments":
    tool = createMockTool("test", schema = { x: { type: Integer, required: true } })

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "test", arguments: { y: "wrong" } }
    ])

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results[0].isError == true
    assert results[0].content[0].text contains "validation"

test "unknown tool returns error result":
    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "nonexistent", arguments: {} }
    ])

    results = await executeToolsSequential(assistantMsg, [], config, context, null)
    assert results[0].isError == true
    assert results[0].content[0].text contains "Unknown tool"

test "tool exception becomes error result":
    tool = createMockTool("test")
    tool.execute = async () => throw Error("disk full")

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "test", arguments: {} }
    ])

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results[0].isError == true
    assert results[0].content[0].text contains "disk full"

test "beforeToolCall blocks execution":
    tool = createMockTool("dangerous")
    executeCalled = false
    tool.execute = async () => { executeCalled = true; return mockResult() }

    config.beforeToolCall = async (ctx, signal) =>
        return { block: true, reason: "User denied permission" }

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "dangerous", arguments: {} }
    ])

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert executeCalled == false
    assert results[0].isError == true
    assert results[0].content[0].text contains "User denied permission"

test "afterToolCall overrides result content":
    tool = createMockTool("test")
    tool.execute = async () => {
        return { content: [{ type: "text", text: "original" }], details: null }
    }

    config.afterToolCall = async (ctx, signal) =>
        return { content: [{ type: "text", text: "modified" }] }

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results[0].content[0].text == "modified"

test "afterToolCall partial override preserves unset fields":
    tool = createMockTool("test")
    tool.execute = async () => {
        return { content: [{ type: "text", text: "original" }], details: { key: "value" } }
    }

    config.afterToolCall = async (ctx, signal) =>
        return { isError: true }    # only override isError, keep content and details

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results[0].content[0].text == "original"     # preserved
    assert results[0].details.key == "value"             # preserved
    assert results[0].isError == true                    # overridden
```

### Tests for Parallel Execution

```
test "parallel mode executes non-blocked tools concurrently":
    executionOrder = []
    toolA = createMockTool("a")
    toolA.execute = async () => { executionOrder.append("a-start"); sleep(50); executionOrder.append("a-end"); return mockResult() }
    toolB = createMockTool("b")
    toolB.execute = async () => { executionOrder.append("b-start"); sleep(10); executionOrder.append("b-end"); return mockResult() }

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "a", arguments: {} },
        { id: "tc2", name: "b", arguments: {} },
    ])

    results = await executeToolsParallel(assistantMsg, [toolA, toolB], config, context, null)
    # Both started before either finished
    assert executionOrder[0] == "a-start" or executionOrder[0] == "b-start"
    assert executionOrder[1] == "a-start" or executionOrder[1] == "b-start"
    # Results in source order regardless of completion order
    assert results[0].toolName == "a"
    assert results[1].toolName == "b"

test "parallel mode: blocked tool does not delay others":
    config.beforeToolCall = async (ctx) =>
        if ctx.toolCall.name == "blocked":
            return { block: true, reason: "no" }
        return null

    toolBlocked = createMockTool("blocked")
    toolFast = createMockTool("fast")
    toolFast.execute = async () => return mockResult("fast-result")

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "blocked", arguments: {} },
        { id: "tc2", name: "fast", arguments: {} },
    ])

    results = await executeToolsParallel(assistantMsg, [toolBlocked, toolFast], config, context, null)
    assert results[0].isError == true
    assert results[1].isError == false
    assert results[1].content[0].text == "fast-result"
```

### Tests for Streaming Updates

```
test "onUpdate callback emits tool_execution_update events":
    events = []
    captureEmit = (event) => events.append(event)

    tool = createMockTool("slow")
    tool.execute = async (id, params, signal, onUpdate) =>
        onUpdate({ content: [{ type: "text", text: "25%" }], details: { progress: 0.25 } })
        onUpdate({ content: [{ type: "text", text: "75%" }], details: { progress: 0.75 } })
        return { content: [{ type: "text", text: "done" }], details: { progress: 1.0 } }

    await executeWithEventCapture(tool, assistantMsg, captureEmit)
    updateEvents = events.filter(e => e.type == "tool_execution_update")
    assert updateEvents.length == 2
    assert updateEvents[0].partialResult.details.progress == 0.25
```

### Tests for prepareArguments

```
test "prepareArguments normalizes legacy parameter names":
    tool = createMockTool("read", schema = { path: String })
    tool.prepareArguments = (args) =>
        if args.file_path and not args.path:
            return { path: args.file_path }
        return args

    assistantMsg = createAssistantMessage(toolCalls = [
        { id: "tc1", name: "read", arguments: { file_path: "/tmp/test.txt" } }
    ])

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results[0].isError == false   # validation passed after normalization

test "prepareArguments exception becomes error result":
    tool = createMockTool("test")
    tool.prepareArguments = (args) => throw Error("cannot normalize")

    results = await executeToolsSequential(assistantMsg, [tool], config, context, null)
    assert results[0].isError == true
```

## References

- Source: `packages/agent/src/types.ts` -- AgentTool, AgentToolResult,
  AgentToolUpdateCallback, BeforeToolCallContext, BeforeToolCallResult,
  AfterToolCallContext, AfterToolCallResult, ToolExecutionMode, AgentEvent
- Source: `packages/coding-agent/src/core/tools/` -- tool implementations
  (read.ts, edit.ts, write.ts, bash.ts, grep.ts, find.ts, ls.ts)
- Source: `packages/coding-agent/src/core/tools/index.ts` -- tool registry,
  factory functions, tool collections
- Depends on: `message-transformation-pipeline.skill.md`
- Pattern: Strategy pattern (each tool is a strategy)
- Pattern: Factory pattern (createReadTool, createBashTool, etc.)
- Pattern: Hook/Interceptor pattern (beforeToolCall, afterToolCall)
- Pattern: JSON Schema validation as a contract boundary
