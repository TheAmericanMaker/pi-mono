# Agent Loop with Tool Execution

## Purpose

Implement the core agent loop pattern used in LLM-powered agents: send a prompt to the model, receive a response that may contain tool calls, execute those tools, feed results back, and repeat until the model stops requesting tools. This pattern also supports mid-run steering (injecting user guidance while tools execute) and follow-up messages (continuing after the agent would otherwise stop).

## Dependencies

- [LLM Streaming Event Protocol](./llm-streaming-event-protocol.skill.md) — event stream consumed during LLM calls
- [Message Transformation Pipeline](./message-transformation-pipeline.skill.md) — converting internal messages to LLM format
- [Tool System Design](./tool-system-design.skill.md) — tool definitions, validation, and execution

## Core Concepts

### The Loop

The agent loop is a two-level loop:

1. **Inner loop** — processes tool calls and steering messages until the model stops requesting tools
2. **Outer loop** — checks for follow-up messages after the inner loop exits; if any exist, they become pending messages and the inner loop restarts

### Turns

A "turn" is one cycle of: process pending messages → get LLM response → execute tool calls. The loop emits `turn_start` and `turn_end` events for each turn.

### Steering Messages

Messages injected into the conversation mid-run. Checked after each turn's tool calls complete. Allows the user to guide the agent while it's working (e.g., "skip that file" or "use a different approach").

### Follow-Up Messages

Messages that wait until the agent would otherwise stop, then continue the conversation. Unlike steering messages (which inject during active processing), follow-ups restart the loop.

### Agent Wrapper

A stateful wrapper around the loop that provides:
- State management (system prompt, model, tools, messages, thinking level)
- Subscription system for events
- Methods: `prompt()`, `continue()`, `steer()`, `followUp()`
- Abort control via AbortSignal
- Queue modes: "all" (process all queued messages) or "one-at-a-time"

## Data Structures

```
AgentState {
    systemPrompt: string
    model: Model
    thinkingLevel: enum { off, minimal, low, medium, high, xhigh }
    tools: list of AgentTool
    messages: list of AgentMessage      // Full conversation history
    isStreaming: bool                    // Read-only, true while processing
    streamingMessage: AgentMessage?     // Partial message during streaming
    pendingToolCalls: set of string     // Tool call IDs currently executing
    errorMessage: string?              // From most recent failed turn
}

AgentContext {
    systemPrompt: string
    model: Model
    thinkingLevel: ThinkingLevel
    tools: list of AgentTool
    messages: list of AgentMessage
}

AgentLoopConfig {
    model: Model
    convertToLlm: fn(AgentMessage[]) → Message[]
    transformContext: fn(AgentMessage[], AbortSignal?) → AgentMessage[]  // optional
    getApiKey: fn(provider) → string?                                    // optional
    getSteeringMessages: fn() → AgentMessage[]                          // optional
    getFollowUpMessages: fn() → AgentMessage[]                          // optional
    toolExecution: "sequential" | "parallel"                             // default: parallel
    beforeToolCall: fn(BeforeToolCallContext, AbortSignal?) → BeforeToolCallResult?
    afterToolCall: fn(AfterToolCallContext, AbortSignal?) → AfterToolCallResult?
    // Plus streaming options: temperature, maxTokens, signal, etc.
}

AgentEvent = discriminated union {
    | { type: "agent_start" }
    | { type: "agent_end", messages: AgentMessage[] }
    | { type: "turn_start" }
    | { type: "turn_end", message: AgentMessage, toolResults: ToolResultMessage[] }
    | { type: "message_start", message: AgentMessage }
    | { type: "message_update", message: AgentMessage, event: AssistantMessageEvent }
    | { type: "message_end", message: AgentMessage }
    | { type: "tool_execution_start", toolCallId, toolName, args }
    | { type: "tool_execution_update", toolCallId, toolName, args, partialResult }
    | { type: "tool_execution_end", toolCallId, toolName, result, isError }
}
```

## Algorithm / Flow

### Main Loop

