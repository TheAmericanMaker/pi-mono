# LLM Streaming Event Protocol

## Purpose

Define a typed, async-iterable event protocol for streaming LLM responses from
provider to consumer. The protocol decouples producers (HTTP/SSE/WebSocket
adapters) from consumers (UI renderers, loggers, agent loops) via an in-memory
push/pull queue that delivers strongly-typed discriminated-union events.

The two main goals are:

1. **Incremental delivery** -- consumers can render partial text, thinking, and
   tool-call content the moment each delta arrives, without waiting for the full
   response.
2. **Typed completion** -- every stream eventually resolves to a final result
   (the completed `AssistantMessage`) accessible via a promise, so callers that
   only care about the end state can simply `await stream.result()`.

## Dependencies

None. This is a foundational primitive with zero external dependencies. It uses
only language-level async iteration and promise facilities.

## Core Concepts

### Discriminated Union Events

Every event carries a `type` field that acts as the discriminant. Consumers
switch on `type` to handle each case. Events fall into three categories:

- **Lifecycle events**: `start`, `done`, `error` -- one `start` per stream,
  exactly one terminal event (`done` or `error`).
- **Content delta events**: grouped in start/delta/end triples for text,
  thinking, and tool calls. Each triple shares a `contentIndex` that identifies
  which content block in the accumulating message is being updated.
- **Terminal events**: `done` carries the final successful message; `error`
  carries the final message with an error stop-reason and error text.

### Push/Pull Async Queue

The `EventStream` is the central data structure. It bridges a synchronous
producer (the provider adapter calling `push()`) with an asynchronous consumer
(the agent loop or UI iterating with `for await`). The queue is unbounded on the
producer side and single-consumer on the pull side.

### Partial Message Accumulation

Every non-terminal event includes a `partial` field containing the
`AssistantMessage` as it has been accumulated so far. This lets consumers render
the latest snapshot without maintaining their own accumulator.

## Data Structures

### EventStream<T, R>

```
class EventStream<T, R>:
    # Internal state
    queue: List<T>                          # buffered events not yet consumed
    waiters: List<Callback<IteratorResult<T>>>  # blocked consumers
    done: Boolean                           # true after terminal event or end()
    finalResultPromise: Promise<R>          # resolved on completion
    resolveFinalResult: Callback<R>         # resolver for the promise

    # Constructor parameters (injected strategies)
    isComplete: Function(T) -> Boolean      # predicate: is this a terminal event?
    extractResult: Function(T) -> R         # extract the final result from terminal event

    # Public API
    push(event: T) -> Void
    end(result?: R) -> Void
    result() -> Promise<R>
    [AsyncIterator]() -> AsyncIterator<T>
```

### AssistantMessageEventStream

A specialization of `EventStream<AssistantMessageEvent, AssistantMessage>` with
built-in completion predicate and result extractor:

```
class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage>:
    constructor():
        super(
            isComplete = (event) => event.type == "done" OR event.type == "error",
            extractResult = (event) =>
                if event.type == "done": return event.message
                if event.type == "error": return event.error
                else: throw Error("Unexpected terminal event type")
        )
```

### AssistantMessageEvent (Discriminated Union)

```
AssistantMessageEvent = one of:
    # Lifecycle
    { type: "start",          partial: AssistantMessage }

    # Text content streaming
    { type: "text_start",     contentIndex: Integer, partial: AssistantMessage }
    { type: "text_delta",     contentIndex: Integer, delta: String, partial: AssistantMessage }
    { type: "text_end",       contentIndex: Integer, content: String, partial: AssistantMessage }

    # Thinking/reasoning content streaming
    { type: "thinking_start", contentIndex: Integer, partial: AssistantMessage }
    { type: "thinking_delta", contentIndex: Integer, delta: String, partial: AssistantMessage }
    { type: "thinking_end",   contentIndex: Integer, content: String, partial: AssistantMessage }

    # Tool call streaming
    { type: "toolcall_start", contentIndex: Integer, partial: AssistantMessage }
    { type: "toolcall_delta", contentIndex: Integer, delta: String, partial: AssistantMessage }
    { type: "toolcall_end",   contentIndex: Integer, toolCall: ToolCall, partial: AssistantMessage }

    # Terminal events (exactly one per stream)
    { type: "done",  reason: "stop" | "length" | "toolUse", message: AssistantMessage }
    { type: "error", reason: "aborted" | "error",           error: AssistantMessage }
```

