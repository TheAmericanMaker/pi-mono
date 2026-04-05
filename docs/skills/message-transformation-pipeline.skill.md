# Message Transformation Pipeline

## Purpose

Maintain a rich internal message representation (AgentMessage) that supports
app-specific custom message types (bash executions, compaction summaries, branch
summaries, extension-injected messages) while ensuring only standard LLM-
compatible messages reach the model. The pipeline transforms messages through
two stages before each LLM call:

1. **transformContext** -- prune, inject, or reorder at the AgentMessage level
   (e.g., context window management, steering message injection).
2. **convertToLlm** -- filter out custom types, convert remaining messages to
   the three standard LLM roles (user, assistant, toolResult).

This separation means:

- The internal transcript is a complete audit log of everything that happened.
- The LLM only sees what it needs to see.
- Custom message types can carry UI-only data that never leaves the client.
- Adding new custom message types requires no changes to the LLM layer.

## Dependencies

None. This pattern is independent of the streaming protocol and provider
abstraction. It operates on message arrays and produces message arrays.

## Core Concepts

### Two-Tier Message Types

**LLM Messages** are the three roles every LLM understands:

- `UserMessage` -- user input (text and/or images)
- `AssistantMessage` -- model response (text, thinking, tool calls)
- `ToolResultMessage` -- result of a tool execution

**AgentMessages** are the union of LLM Messages plus custom app-specific
message types. The agent loop stores AgentMessages internally but converts
to LLM Messages before each API call.

### Declaration Merging for Extensibility

Custom message types are added via declaration merging on an initially empty
`CustomAgentMessages` interface. Each app or extension declares its custom
types, and the `AgentMessage` union automatically includes them:

```
# Base definition (in agent core)
interface CustomAgentMessages:
    # Empty by default

type AgentMessage = Message | CustomAgentMessages[values]

# App extends via declaration merging
extend CustomAgentMessages:
    bashExecution: BashExecutionMessage
    compactionSummary: CompactionSummaryMessage
    branchSummary: BranchSummaryMessage
    custom: CustomMessage
```

This approach means:

- The base agent package has no knowledge of app-specific types.
- Type safety is maintained -- exhaustive switches catch missing cases.
- Multiple extensions can each add their own types without conflicts.

### Two-Stage Transformation

The pipeline has two stages, both configured on the agent loop:

```
AgentMessage[] --[transformContext]--> AgentMessage[] --[convertToLlm]--> Message[]
```

**transformContext** (optional): operates on the full AgentMessage array.
Used for context window management (pruning old messages when tokens exceed
limits), injecting context from external sources, or any operation that
benefits from seeing the full custom message types.

**convertToLlm** (required): maps each AgentMessage to zero or more LLM
Messages. Custom types are either converted to a standard role (typically
`user`) or filtered out entirely. Standard LLM messages pass through
unchanged.

## Data Structures

### LLM Message Types

```
UserMessage:
    role: "user"
    content: String | List<TextContent | ImageContent>
    timestamp: Integer              # Unix milliseconds

AssistantMessage:
    role: "assistant"
    content: List<TextContent | ThinkingContent | ToolCall>
    api: String
    provider: String
    model: String
    responseId?: String
    usage: Usage
    stopReason: StopReason
    errorMessage?: String
    timestamp: Integer

ToolResultMessage:
    role: "toolResult"
    toolCallId: String
    toolName: String
    content: List<TextContent | ImageContent>
    details?: Any                   # arbitrary structured data for UI
    isError: Boolean
    timestamp: Integer

Message = UserMessage | AssistantMessage | ToolResultMessage
```

### Custom Message Types (App-Specific Examples)

```
BashExecutionMessage:
    role: "bashExecution"
    command: String
    output: String
    exitCode: Integer | null
    cancelled: Boolean
    truncated: Boolean
    fullOutputPath?: String
    timestamp: Integer
    excludeFromContext?: Boolean     # if true, skip during convertToLlm

CompactionSummaryMessage:
    role: "compactionSummary"
    summary: String
    tokensBefore: Integer
    timestamp: Integer

BranchSummaryMessage:
    role: "branchSummary"
    summary: String
    fromId: String
    timestamp: Integer

CustomMessage:
    role: "custom"
    customType: String              # extension-defined subtype
    content: String | List<TextContent | ImageContent>
    display: Boolean                # whether to show in UI
    details?: Any                   # extension-specific payload
    timestamp: Integer
```

### AgentMessage (Union Type)

```
AgentMessage = Message
             | BashExecutionMessage
             | CompactionSummaryMessage
             | BranchSummaryMessage
             | CustomMessage
```

