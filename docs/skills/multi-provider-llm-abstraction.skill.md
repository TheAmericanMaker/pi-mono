# Multi-Provider LLM Abstraction

## Purpose

Provide a unified streaming API across many LLM providers (Anthropic, OpenAI,
Google, AWS Bedrock, Azure, Mistral, and arbitrary OpenAI-compatible endpoints).
Callers interact with a single `streamSimple(model, context, options)` function.
The system dispatches to the correct provider based on the model's `api` field,
hiding all provider-specific serialization, authentication, and transport
details.

This enables:

- **Swappable models** -- change the model and the right provider is used
  automatically.
- **Extension providers** -- third-party plugins can register new API types at
  runtime and unregister them on cleanup.
- **Lazy loading** -- provider implementations are loaded on first use, avoiding
  the startup cost of importing every SDK.

## Dependencies

- `llm-streaming-event-protocol.skill.md` -- all provider stream functions
  return an `AssistantMessageEventStream` and push events following that
  protocol.

## Core Concepts

### API Type vs Provider

These are deliberately separate dimensions:

- **API type** (`api`): the wire protocol / serialization format. Examples:
  `"openai-completions"`, `"anthropic-messages"`, `"google-generative-ai"`.
- **Provider** (`provider`): the hosting service. Examples: `"anthropic"`,
  `"openai"`, `"openrouter"`, `"azure-openai-responses"`.

Multiple providers can share the same API type. For example, OpenRouter,
Groq, Cerebras, and xAI all use the `"openai-completions"` API type with
different base URLs and compatibility flags. The registry maps API types to
implementations, not providers.

### Provider Registry

A global `Map<String, RegisteredApiProvider>` that maps API type strings to
provider objects. Each entry optionally carries a `sourceId` so that all
providers registered by a particular extension can be bulk-removed on extension
unload.

### Model as Dispatch Key

A `Model` struct carries its `api` field. When `streamSimple()` is called, the
registry looks up the provider by `model.api`. If the model's API is not
registered, the call fails. This means models are self-describing -- no external
routing table is needed.

### Compatibility Shims

For OpenAI-compatible providers, a `compat` struct on the model carries boolean
flags and enum fields that describe which features the endpoint actually
supports. This avoids URL-based heuristics and lets each model explicitly
declare its capabilities.

## Data Structures

### Model

```
Model<TApi>:
    id: String                  # e.g. "claude-sonnet-4-20250514"
    name: String                # human-readable display name
    api: TApi                   # API type string (dispatch key)
    provider: String            # hosting provider identifier
    baseUrl: String             # endpoint URL
    reasoning: Boolean          # whether model supports thinking/reasoning
    input: List<String>         # supported input modalities: "text", "image"
    cost:
        input: Float            # dollars per million input tokens
        output: Float           # dollars per million output tokens
        cacheRead: Float        # dollars per million cached-read tokens
        cacheWrite: Float       # dollars per million cached-write tokens
    contextWindow: Integer      # max context size in tokens
    maxTokens: Integer          # max output tokens
    headers?: Map<String, String>   # custom headers for this model
    compat?: CompatStruct       # provider-specific compatibility flags
```

### ApiProvider

```
ApiProvider<TApi, TOptions>:
    api: TApi                               # the API type this provider handles
    stream: Function(Model, Context, StreamOptions?) -> EventStream
    streamSimple: Function(Model, Context, SimpleStreamOptions?) -> EventStream
```

Internally the registry wraps these with type-checking shims that verify
`model.api == provider.api` at call time.

### RegisteredApiProvider

```
RegisteredApiProvider:
    provider: ApiProvider
    sourceId?: String           # identifies the extension that registered it
```

### Context

```
Context:
    systemPrompt?: String
    messages: List<Message>     # conversation history (user, assistant, toolResult)
    tools?: List<Tool>          # available tools with JSON Schema parameters
```

### StreamOptions

```
StreamOptions:
    temperature?: Float
    maxTokens?: Integer
    signal?: AbortSignal
    apiKey?: String
    transport?: "sse" | "websocket" | "auto"
    cacheRetention?: "none" | "short" | "long"
    sessionId?: String
    onPayload?: Function(payload, model) -> payload | undefined
    headers?: Map<String, String>
    maxRetryDelayMs?: Integer       # cap on server-requested retry delay
    metadata?: Map<String, Any>     # provider-specific metadata (e.g. user_id)
```