### AssistantMessage

```
AssistantMessage:
    role: "assistant"
    content: List<TextContent | ThinkingContent | ToolCall>
    api: String              # e.g. "anthropic-messages", "openai-completions"
    provider: String         # e.g. "anthropic", "openai"
    model: String            # model identifier
    responseId?: String      # provider-specific response ID
    usage: Usage
    stopReason: StopReason
    errorMessage?: String    # present when stopReason is "error" or "aborted"
    timestamp: Integer       # Unix milliseconds
```

### Usage

```
Usage:
    input: Integer           # input tokens
    output: Integer          # output tokens
    cacheRead: Integer       # tokens served from cache
    cacheWrite: Integer      # tokens written to cache
    totalTokens: Integer     # input + output
    cost:
        input: Float         # dollar cost of input tokens
        output: Float        # dollar cost of output tokens
        cacheRead: Float
        cacheWrite: Float
        total: Float         # sum of all cost components
```

### StopReason

```
StopReason = "stop" | "length" | "toolUse" | "error" | "aborted"
```

- `stop` -- model finished naturally
- `length` -- max tokens reached
- `toolUse` -- model wants to call tools
- `error` -- unrecoverable provider/network error
- `aborted` -- cancelled via AbortSignal

### Content Block Types

```
TextContent:
    type: "text"
    text: String
    textSignature?: String

ThinkingContent:
    type: "thinking"
    thinking: String
    thinkingSignature?: String
    redacted?: Boolean

ToolCall:
    type: "toolCall"
    id: String
    name: String
    arguments: Map<String, Any>
```

## Algorithm / Flow

### push(event)

```
function push(event):
    if done:
        return                          # silently ignore post-completion pushes

    if isComplete(event):
        done = true
        resolveFinalResult(extractResult(event))

    if waiters is not empty:
        waiter = waiters.removeFirst()
        waiter.resolve({ value: event, done: false })
    else:
        queue.append(event)
```

Key detail: even terminal events are delivered to the consumer via the queue (or
directly to a waiter). The `done` flag only prevents *future* pushes. The
consumer still receives the terminal event as a yielded value.

### end(result?)

```
function end(result?):
    done = true
    if result is not undefined:
        resolveFinalResult(result)

    # Wake all blocked consumers with a done signal
    while waiters is not empty:
        waiter = waiters.removeFirst()
        waiter.resolve({ value: undefined, done: true })
```

This is for abnormal termination when the producer cannot push a proper terminal
event (e.g., adapter crash). Normal streams should terminate via a `done` or
`error` event through `push()`.

### Async iteration (for-await consumer)

```
async generator [AsyncIterator]():
    loop forever:
        if queue is not empty:
            yield queue.removeFirst()
        else if done:
            return                      # stream exhausted
        else:
            # Park until producer delivers
            iterResult = await new Promise(resolve => waiters.append(resolve))
            if iterResult.done:
                return
            yield iterResult.value
```

### result()

```
function result():
    return finalResultPromise
```

Callers can `await stream.result()` without iterating. This is useful for
fire-and-forget streaming where only the final message matters.

### Typical Producer Flow