### Pipeline Configuration

```
AgentLoopConfig:
    convertToLlm: Function(List<AgentMessage>) -> List<Message>
    transformContext?: Function(List<AgentMessage>, AbortSignal?) -> Promise<List<AgentMessage>>
    getSteeringMessages?: Function() -> Promise<List<AgentMessage>>
    getFollowUpMessages?: Function() -> Promise<List<AgentMessage>>
    # ... other config fields
```

## Algorithm / Flow

### Full Pipeline (Before Each LLM Call)

```
function prepareContext(config, agentMessages, signal):
    # Stage 1: Transform at AgentMessage level (optional)
    if config.transformContext is defined:
        messages = await config.transformContext(agentMessages, signal)
    else:
        messages = agentMessages

    # Stage 2: Convert to LLM-compatible messages
    llmMessages = await config.convertToLlm(messages)

    return llmMessages
```

### convertToLlm Implementation

```
function convertToLlm(messages: List<AgentMessage>) -> List<Message>:
    result = empty list

    for each message in messages:
        switch message.role:

            case "user":
            case "assistant":
            case "toolResult":
                # Standard LLM messages pass through unchanged
                result.append(message)

            case "bashExecution":
                if message.excludeFromContext:
                    continue                    # skip entirely
                text = formatBashExecution(message)
                result.append({
                    role: "user",
                    content: [{ type: "text", text: text }],
                    timestamp: message.timestamp,
                })

            case "compactionSummary":
                text = COMPACTION_PREFIX + message.summary + COMPACTION_SUFFIX
                result.append({
                    role: "user",
                    content: [{ type: "text", text: text }],
                    timestamp: message.timestamp,
                })

            case "branchSummary":
                text = BRANCH_PREFIX + message.summary + BRANCH_SUFFIX
                result.append({
                    role: "user",
                    content: [{ type: "text", text: text }],
                    timestamp: message.timestamp,
                })

            case "custom":
                content = message.content
                if content is String:
                    content = [{ type: "text", text: content }]
                result.append({
                    role: "user",
                    content: content,
                    timestamp: message.timestamp,
                })

            default:
                # Unknown custom type -- silently skip
                continue

    return result
```

### formatBashExecution

```
function formatBashExecution(msg: BashExecutionMessage) -> String:
    text = "Ran `" + msg.command + "`\n"

    if msg.output is not empty:
        text += "```\n" + msg.output + "\n```"
    else:
        text += "(no output)"

    if msg.cancelled:
        text += "\n\n(command cancelled)"
    else if msg.exitCode is not null and msg.exitCode != 0:
        text += "\n\nCommand exited with code " + msg.exitCode

    if msg.truncated and msg.fullOutputPath is not empty:
        text += "\n\n[Output truncated. Full output: " + msg.fullOutputPath + "]"

    return text
```

### Agent Loop Integration

```
function agentLoop(config, state):
    while not aborted:
        # Prepare context for LLM
        llmMessages = await prepareContext(config, state.messages, signal)
        context = {
            systemPrompt: state.systemPrompt,
            messages: llmMessages,
            tools: convertTools(state.tools),
        }

        # Call LLM
        stream = config.streamFn(config.model, context, streamOptions)

        # Process response
        assistantMessage = await processStream(stream, state)
        state.messages.append(assistantMessage)

        # Execute tool calls if any
        if assistantMessage has tool calls:
            toolResults = await executeToolCalls(assistantMessage, state)
            state.messages.appendAll(toolResults)

        # Check for steering messages (injected mid-run)
        if config.getSteeringMessages is defined:
            steeringMsgs = await config.getSteeringMessages()
            if steeringMsgs is not empty:
                state.messages.appendAll(steeringMsgs)
                continue        # another turn with injected context

        # Check for follow-up messages (after agent would stop)
        if assistantMessage.stopReason != "toolUse":
            if config.getFollowUpMessages is defined:
                followUps = await config.getFollowUpMessages()
                if followUps is not empty:
                    state.messages.appendAll(followUps)
                    continue    # another turn with follow-up context

        # No tool calls, no steering, no follow-ups -- done
        if assistantMessage.stopReason != "toolUse":
            break
```

### Context Window Management (transformContext Example)