```
function runLoop(context, newMessages, config, signal, emit):
    firstTurn = true
    pendingMessages = config.getSteeringMessages() or []

    // === OUTER LOOP: handles follow-up messages ===
    while true:
        hasMoreToolCalls = true

        // === INNER LOOP: process tool calls + steering ===
        while hasMoreToolCalls or pendingMessages is not empty:

            if not firstTurn:
                emit({ type: "turn_start" })
            else:
                firstTurn = false

            // 1. Inject pending steering messages
            if pendingMessages is not empty:
                for msg in pendingMessages:
                    emit({ type: "message_start", message: msg })
                    emit({ type: "message_end", message: msg })
                    context.messages.append(msg)
                    newMessages.append(msg)
                pendingMessages = []

            // 2. Stream assistant response from LLM
            assistantMsg = streamAssistantResponse(context, config, signal, emit)
            newMessages.append(assistantMsg)

            // 3. Check for error/abort
            if assistantMsg.stopReason in ["error", "aborted"]:
                emit({ type: "turn_end", message: assistantMsg, toolResults: [] })
                emit({ type: "agent_end", messages: newMessages })
                return

            // 4. Extract and execute tool calls
            toolCalls = assistantMsg.content.filter(c => c.type == "toolCall")
            hasMoreToolCalls = len(toolCalls) > 0

            toolResults = []
            if hasMoreToolCalls:
                toolResults = executeToolCalls(context, assistantMsg, config, signal, emit)
                for result in toolResults:
                    context.messages.append(result)
                    newMessages.append(result)

            emit({ type: "turn_end", message: assistantMsg, toolResults: toolResults })

            // 5. Check for new steering messages
            pendingMessages = config.getSteeringMessages() or []

        // === Inner loop exited: agent would stop ===

        // 6. Check for follow-up messages
        followUps = config.getFollowUpMessages() or []
        if followUps is not empty:
            pendingMessages = followUps
            continue  // Restart inner loop

        break  // No follow-ups, done

    emit({ type: "agent_end", messages: newMessages })
```

### Streaming an Assistant Response

```
function streamAssistantResponse(context, config, signal, emit):
    // 1. Transform context (optional pruning/injection at AgentMessage level)
    messages = context.messages
    if config.transformContext:
        messages = config.transformContext(messages, signal)

    // 2. Convert to LLM-compatible format
    llmMessages = config.convertToLlm(messages)

    // 3. Build LLM context
    llmContext = {
        systemPrompt: context.systemPrompt,
        messages: llmMessages,
        tools: context.tools.map(t => { name: t.name, description: t.description, parameters: t.parameters })
    }

    // 4. Resolve API key dynamically
    apiKey = config.getApiKey(context.model.provider) if config.getApiKey else null

    // 5. Call stream function
    eventStream = streamSimple(context.model, llmContext, {
        reasoning: context.thinkingLevel,
        apiKey: apiKey,
        signal: signal,
        ...config.streamOptions
    })

    // 6. Iterate events and emit
    finalMessage = null
    for event in eventStream:
        if event.type == "start":
            emit({ type: "message_start", message: event.partial })
        elif event.type in ["text_delta", "thinking_delta", "toolcall_delta"]:
            emit({ type: "message_update", message: event.partial, event: event })
        elif event.type == "done":
            finalMessage = event.message
            emit({ type: "message_end", message: event.message })
        elif event.type == "error":
            finalMessage = event.error
            emit({ type: "message_end", message: event.error })

    context.messages.append(finalMessage)
    return finalMessage
```

### Tool Execution (Parallel Mode)

```
function executeToolCalls(context, assistantMsg, config, signal, emit):
    toolCalls = assistantMsg.content.filter(c => c.type == "toolCall")
    results = new Map()  // Preserve ordering

    // Phase 1: Preflight (sequential — order matters for beforeToolCall)
    approved = []
    for tc in toolCalls:
        tool = findTool(context.tools, tc.name)
        if not tool:
            results[tc.id] = makeErrorResult(tc, "Unknown tool: " + tc.name)
            continue

        // Validate arguments
        args = tool.prepareArguments(tc.arguments) if tool.prepareArguments else tc.arguments
        validationError = validateJsonSchema(args, tool.parameters)
        if validationError:
            results[tc.id] = makeErrorResult(tc, "Invalid arguments: " + validationError)
            continue

        // Before hook (can block)
        hookResult = config.beforeToolCall({ assistantMessage: assistantMsg, toolCall: tc, args, context }, signal)
        if hookResult and hookResult.block:
            results[tc.id] = makeErrorResult(tc, hookResult.reason or "Tool call blocked")
            continue

        approved.append({ tc, tool, args })

    // Phase 2: Execute approved tools concurrently
    emit({ type: "tool_execution_start", ... }) for each approved tool

    futures = []
    for { tc, tool, args } in approved:
        future = async:
            result = tool.execute(tc.id, args, signal, onUpdate)
            return { tc, tool, args, result }
        futures.append(future)

    executedResults = awaitAll(futures)

    // Phase 3: Finalize in source order
    for { tc, tool, args, result } in executedResults:
        // After hook (can modify result)
        hookResult = config.afterToolCall({
            assistantMessage: assistantMsg, toolCall: tc, args,
            result, isError: false, context
        }, signal)

        if hookResult:
            if hookResult.content: result.content = hookResult.content
            if hookResult.details: result.details = hookResult.details
            if hookResult.isError != null: isError = hookResult.isError

        results[tc.id] = makeToolResultMessage(tc, result)
        emit({ type: "tool_execution_end", toolCallId: tc.id, ... })

    // Return results in original tool call order
    return [results[tc.id] for tc in toolCalls]
```

### Agent Class