### SimpleStreamOptions

```
SimpleStreamOptions extends StreamOptions:
    reasoning?: ThinkingLevel       # "minimal" | "low" | "medium" | "high" | "xhigh"
    thinkingBudgets?: ThinkingBudgets
```

### ThinkingBudgets

```
ThinkingBudgets:
    minimal?: Integer       # token budget for minimal thinking
    low?: Integer
    medium?: Integer
    high?: Integer
```

### KnownApi (Enumeration)

```
KnownApi:
    "openai-completions"
    "openai-responses"
    "azure-openai-responses"
    "openai-codex-responses"
    "anthropic-messages"
    "bedrock-converse-stream"
    "google-generative-ai"
    "google-gemini-cli"
    "google-vertex"
    "mistral-conversations"
```

The `Api` type is `KnownApi | String`, allowing arbitrary custom API types.

### KnownProvider (Enumeration)

```
KnownProvider:
    "anthropic"
    "openai"
    "google"
    "google-gemini-cli"
    "google-antigravity"
    "google-vertex"
    "amazon-bedrock"
    "azure-openai-responses"
    "openai-codex"
    "github-copilot"
    "xai"
    "groq"
    "cerebras"
    "openrouter"
    "vercel-ai-gateway"
    "mistral"
    "minimax"
    "minimax-cn"
    "huggingface"
    "zai"
    "opencode"
    "opencode-go"
    "kimi-coding"
```

### OpenAICompletionsCompat

Compatibility flags for OpenAI-compatible endpoints. Each flag overrides
auto-detection that would otherwise be based on the base URL.

```
OpenAICompletionsCompat:
    supportsStore?: Boolean
    supportsDeveloperRole?: Boolean
    supportsReasoningEffort?: Boolean
    reasoningEffortMap?: Map<ThinkingLevel, String>
    supportsUsageInStreaming?: Boolean
    maxTokensField?: "max_completion_tokens" | "max_tokens"
    requiresToolResultName?: Boolean
    requiresAssistantAfterToolResult?: Boolean
    requiresThinkingAsText?: Boolean
    thinkingFormat?: "openai" | "openrouter" | "zai" | "qwen" | "qwen-chat-template"
    openRouterRouting?: { only?: List<String>, order?: List<String> }
    vercelGatewayRouting?: { only?: List<String>, order?: List<String> }
    zaiToolStream?: Boolean
    supportsStrictMode?: Boolean
```

## Algorithm / Flow

### Registration

```
global registry: Map<String, RegisteredApiProvider> = empty map

function registerApiProvider(provider, sourceId?):
    wrappedProvider = {
        api: provider.api,
        stream: wrapWithApiCheck(provider.api, provider.stream),
        streamSimple: wrapWithApiCheck(provider.api, provider.streamSimple),
    }
    registry.set(provider.api, { provider: wrappedProvider, sourceId: sourceId })

function wrapWithApiCheck(expectedApi, streamFn):
    return function(model, context, options):
        if model.api != expectedApi:
            throw Error("Mismatched api: got {model.api}, expected {expectedApi}")
        return streamFn(model, context, options)
```

### Lookup

```
function getApiProvider(api):
    entry = registry.get(api)
    if entry is null:
        return null
    return entry.provider

function getApiProviders():
    return registry.values().map(entry => entry.provider)
```

### Unregistration (Extension Cleanup)

```
function unregisterApiProviders(sourceId):
    for each (api, entry) in registry:
        if entry.sourceId == sourceId:
            registry.delete(api)

function clearApiProviders():
    registry.clear()
```

### Dispatching a Request

```
function streamSimple(model, context, options):
    provider = getApiProvider(model.api)
    if provider is null:
        # Return an error stream rather than throwing
        stream = new AssistantMessageEventStream()
        errorMsg = createErrorAssistantMessage("No provider for api: " + model.api)
        stream.push({ type: "error", reason: "error", error: errorMsg })
        return stream

    return provider.streamSimple(model, context, options)
```

### API Key Resolution Chain

When a provider needs an API key, it resolves through this chain (first
non-empty value wins):

