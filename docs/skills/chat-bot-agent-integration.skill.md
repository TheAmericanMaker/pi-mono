# Chat Bot Agent Integration

## Purpose

Integrate an LLM agent into a chat platform (Slack, Discord, etc.) with
per-channel context management, persistent memory, sandbox execution, event
scheduling, and message backfilling.  The bot maintains full conversation
history per channel, manages context windows for long-lived channels, and
executes tools inside isolated sandboxes.

## Dependencies

- **agent-loop-with-tool-execution.skill.md** -- the core agent loop that
  drives LLM inference and tool execution.
- **context-window-management.skill.md** -- compaction and context budget
  management for long-running channels.
- **session-persistence-with-branching.skill.md** -- JSONL-based session
  storage used for per-channel context files.

## Core Concepts

### Per-channel isolation

Each chat channel (or DM) gets its own directory containing:
- `log.jsonl` -- full message history (human-readable, for grep)
- `context.jsonl` -- structured messages for the LLM (same format as the
  agent's session manager)
- `MEMORY.md` -- persistent knowledge for this channel
- `attachments/` -- downloaded files shared by users
- `scratch/` -- working directory for the bot's file operations
- `skills/` -- channel-specific reusable scripts

### Message log vs. context

The **log** is a permanent, append-only record of all messages (user messages
and bot responses, but not tool call details).  The **context** is a curated
subset fed to the LLM, including tool calls and results.  The log is used for
backfilling and history search; the context is used for agent inference.

### Memory system

Two levels of persistent memory:
- **Global MEMORY.md** (workspace level) -- knowledge shared across all
  channels (user preferences, project info, installed skills).
- **Channel MEMORY.md** -- channel-specific decisions, ongoing work, etc.

Both are injected into the system prompt.  The agent can read and write memory
files using its file tools.

### Sandbox execution

Tools (bash, read, write, edit) execute inside a sandbox -- either a Docker
container or the host machine.  The sandbox provides isolation and persistence
across conversations.

### Event scheduling

The bot supports scheduled events that trigger future actions:
- **Immediate** -- fires as soon as detected (for webhooks/scripts).
- **One-shot** -- fires once at a specific time (reminders).
- **Periodic** -- fires on a cron schedule (recurring tasks).

Events are JSON files in a watched directory.  When an event fires, it becomes
a message to the channel's agent.

## Data Structures

```
// ---- Channel store ----

record ChannelStoreConfig:
    workingDir : string
    botToken   : string            // for authenticated file downloads

class ChannelStore:
    workingDir      : string
    pendingDownloads : list<PendingDownload>
    recentlyLogged   : map<string, integer>   // "channelId:ts" -> epoch, for dedup

    getChannelDir(channelId) -> string
    generateLocalFilename(originalName, timestamp) -> string
    processAttachments(channelId, files, timestamp) -> list<Attachment>

record Attachment:
    original : string              // original filename
    local    : string              // relative path in workspace

// ---- Message types ----

record LoggedMessage:
    date        : string           // ISO 8601
    ts          : string           // platform timestamp (Slack ts)
    user        : string           // user ID or "bot"
    userName    : string?
    displayName : string?
    text        : string
    attachments : list<Attachment>
    isBot       : bool

record PendingMessage:
    userName    : string
    text        : string
    attachments : list<{ local: string }>
    timestamp   : integer

// ---- Slack context (per-message) ----

record SlackContext:
    message:
        text        : string
        rawText     : string
        user        : string
        userName    : string?
        channel     : string
        ts          : string
        attachments : list<{ local: string }>
    channelName : string?
    channels    : list<ChannelInfo>
    users       : list<UserInfo>

    respond(text, shouldLog?)         -> Promise<void>
    replaceMessage(text)              -> Promise<void>
    respondInThread(text)             -> Promise<void>
    setTyping(isTyping)               -> Promise<void>
    uploadFile(filePath, title?)      -> Promise<void>
    setWorking(working)               -> Promise<void>
    deleteMessage()                   -> Promise<void>

record ChannelInfo:
    id   : string
    name : string

record UserInfo:
    id          : string
    userName    : string
    displayName : string

// ---- Event types ----

record ImmediateEvent:
    type      = "immediate"
    channelId : string
    text      : string

record OneShotEvent:
    type      = "one-shot"
    channelId : string
    text      : string
    at        : string             // ISO 8601 with timezone offset

record PeriodicEvent:
    type      = "periodic"
    channelId : string
    text      : string
    schedule  : string             // cron expression
    timezone  : string             // IANA timezone name

union MomEvent = ImmediateEvent | OneShotEvent | PeriodicEvent

// ---- Agent runner ----

interface AgentRunner:
    run(ctx, store, pendingMessages?) -> { stopReason, errorMessage? }
    abort() -> void

// ---- Sandbox ----

union SandboxConfig:
    { type: "host" }
    | { type: "docker", container: string }

interface Executor:
    exec(command, options?) -> ExecResult
    getWorkspacePath(hostPath) -> string

record ExecResult:
    stdout : string
    stderr : string
    exitCode : integer
```

## Algorithm / Flow

### 1. Architecture overview

```
Chat Platform (Slack / Discord)
    |
    v
Message Router (per-channel queue)
    |
    v
Channel Handler
    |-- Log message to log.jsonl
    |-- Sync missed messages to session
    |-- Build system prompt (with memory)
    |-- Run agent loop
    |-- Save context
    |-- Send response
    |
    v
Agent (LLM + tools in sandbox)
    |
    v
Response Formatter -> Chat Platform
```

### 2. Message handling

```
function handleSlackEvent(event, slack, handler):
    channelId = event.channel

    // Per-channel sequential queue prevents concurrent agent runs
    channelQueue = getOrCreateQueue(channelId)

    channelQueue.enqueue(async () =>
        // Check if agent is already running for this channel
        if handler.isRunning(channelId):
            // Queue the message -- it will be picked up as a pending message
            queuePendingMessage(channelId, event)
            return

        store = getChannelStore()
        channelDir = store.getChannelDir(channelId)

        // Process attachments (download files)
        attachments = store.processAttachments(channelId, event.files, event.ts)

        // Log user message
        logUserMessage(channelDir, event, attachments)

        // Build context for agent
        ctx = buildSlackContext(event, slack, channelDir, attachments)

        // Run agent
        runner = getOrCreateRunner(sandboxConfig, channelId, channelDir)
        result = await runner.run(ctx, store)

        if result.errorMessage:
            ctx.respond("Error: " + result.errorMessage)
    )
```

### 3. Agent runner creation (one per channel, persistent)

```
function createRunner(sandboxConfig, channelId, channelDir):
    executor = createExecutor(sandboxConfig)
    workspacePath = executor.getWorkspacePath(parentDir(channelDir))

    // Create tools (bash, read, write, edit, attach)
    tools = createMomTools(executor)

    // Load memory and skills
    memory = getMemory(channelDir)
    skills = loadSkills(channelDir, workspacePath)
    systemPrompt = buildSystemPrompt(workspacePath, channelId, memory,
                                      sandboxConfig, channels, users, skills)

    // Create session manager backed by context.jsonl
    contextFile = join(channelDir, "context.jsonl")
    sessionManager = SessionManager.open(contextFile, channelDir)

    // Create agent
    agent = Agent({
        systemPrompt, model, tools,
        getApiKey: () => getApiKey("anthropic")
    })

    // Load existing context messages
    loaded = sessionManager.buildSessionContext()
    if loaded.messages is not empty:
        agent.state.messages = loaded.messages

    // Subscribe to agent events (tool labels, results -> Slack)
    subscribeToAgentEvents(agent, runState)

    return AgentRunner { run, abort }
```

### 4. Agent run (per message)

```
function run(ctx, store, pendingMessages?):
    // 1. Refresh system prompt with latest memory/channels/users/skills
    memory = getMemory(channelDir)
    skills = loadSkills(channelDir, workspacePath)
    agent.state.systemPrompt = buildSystemPrompt(workspacePath, channelId,
                                                  memory, sandboxConfig,
                                                  ctx.channels, ctx.users, skills)

    // 2. Sync missed messages from log to session
    syncLogToSessionManager(sessionManager, channelDir, ctx.message.ts)

    // 3. Add pending messages (queued while agent was busy)
    if pendingMessages is not empty:
        for msg in pendingMessages:
            promptText = formatPendingMessage(msg)
            agent.addUserMessage(promptText, msg.attachments)

    // 4. Format current message
    promptText = formatUserMessage(ctx.message)

    // 5. Show typing indicator
    ctx.setTyping(true)

    // 6. Run agent loop
    try:
        result = await agent.prompt(promptText, images)

        // 7. Send response
        responseText = extractAssistantResponse(result)
        if responseText == "[SILENT]":
            ctx.deleteMessage()    // periodic event with nothing to report
        else:
            ctx.respond(responseText)

        // 8. Save context
        syncAgentToSession(sessionManager, agent)

        return { stopReason: result.stopReason }
    catch error:
        return { stopReason: "error", errorMessage: error.message }
    finally:
        ctx.setTyping(false)
```

### 5. Message backfilling (sync log to session)

```
function syncLogToSessionManager(sessionManager, channelDir, excludeTs):
    logFile = join(channelDir, "log.jsonl")
    if not exists(logFile): return 0

    // Build set of existing message content in session (for dedup)
    existingMessages = set()
    for entry in sessionManager.getEntries():
        if entry.type == "message" and entry.message.role == "user":
            normalized = normalizeMessageForDedup(entry.message.content)
            existingMessages.add(normalized)

    // Read log and find new user messages
    logLines = readFile(logFile).split("\n")
    newMessages = []

    for line in logLines:
        logMsg = parseJSON(line)
        if logMsg.isBot: continue           // skip bot messages
        if logMsg.ts == excludeTs: continue  // skip current message

        normalized = normalizeForDedup(logMsg)
        if normalized in existingMessages: continue

        newMessages.append(formatAsUserMessage(logMsg))

    // Add new messages to session in chronological order
    sort(newMessages, by: timestamp)
    for msg in newMessages:
        sessionManager.addMessage(msg)

    return len(newMessages)
```

### 6. Memory loading

```
function getMemory(channelDir):
    parts = []

    // Global memory (workspace level)
    globalPath = join(parentDir(channelDir), "MEMORY.md")
    if exists(globalPath):
        content = readFile(globalPath).trim()
        if content is not empty:
            parts.append("### Global Workspace Memory\n" + content)

    // Channel-specific memory
    channelPath = join(channelDir, "MEMORY.md")
    if exists(channelPath):
        content = readFile(channelPath).trim()
        if content is not empty:
            parts.append("### Channel-Specific Memory\n" + content)

    if parts is empty:
        return "(no working memory yet)"

    return join(parts, "\n\n")
```

### 7. Event scheduling

```
class EventsWatcher:
    eventsDir  : string
    timers     : map<string, Timer>      // one-shot timers
    crons      : map<string, CronJob>    // periodic cron jobs
    knownFiles : set<string>
    watcher    : FSWatcher

    function start():
        ensureDir(eventsDir)
        scanExisting()                   // process files already present
        watcher = watch(eventsDir, onFileChange)

    function scanExisting():
        for file in listDir(eventsDir):
            if file.endsWith(".json"):
                processEventFile(file)

    function onFileChange(filename):
        debounce(filename, 100ms, () =>
            processEventFile(filename)
        )

    function processEventFile(filename):
        path = join(eventsDir, filename)
        if not exists(path):
            // File deleted -- cancel any scheduled timer/cron
            cancelScheduled(filename)
            return

        event = parseJSON(readFile(path))

        switch event.type:
            case "immediate":
                fireEvent(filename, event)
                deleteFile(path)           // auto-cleanup

            case "one-shot":
                delay = parseISO(event.at) - currentTimeMillis()
                if delay <= 0:
                    fireEvent(filename, event)
                    deleteFile(path)
                else:
                    timers[filename] = setTimeout(delay, () =>
                        fireEvent(filename, event)
                        deleteFile(path)
                    )

            case "periodic":
                if crons.has(filename):
                    crons[filename].stop()
                crons[filename] = Cron(event.schedule, { timezone: event.timezone }, () =>
                    fireEvent(filename, event)
                )

    function fireEvent(filename, event):
        // Format event as a message to the channel
        prefix = "[EVENT:{filename}:{event.type}]"
        if event.type == "one-shot":
            prefix = "[EVENT:{filename}:one-shot:{event.at}]"
        elif event.type == "periodic":
            prefix = "[EVENT:{filename}:periodic:{event.schedule}]"

        message = prefix + " " + event.text

        // Dispatch as a chat event to the channel
        slackEvent = {
            type: "event",
            channel: event.channelId,
            text: message,
            user: "scheduler"
        }
        handler.handleEvent(slackEvent, slack, isEvent=true)

    function cancelScheduled(filename):
        if timers.has(filename):
            clearTimeout(timers[filename])
            timers.delete(filename)
        if crons.has(filename):
            crons[filename].stop()
            crons.delete(filename)
        knownFiles.delete(filename)

    function stop():
        watcher?.close()
        for timer in timers.values(): clearTimeout(timer)
        for cron in crons.values(): cron.stop()
```

### 8. Sandbox execution

```
class DockerExecutor implements Executor:
    container : string

    function exec(command, options?):
        args = ["docker", "exec", "-i", container, "sh", "-c", command]
        result = spawnProcess(args, timeout=options?.timeout, signal=options?.signal)
        return ExecResult(result.stdout, result.stderr, result.exitCode)

    function getWorkspacePath(hostPath):
        return "/workspace"        // fixed mount point inside container

class HostExecutor implements Executor:
    function exec(command, options?):
        result = spawnProcess(["sh", "-c", command], timeout=options?.timeout)
        return ExecResult(result.stdout, result.stderr, result.exitCode)

    function getWorkspacePath(hostPath):
        return hostPath            // no translation needed
```

### 9. Per-channel queue (sequential processing)

```
class ChannelQueue:
    queue      : list<function>
    processing : bool

    function enqueue(work):
        queue.append(work)
        processNext()

    function processNext():
        if processing or queue is empty: return
        processing = true

        work = queue.removeFirst()
        try:
            await work()
        catch error:
            logError(error)
        finally:
            processing = false
            processNext()          // process next item if any
```

### 10. Thread integration

```
function handleToolResult(ctx, toolName, args, result, isError, duration):
    // Post tool label to main channel (brief)
    label = args.label or toolName
    ctx.respond("-> " + label, shouldLog=false)

    // Post full details to reply thread (verbose)
    threadMessage = formatToolResultForThread(toolName, label, args, result, duration, isError)
    ctx.respondInThread(threadMessage)

    if isError:
        ctx.respond("Error: " + truncate(result, 200))
```

### 11. Silent completion for periodic events

```
function handleAgentResponse(ctx, responseText):
    if responseText.trim() == "[SILENT]":
        // Periodic event with nothing to report
        ctx.deleteMessage()        // remove the status/typing message
        return

    ctx.respond(responseText)
```

## Key Design Decisions

1. **Per-channel sequential queue.**  Only one agent run per channel at a time.
   Additional messages are queued and delivered as "pending messages" when the
   agent finishes.  This prevents race conditions on shared context files.

2. **Separate log and context files.**  The log is permanent and human-readable
   (for grep-based history search).  The context is managed by the session
   manager and subject to compaction.  This separation lets the bot search old
   history even after context has been compacted.

3. **Persistent runners (one per channel).**  The agent runner is created once
   per channel and reused across messages.  This avoids re-loading context from
   disk on every message and keeps the agent's in-memory state warm.

4. **System prompt refreshed per run.**  Memory, skills, and user/channel lists
   can change between messages.  The system prompt is rebuilt from current state
   at the start of each agent run.

5. **Event files as scheduling mechanism.**  Using JSON files in a watched
   directory is simple, inspectable, and debuggable.  The agent creates events
   by writing files with its bash/write tools.  No database required.

6. **Silent completion ([SILENT]).**  Periodic events often find nothing to
   report.  The `[SILENT]` convention lets the agent suppress output without
   cluttering the channel.

7. **Thread-based detail offloading.**  Tool call details go into reply threads
   rather than the main channel.  This keeps the channel clean while preserving
   full debugging information.

8. **Docker sandbox with persistent container.**  The container stays running
   across messages and sessions.  This means installed packages and
   configuration persist, and the agent can maintain a working environment.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| Bot receives messages while agent is running | Messages are queued; delivered as pending messages on next run |
| Bot restarts and misses messages | `syncLogToSessionManager` backfills from log.jsonl; deduplication by content hash prevents double-processing |
| Context grows too large | Session manager's compaction kicks in; old messages are summarized |
| Docker container not running | Startup validation fails with clear error; agent doesn't start |
| Network error downloading attachment | Download is queued and retried; agent sees the local path but file may be missing until download completes |
| Corrupt event JSON | `parseJSON` fails; event file is skipped with error log |
| Event file deleted while timer pending | Timer fires but file is gone; event is skipped |
| Multiple immediate events created simultaneously | File watcher debounces (100ms); each file processed independently |
| Maximum 5 events queued | System prompt instructs agent to limit events; no hard enforcement (soft limit) |
| MEMORY.md read error | Error logged; memory section shows "(no working memory yet)" |
| Agent response exceeds Slack message limit | Caller should truncate or split; platform-specific formatting |
| Duplicate message detection | `recentlyLogged` map tracks recent timestamps; duplicates within 60s are skipped |
| OAuth token expires during agent run | Auth storage handles refresh transparently (see multi-provider-auth-with-oauth) |

## Testing Strategy

### Unit tests

```
test "message backfill deduplicates correctly":
    sessionManager = createSessionManager()
    // Add a message manually
    sessionManager.addMessage({ role: "user", content: "[user]: hello" })
    // Log the same message
    writeLog(channelDir, { text: "hello", userName: "user" })
    synced = syncLogToSessionManager(sessionManager, channelDir)
    assert synced == 0    // no new messages

test "memory loading combines global and channel":
    writeFile(workspaceDir + "/MEMORY.md", "global info")
    writeFile(channelDir + "/MEMORY.md", "channel info")
    memory = getMemory(channelDir)
    assert "global info" in memory
    assert "channel info" in memory

test "memory loading returns placeholder when empty":
    memory = getMemory(emptyChannelDir)
    assert memory == "(no working memory yet)"

test "channel queue processes sequentially":
    queue = ChannelQueue()
    order = []
    queue.enqueue(async () => sleep(10); order.append(1))
    queue.enqueue(async () => order.append(2))
    await waitForQueue(queue)
    assert order == [1, 2]

test "silent response deletes status message":
    ctx = mockSlackContext()
    handleAgentResponse(ctx, "[SILENT]")
    assert ctx.deleteMessageCalled == true
    assert ctx.respondCalled == false
```

### Event scheduler tests

```
test "immediate event fires and auto-deletes":
    writeEvent("test.json", { type: "immediate", channelId: "C1", text: "hello" })
    watcher.processEventFile("test.json")
    assert eventFired("test.json")
    assert not exists(eventsDir + "/test.json")

test "one-shot event fires at scheduled time":
    future = currentTime() + 100ms
    writeEvent("reminder.json", { type: "one-shot", channelId: "C1",
                                    text: "remind", at: toISO(future) })
    watcher.processEventFile("reminder.json")
    sleep(150ms)
    assert eventFired("reminder.json")

test "periodic event fires on schedule":
    writeEvent("daily.json", { type: "periodic", channelId: "C1",
                                text: "check", schedule: "* * * * *",
                                timezone: "UTC" })
    watcher.processEventFile("daily.json")
    // Verify cron job was created
    assert watcher.crons.has("daily.json")

test "deleted event file cancels scheduled timer":
    writeEvent("cancel-me.json", { type: "one-shot", channelId: "C1",
                                    text: "x", at: futureTime() })
    watcher.processEventFile("cancel-me.json")
    deleteFile(eventsDir + "/cancel-me.json")
    watcher.onFileChange("cancel-me.json")
    assert not watcher.timers.has("cancel-me.json")
```

### Sandbox tests

```
test "docker executor runs command in container":
    executor = DockerExecutor("test-container")
    result = executor.exec("echo hello")
    assert result.stdout.trim() == "hello"
    assert result.exitCode == 0

test "host executor runs command directly":
    executor = HostExecutor()
    result = executor.exec("echo hello")
    assert result.stdout.trim() == "hello"

test "docker workspace path translation":
    executor = DockerExecutor("test-container")
    assert executor.getWorkspacePath("/home/user/data") == "/workspace"
```

## References

- Slack Socket Mode: https://api.slack.com/apis/socket-mode
- Cron expression format: https://crontab.guru/
- Source: `packages/mom/src/agent.ts`
- Source: `packages/mom/src/slack.ts`
- Source: `packages/mom/src/context.ts`
- Source: `packages/mom/src/store.ts`
- Source: `packages/mom/src/events.ts`
- Source: `packages/mom/src/sandbox.ts`