```
stream = new AssistantMessageEventStream()

# Emit lifecycle start
partial = createEmptyAssistantMessage(model, provider, api)
stream.push({ type: "start", partial: partial })

# For each SSE chunk from provider:
for chunk in providerSSEStream:
    # Accumulate into partial message
    if chunk is text delta:
        idx = getOrCreateTextBlock(partial, chunk.index)
        partial.content[idx].text += chunk.text
        stream.push({ type: "text_delta", contentIndex: idx, delta: chunk.text, partial: partial })

    else if chunk is tool call delta:
        idx = getOrCreateToolCallBlock(partial, chunk.index)
        # Tool call arguments arrive as JSON string fragments
        accumulatedJson += chunk.arguments
        stream.push({ type: "toolcall_delta", contentIndex: idx, delta: chunk.arguments, partial: partial })

    else if chunk is final:
        partial.usage = chunk.usage
        partial.stopReason = mapStopReason(chunk.finishReason)
        stream.push({ type: "done", reason: partial.stopReason, message: partial })

# On network error:
partial.stopReason = "error"
partial.errorMessage = error.message
stream.push({ type: "error", reason: "error", error: partial })
```

### Typical Consumer Flow

```
stream = provider.stream(model, context, options)

for await event in stream:
    switch event.type:
        case "start":
            ui.showPlaceholder()

        case "text_delta":
            ui.appendText(event.delta)

        case "thinking_delta":
            ui.appendThinking(event.delta)

        case "toolcall_end":
            agent.queueToolExecution(event.toolCall)

        case "done":
            ui.finalize(event.message)

        case "error":
            ui.showError(event.error.errorMessage)

finalMessage = await stream.result()
```

## Key Design Decisions

1. **Errors in-band, not thrown.** Provider stream functions must never throw.
   All failures (network, auth, rate limit) are encoded as `error` events pushed
   into the stream. This means consumers have exactly one code path for both
   success and failure -- the iteration loop.

2. **Single consumer.** The queue assumes one iterator. Multiple concurrent
   iterators would race for events. If fan-out is needed, build a separate
   multiplexer on top.

3. **Terminal event is yielded, not swallowed.** When `isComplete` returns true,
   the event is still delivered to the consumer. The `done` flag only prevents
   subsequent pushes. This ensures the consumer sees the `done`/`error` event
   with its payload.

4. **Separate `result()` promise.** Some callers want the final
   `AssistantMessage` without iterating (e.g., simple request-response usage).
   The `finalResultPromise` is resolved from the terminal event regardless of
   whether anyone is iterating.

5. **Content index for multi-block messages.** A single response can contain
   interleaved text, thinking, and tool-call blocks. The `contentIndex` ties
   deltas to the correct block in the accumulating content array.

6. **Partial message on every event.** Carrying the full partial snapshot avoids
   forcing consumers to maintain their own accumulator. The tradeoff is slightly
   larger event objects, but in practice the partial is a single shared reference.

7. **Factory function for extensions.** `createAssistantMessageEventStream()` is
   exported for use by extension/plugin providers that need to create streams
   without importing the class directly.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|---|---|
| `push()` after `done` | Silently ignored. No error, no delivery. |
| `end()` without prior terminal event | Wakes all blocked consumers with `done: true`. `finalResultPromise` resolves with explicitly passed result or remains pending. |
| `end()` called twice | Safe. Second call is a no-op (waiters already drained). |
| Consumer iterates after stream is complete | Drains any remaining queued events, then returns immediately. |
| No consumer ever iterates | Events queue in memory. `result()` still resolves. |
| AbortSignal fires mid-stream | Producer should push an `error` event with reason `"aborted"` and stop pushing. |
| Provider returns empty response | `start` event followed immediately by `done` with an empty content array. |
| Tool call JSON arrives in fragments | `toolcall_delta` events carry raw JSON string fragments. The producer must accumulate and parse the complete JSON before emitting `toolcall_end` with the parsed `ToolCall`. |
| Out-of-order content indices | Indices reference positions in the content array. Providers must ensure indices are monotonically assigned. Gaps or reuse would corrupt the partial message. |
| Thinking content redacted by safety filters | `thinking_end` event carries the content with `redacted: true`. The opaque signature is preserved for multi-turn continuity. |