```
function resolveApiKey(model, options, config):
    # 1. Explicit option (highest priority)
    if options.apiKey is not empty:
        return options.apiKey

    # 2. Dynamic callback (for short-lived tokens like OAuth)
    if config.getApiKey is defined:
        key = await config.getApiKey(model.provider)
        if key is not empty:
            return key

    # 3. Environment variable (convention: PROVIDER_API_KEY)
    envKey = getEnvVar(toUpperSnakeCase(model.provider) + "_API_KEY")
    if envKey is not empty:
        return envKey

    # 4. Stored credentials (OS keychain, config file, etc.)
    stored = loadStoredCredential(model.provider)
    if stored is not empty:
        return stored

    return null   # provider will likely fail with auth error
```

### Lazy Loading Provider Modules

```
# Providers are registered at startup but their heavy SDK imports
# happen inside the stream function itself.

function createAnthropicProvider():
    return {
        api: "anthropic-messages",
        stream: function(model, context, options):
            # Dynamic import -- only loads the Anthropic SDK when first called
            anthropicModule = dynamicImport("./providers/anthropic")
            return anthropicModule.stream(model, context, options),
        streamSimple: function(model, context, options):
            anthropicModule = dynamicImport("./providers/anthropic")
            return anthropicModule.streamSimple(model, context, options),
    }
```

## Key Design Decisions

1. **Registry keyed by API type, not provider.** Multiple providers (OpenRouter,
   Groq, xAI) share the `"openai-completions"` API type. They are the *same
   wire protocol* with different base URLs and compat flags. The model's
   `baseUrl` and `compat` fields handle the differences. Only truly different
   wire protocols get separate registry entries.

2. **Errors encoded in the stream, never thrown.** The contract for all stream
   functions is: return an `AssistantMessageEventStream`, encode failures as
   events. This is enforced by the type system (return type is not a Promise
   that could reject) and lets consumers use a single iteration loop for both
   success and error paths.

3. **Transport abstraction.** Most providers use SSE (Server-Sent Events), but
   some (OpenAI Codex) use WebSocket. The `transport` option lets the caller
   request a specific transport. Providers that do not support the requested
   transport silently ignore it and use their default.

4. **Compatibility flags over URL heuristics.** Early versions detected provider
   capabilities by pattern-matching the base URL. This broke for proxied
   endpoints and custom deployments. The `compat` struct on the model provides
   explicit, per-model capability declarations.

5. **sourceId for extension lifecycle.** Extensions register providers on load
   and must clean up on unload. The `sourceId` parameter to
   `registerApiProvider` enables `unregisterApiProviders(sourceId)` to remove
   exactly the providers that extension added, without affecting others.

6. **Wrap functions with API type check.** The registry wraps every stream
   function to assert `model.api == provider.api` at runtime. This catches
   misconfigured models early rather than producing mysterious provider errors.

7. **stream vs streamSimple.** `stream` is the raw provider call with
   provider-specific options. `streamSimple` is the unified interface that adds
   reasoning/thinking support. Most callers use `streamSimple`; only advanced
   integrations use `stream` directly.

8. **CacheRetention as a hint.** The `cacheRetention` field is mapped by each
   provider to its own caching mechanism (Anthropic prompt caching, Google
   context caching, etc.). Providers that do not support caching ignore it.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|---|---|
| Unknown API type | `getApiProvider` returns null. Callers must handle this (typically emit error stream). |
| Model api mismatch | Runtime assertion in wrapper throws immediately. This is a programming error, not a user error. |
| Provider SDK import fails | Dynamic import throws inside stream function. Provider must catch and push error event. |
| Rate limit with retry-after | Provider reads `maxRetryDelayMs`. If server requests longer delay, push error with the requested delay so higher-level logic can retry with user visibility. |
| Network timeout | Provider pushes error event with reason `"error"` and descriptive `errorMessage`. |
| AbortSignal fires | Provider pushes error event with reason `"aborted"`. |
| Extension unloaded mid-stream | Existing streams continue (they hold a reference to the provider function). Only future lookups fail. |
| Two extensions register same API | Last registration wins (Map.set overwrites). This is by design -- allows overrides. |
| Empty context (no messages) | Valid. Provider sends system prompt only. Model generates initial response. |
| No tools provided | `context.tools` is undefined. Provider omits tool definitions from the request. |
| Custom headers conflict | `options.headers` merged with model `headers`. Option headers override model headers on conflict. |
| onPayload callback modifies request | If `onPayload` returns a value, it replaces the request body. If it returns undefined, original payload is used. Provider must await the callback. |