```
class Agent:
    state: MutableAgentState
    subscribers: list of fn(AgentEvent)
    activeRun: { abortController, promise }?
    steeringQueue: list of AgentMessage
    followUpQueue: list of AgentMessage

    constructor(options):
        state = createMutableState(options.initialState)
        // Wire up getSteeringMessages and getFollowUpMessages to drain queues

    function prompt(message: AgentMessage):
        // Add message, start loop, return event stream
        assert not state.isStreaming
        state.isStreaming = true
        abortController = new AbortController()
        stream = agentLoop([message], buildContext(), buildConfig(), abortController.signal)
        // Forward events to subscribers
        // On agent_end: state.isStreaming = false

    function steer(message: AgentMessage):
        // Add to steering queue (processed during next tool execution gap)
        steeringQueue.append(message)

    function followUp(message: AgentMessage):
        // Add to follow-up queue (processed after agent would stop)
        followUpQueue.append(message)

    function abort():
        activeRun?.abortController.abort()

    function subscribe(callback: fn(AgentEvent)) → unsubscribe:
        subscribers.append(callback)
        return () => subscribers.remove(callback)
```

## Key Design Decisions

1. **Two-level loop** — Steering happens within the active processing cycle; follow-ups restart it. This separation gives fine-grained control over message timing.

2. **Event-driven, not callback-driven** — Consumers iterate an event stream rather than registering callbacks. This makes it easy to build different UIs (TUI, web, tests) on the same loop.

3. **Context transform separate from LLM conversion** — `transformContext` works at the AgentMessage level (pruning, injecting), while `convertToLlm` handles format conversion. Two separate concerns, two separate hooks.

4. **Errors in stream, never thrown** — The stream function contract requires errors to be encoded as events, not exceptions. This ensures the loop always gets a clean `agent_end` event.

5. **Dynamic API key resolution** — The `getApiKey` callback is called per-LLM-call, not once at startup. This handles OAuth tokens that expire during long tool execution phases.

6. **Parallel tool execution with sequential preflight** — beforeToolCall hooks run sequentially (order matters for permission checks), but approved tools execute concurrently. Results are emitted in source order regardless of completion order.

## Edge Cases & Failure Modes

| Scenario | Handling |
|---|---|
| Abort during LLM streaming | Signal propagates to stream; returns message with stopReason "aborted" |
| Abort during tool execution | Signal passed to tool execute(); tool must honor it |
| Tool throws exception | Catch, wrap as error tool result, continue loop |
| LLM returns unknown tool name | Emit error tool result "Unknown tool", continue |
| LLM hallucinates invalid arguments | Schema validation catches it; error result sent back |
| Empty tool call list but stopReason="toolUse" | Treat as no tool calls (hasMoreToolCalls = false) |
| Steering message arrives during LLM streaming | Queued, processed after current turn's tool calls |
| Multiple follow-up messages | All processed as pending messages in next inner loop iteration |
| convertToLlm throws | Contract says it must not throw; if it does, loop breaks uncleanly |
| transformContext returns empty | LLM gets no messages; will likely return an error or empty response |
| Concurrent prompt() calls | Assert not streaming; second call should throw or queue |

## Testing Strategy

1. **Unit: Loop termination** — Verify loop exits when model returns stopReason "stop" with no tool calls
2. **Unit: Tool call cycle** — Model returns tool call → tool executes → result fed back → model responds
3. **Unit: Multi-turn** — Multiple rounds of tool calls before final response
4. **Unit: Parallel execution** — Two tool calls execute concurrently; results arrive in source order
5. **Unit: Sequential execution** — Same scenario but tools run one-at-a-time
6. **Unit: Steering injection** — Steering message injected between turns; appears in context
7. **Unit: Follow-up continuation** — Agent stops, follow-up message restarts loop
8. **Unit: Abort mid-stream** — Abort signal during LLM call; loop exits with "aborted"
9. **Unit: Abort mid-tool** — Abort signal during tool execution; loop exits cleanly
10. **Unit: beforeToolCall blocking** — Hook returns block; error result emitted, tool not executed
11. **Unit: afterToolCall modification** — Hook modifies result content; modified version emitted
12. **Unit: Error recovery** — Model returns error; loop exits with agent_end
13. **Integration: Event ordering** — Verify correct sequence: agent_start → turn_start → message_start → ... → turn_end → agent_end
14. **Integration: State management** — Agent.state reflects streaming status, pending tool calls during execution
15. **Faux provider** — Use a fake LLM provider that returns scripted responses for deterministic tests

## References

- `packages/agent/src/agent-loop.ts` — Main loop implementation (runLoop, streamAssistantResponse, executeToolCalls)
- `packages/agent/src/agent.ts` — Agent class with state management and subscription system
- `packages/agent/src/types.ts` — AgentState, AgentLoopConfig, AgentEvent, AgentTool, BeforeToolCallContext/Result, AfterToolCallContext/Result
- `packages/ai/src/types.ts` — StreamFunction contract, AssistantMessage, StopReason
- `packages/ai/src/utils/event-stream.ts` — EventStream class used by agentLoop return value