## Testing Strategy

### Unit Tests for EventStream

```
test "push delivers to waiting consumer":
    stream = new EventStream(isComplete = alwaysFalse, extractResult = identity)
    # Start consumer in background
    consumer = asyncCollect(stream, maxEvents = 1)
    # Push after consumer is waiting
    stream.push("event1")
    assert consumer.result == ["event1"]

test "push queues when no consumer":
    stream = new EventStream(...)
    stream.push("event1")
    stream.push("event2")
    result = asyncCollect(stream, maxEvents = 2)
    assert result == ["event1", "event2"]

test "terminal event resolves finalResultPromise":
    stream = new AssistantMessageEventStream()
    msg = createAssistantMessage(stopReason = "stop")
    stream.push({ type: "done", reason: "stop", message: msg })
    assert await stream.result() == msg

test "error event resolves finalResultPromise with error message":
    stream = new AssistantMessageEventStream()
    msg = createAssistantMessage(stopReason = "error", errorMessage = "rate limited")
    stream.push({ type: "error", reason: "error", error: msg })
    result = await stream.result()
    assert result.stopReason == "error"
    assert result.errorMessage == "rate limited"

test "push after done is ignored":
    stream = new AssistantMessageEventStream()
    stream.push({ type: "done", ... })
    stream.push({ type: "text_delta", ... })   # should be silently dropped
    events = asyncCollectAll(stream)
    assert events.length == 1
    assert events[0].type == "done"

test "end() wakes blocked consumers":
    stream = new EventStream(...)
    consumer = asyncCollectAll(stream)       # blocks waiting for events
    stream.end()
    assert consumer.result == []             # empty, consumer saw done signal

test "iteration drains queue then returns on done":
    stream = new EventStream(...)
    stream.push("a")
    stream.push("b")
    stream.end()
    result = asyncCollectAll(stream)
    assert result == ["a", "b"]
```

### Integration Tests

```
test "full stream lifecycle with text content":
    stream = new AssistantMessageEventStream()
    # Simulate producer
    async produce():
        partial = emptyMessage()
        stream.push({ type: "start", partial: partial })
        partial.content = [{ type: "text", text: "Hello" }]
        stream.push({ type: "text_start", contentIndex: 0, partial: partial })
        stream.push({ type: "text_delta", contentIndex: 0, delta: " world", partial: partial })
        stream.push({ type: "text_end", contentIndex: 0, content: "Hello world", partial: partial })
        partial.stopReason = "stop"
        stream.push({ type: "done", reason: "stop", message: partial })

    produce()

    events = asyncCollectAll(stream)
    assert events.length == 5
    assert events[0].type == "start"
    assert events[4].type == "done"
    assert (await stream.result()).content[0].text == "Hello world"

test "abort signal produces error event":
    controller = new AbortController()
    stream = simulateProvider(model, context, { signal: controller.signal })
    controller.abort()
    result = await stream.result()
    assert result.stopReason == "aborted"
```

### Property-Based Tests

```
test "every stream terminates with exactly one done or error":
    for randomEventSequence in generateRandomSequences():
        stream = new AssistantMessageEventStream()
        for event in randomEventSequence:
            stream.push(event)
        terminalEvents = filter(asyncCollectAll(stream), e => e.type in ["done", "error"])
        assert terminalEvents.length <= 1

test "result() always resolves if a terminal event is pushed":
    stream = new AssistantMessageEventStream()
    stream.push(randomTerminalEvent())
    result = await withTimeout(stream.result(), 1000ms)
    assert result is not timeout
```

## References

- Source: `packages/ai/src/utils/event-stream.ts` -- EventStream and
  AssistantMessageEventStream implementation
- Source: `packages/ai/src/types.ts` -- AssistantMessageEvent union,
  AssistantMessage, Usage, StopReason, content block types
- Pattern: CSP (Communicating Sequential Processes) -- push/pull channels
- Pattern: Async iterators (Symbol.asyncIterator protocol)