## Testing Strategy

### Unit Tests for Registry

```
test "registerApiProvider adds to registry":
    clearApiProviders()
    mockProvider = { api: "test-api", stream: mockFn, streamSimple: mockFn }
    registerApiProvider(mockProvider)
    assert getApiProvider("test-api") is not null
    assert getApiProvider("test-api").api == "test-api"

test "getApiProvider returns null for unknown api":
    clearApiProviders()
    assert getApiProvider("nonexistent") is null

test "unregisterApiProviders removes by sourceId":
    clearApiProviders()
    registerApiProvider(providerA, sourceId = "ext-1")
    registerApiProvider(providerB, sourceId = "ext-2")
    unregisterApiProviders("ext-1")
    assert getApiProvider(providerA.api) is null
    assert getApiProvider(providerB.api) is not null

test "unregisterApiProviders leaves others untouched":
    clearApiProviders()
    registerApiProvider(providerA, sourceId = "ext-1")
    registerApiProvider(providerB)    # no sourceId
    unregisterApiProviders("ext-1")
    assert getApiProvider(providerB.api) is not null

test "clearApiProviders empties registry":
    registerApiProvider(providerA)
    clearApiProviders()
    assert getApiProviders().length == 0

test "api type mismatch throws":
    clearApiProviders()
    provider = { api: "anthropic-messages", stream: mockFn, streamSimple: mockFn }
    registerApiProvider(provider)
    wrongModel = createModel(api = "openai-completions")
    assert throws(() => getApiProvider("anthropic-messages").stream(wrongModel, ctx))
```

### Integration Tests with Mock Providers

```
test "streamSimple dispatches to correct provider":
    clearApiProviders()
    events = []
    mockProvider = createMockProvider("test-api", events)
    registerApiProvider(mockProvider)

    model = createModel(api = "test-api")
    context = { messages: [userMessage("hello")] }
    stream = streamSimple(model, context, {})

    result = await stream.result()
    assert result.stopReason == "stop"
    assert events contains "streamSimple called"

test "streamSimple returns error stream for unknown api":
    clearApiProviders()
    model = createModel(api = "unknown-api")
    stream = streamSimple(model, context, {})
    result = await stream.result()
    assert result.stopReason == "error"
    assert result.errorMessage contains "No provider"

test "api key resolution chain":
    # Priority 1: explicit option
    key = resolveApiKey(model, { apiKey: "explicit" }, config)
    assert key == "explicit"

    # Priority 2: dynamic callback
    config.getApiKey = () => "dynamic"
    key = resolveApiKey(model, {}, config)
    assert key == "dynamic"

    # Priority 3: environment variable
    setEnv("ANTHROPIC_API_KEY", "env-key")
    key = resolveApiKey(model, {}, {})
    assert key == "env-key"
```

### Compatibility Flag Tests

```
test "compat flags override auto-detection":
    model = createModel(
        api = "openai-completions",
        baseUrl = "https://custom-proxy.example.com",
        compat = {
            supportsDeveloperRole: false,
            maxTokensField: "max_tokens",
            requiresThinkingAsText: true,
        }
    )
    payload = buildOpenAIPayload(model, context, options)
    assert payload does not contain "developer" role
    assert payload contains "max_tokens" (not "max_completion_tokens")
    assert thinking blocks are converted to text with <thinking> delimiters

test "transport option selects websocket":
    model = createModel(api = "openai-codex-responses")
    stream = streamSimple(model, context, { transport: "websocket" })
    # Verify websocket transport was used (mock level)
    assert mockTransportLog.last == "websocket"

test "cacheRetention is provider-mapped":
    model = createModel(api = "anthropic-messages")
    stream = streamSimple(model, context, { cacheRetention: "long" })
    # Verify Anthropic cache_control headers were set appropriately
```

## References

- Source: `packages/ai/src/api-registry.ts` -- registry implementation
- Source: `packages/ai/src/types.ts` -- Model, Api, Provider, StreamOptions,
  SimpleStreamOptions, OpenAICompletionsCompat, KnownApi, KnownProvider
- Source: `packages/ai/src/providers/` -- provider implementations
- Depends on: `llm-streaming-event-protocol.skill.md`
- Pattern: Service Locator / Provider Registry
- Pattern: Strategy pattern (each provider is a strategy)