```
function contextWindowTransform(messages, signal):
    tokenCount = estimateTokens(messages)

    if tokenCount <= MAX_CONTEXT_TOKENS:
        return messages

    # Strategy: compact older messages into a summary
    # Keep recent messages intact, summarize older ones
    recentCount = findRecentWindowSize(messages, MAX_CONTEXT_TOKENS * 0.6)
    olderMessages = messages[0 : length - recentCount]
    recentMessages = messages[length - recentCount : end]

    summary = await generateSummary(olderMessages, signal)
    compactionMsg = {
        role: "compactionSummary",
        summary: summary,
        tokensBefore: estimateTokens(olderMessages),
        timestamp: now(),
    }

    return [compactionMsg] + recentMessages
```

## Key Design Decisions

1. **Custom messages are first-class citizens.** They live in the same message
   array as LLM messages, maintaining chronological order. This is critical for
   accurate context -- a bash execution that happened between two assistant
   turns must appear in the right position.

2. **Declaration merging over union enums.** Instead of maintaining a central
   enum of all possible message roles, extensions add types via declaration
   merging. This means the agent core package compiles without knowing about
   bash executions or compaction summaries.

3. **convertToLlm must never throw.** The contract is explicit: return a safe
   fallback rather than throwing. An exception here would crash the agent loop
   without producing a normal event sequence. Unknown message types are silently
   filtered out.

4. **transformContext must never throw.** Same contract. If context pruning
   fails, return the original messages unchanged. The LLM call might fail due
   to context length, but that produces a normal error event.

5. **Custom messages can carry display-only data.** The `CustomMessage` type
   has a `display` flag and a `details` field. Even though the message is
   converted to a user message for the LLM, the details payload is only used
   by the UI. This avoids polluting the LLM context with rendering metadata.

6. **excludeFromContext for side-channel commands.** Bash executions prefixed
   with `!!` are marked `excludeFromContext: true`. They appear in the UI
   transcript but are invisible to the LLM. This lets users run commands that
   should not influence the model's behavior.

7. **Summaries use XML-like delimiters.** Compaction and branch summaries are
   wrapped in `<summary>...</summary>` tags. This gives the LLM a clear signal
   that this is condensed context, not literal user input.

8. **All custom messages convert to user role.** The LLM only understands user,
   assistant, and toolResult. Custom messages that should be visible to the LLM
   are mapped to user messages. This is the only safe choice -- injecting
   assistant messages would be putting words in the model's mouth, and
   toolResult requires a matching tool call.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|---|---|
| Unknown custom message role | Silently filtered out by `convertToLlm`. Does not throw. |
| Empty message array | Both stages return empty arrays. LLM receives only system prompt. |
| All messages filtered out | LLM receives only system prompt. May produce generic response. |
| transformContext returns more messages than input | Valid. It may inject context messages. |
| convertToLlm returns empty for non-empty input | Valid. All messages might be display-only custom types. |
| Bash execution with no output and exit code 0 | Converted to `"Ran \`cmd\`\n(no output)"`. |
| Bash execution with truncated output | Includes `[Output truncated. Full output: path]` note. |
| Custom message with image content | Images are included in the user message content array. |
| Compaction summary with empty summary text | Wraps empty string in delimiters. LLM sees `<summary>\n</summary>`. |
| transformContext is async and slow | Agent loop awaits it. AbortSignal is passed so it can be cancelled. |
| Multiple compaction summaries in transcript | Each appears as a separate user message. LLM sees progressive summaries. |
| Message with timestamp 0 | Valid. Timestamps are informational, not used for ordering. Array order is authoritative. |
| Declaration merging adds conflicting role name | Compile-time error. Two extensions cannot declare the same key in CustomAgentMessages. |

## Testing Strategy

### Unit Tests for convertToLlm

```
test "standard messages pass through unchanged":
    userMsg = { role: "user", content: "hello", timestamp: 1000 }
    assistantMsg = { role: "assistant", content: [...], ... }
    toolResult = { role: "toolResult", ... }
    result = convertToLlm([userMsg, assistantMsg, toolResult])
    assert result == [userMsg, assistantMsg, toolResult]

test "bash execution converts to user message":
    bash = {
        role: "bashExecution",
        command: "ls -la",
        output: "file1.txt\nfile2.txt",
        exitCode: 0,
        cancelled: false,
        truncated: false,
        timestamp: 1000,
    }
    result = convertToLlm([bash])
    assert result.length == 1
    assert result[0].role == "user"
    assert result[0].content[0].text contains "ls -la"
    assert result[0].content[0].text contains "file1.txt"

test "bash execution with excludeFromContext is filtered":
    bash = {
        role: "bashExecution",
        command: "secret-cmd",
        excludeFromContext: true,
        ...
    }
    result = convertToLlm([bash])
    assert result.length == 0

test "compaction summary wraps in delimiters":
    msg = { role: "compactionSummary", summary: "User asked about X", ... }
    result = convertToLlm([msg])
    assert result[0].content[0].text startsWith COMPACTION_PREFIX
    assert result[0].content[0].text endsWith COMPACTION_SUFFIX
    assert result[0].content[0].text contains "User asked about X"

test "branch summary wraps in delimiters":
    msg = { role: "branchSummary", summary: "Explored approach Y", ... }
    result = convertToLlm([msg])
    assert result[0].content[0].text contains "<summary>"
    assert result[0].content[0].text contains "Explored approach Y"

test "custom message with string content":
    msg = { role: "custom", customType: "note", content: "important note", ... }
    result = convertToLlm([msg])
    assert result[0].role == "user"
    assert result[0].content[0].text == "important note"

test "custom message with structured content":
    msg = {
        role: "custom",
        customType: "visual",
        content: [
            { type: "text", text: "See image:" },
            { type: "image", data: "base64...", mimeType: "image/png" },
        ],
        ...
    }
    result = convertToLlm([msg])
    assert result[0].content.length == 2
    assert result[0].content[1].type == "image"

test "mixed message types preserve order":
    messages = [
        { role: "user", content: "do something", ... },
        { role: "assistant", content: [...], ... },
        { role: "bashExecution", command: "ls", ... },
        { role: "compactionSummary", summary: "...", ... },
        { role: "user", content: "continue", ... },
    ]
    result = convertToLlm(messages)
    assert result.length == 5
    assert result[0].role == "user"
    assert result[1].role == "assistant"
    assert result[2].role == "user"    # converted bash
    assert result[3].role == "user"    # converted compaction
    assert result[4].role == "user"

test "empty input returns empty output":
    assert convertToLlm([]) == []
```

### Unit Tests for formatBashExecution

```
test "basic command with output":
    msg = { command: "echo hi", output: "hi", exitCode: 0, cancelled: false, truncated: false }
    text = formatBashExecution(msg)
    assert text == "Ran `echo hi`\n```\nhi\n```"

test "command with no output":
    msg = { command: "true", output: "", exitCode: 0, cancelled: false, truncated: false }
    text = formatBashExecution(msg)
    assert text contains "(no output)"

test "cancelled command":
    msg = { command: "sleep 999", output: "...", exitCode: null, cancelled: true, truncated: false }
    text = formatBashExecution(msg)
    assert text contains "(command cancelled)"

test "non-zero exit code":
    msg = { command: "false", output: "", exitCode: 1, cancelled: false, truncated: false }
    text = formatBashExecution(msg)
    assert text contains "exited with code 1"

test "truncated output with path":
    msg = { command: "cat big", output: "...", exitCode: 0, cancelled: false,
            truncated: true, fullOutputPath: "/tmp/out.txt" }
    text = formatBashExecution(msg)
    assert text contains "[Output truncated. Full output: /tmp/out.txt]"
```

### Integration Tests for Pipeline

```
test "full pipeline with transformContext and convertToLlm":
    config = {
        transformContext: function(messages, signal):
            # Simulate pruning: remove first message
            return messages[1:],
        convertToLlm: convertToLlm,
    }
    messages = [
        { role: "user", content: "old message", timestamp: 100 },
        { role: "user", content: "recent message", timestamp: 200 },
        { role: "bashExecution", command: "ls", output: "files", ... },
    ]
    result = await prepareContext(config, messages, null)
    assert result.length == 2          # first message pruned
    assert result[0].content == "recent message"
    assert result[1].role == "user"    # bash converted

test "transformContext failure returns original messages":
    config = {
        transformContext: function(messages, signal):
            # Simulate failure -- contract says return safe fallback
            return messages,           # fallback to original
        convertToLlm: convertToLlm,
    }
    # Pipeline should still work
    result = await prepareContext(config, messages, null)
    assert result is not empty
```

## References

- Source: `packages/agent/src/types.ts` -- AgentMessage, CustomAgentMessages,
  AgentLoopConfig (convertToLlm, transformContext), BeforeToolCallContext,
  AfterToolCallContext
- Source: `packages/coding-agent/src/core/messages.ts` -- BashExecutionMessage,
  CompactionSummaryMessage, BranchSummaryMessage, CustomMessage, convertToLlm
  implementation, declaration merging
- Pattern: Adapter pattern (custom messages adapted to LLM format)
- Pattern: Pipeline / Chain of Responsibility
- Pattern: Declaration merging (open/closed principle for type extension)
