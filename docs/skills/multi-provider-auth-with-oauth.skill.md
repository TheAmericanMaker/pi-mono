# Multi-Provider Authentication with OAuth

## Purpose

Provide a unified credential storage system that supports both static API keys
and short-lived OAuth tokens for multiple LLM providers. The system must handle
concurrent token refresh across multiple processes sharing the same credential
file, automatic migration from legacy storage formats, and a deterministic
resolution chain so that callers always get the highest-priority credential
available for a given provider.

## Dependencies

- **multi-provider-llm-abstraction.skill.md** -- the provider/model registry
  that consumes the API keys this module produces.
- A file-locking library capable of advisory locks on a single file (e.g.
  `proper-lockfile`, `flock`, or OS-level advisory locking).
- An HTTP client for OAuth token refresh endpoints.

## Core Concepts

### Credential types

Every provider maps to exactly one credential entry.  There are two variants:

| Variant    | Description                                         |
|------------|-----------------------------------------------------|
| `api_key`  | Static secret string; never expires on its own.     |
| `oauth`    | Access token + refresh token + expiry timestamp.    |

### Resolution chain

When the caller asks "give me a key for provider X", the system walks a
priority chain and returns the first hit:

1. **Runtime override** -- an in-memory key injected by the CLI (e.g.
   `--api-key` flag). Never persisted.
2. **API key from auth file** -- `type: "api_key"` entry in `auth.json`.
   Supports indirection (`$ENV_VAR` or `file:path` syntax via a
   `resolveConfigValue` helper).
3. **OAuth token from auth file** -- `type: "oauth"` entry. If the access
   token has expired, the system attempts a locked refresh before returning.
4. **Environment variable** -- provider-specific env vars such as
   `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, etc.
5. **Fallback resolver** -- a caller-supplied callback, used for custom
   providers defined in a models config file.

The chain short-circuits: as soon as a non-`undefined` value is found, it is
returned immediately.

### File locking

Multiple OS processes (e.g. several CLI instances) may share the same
`auth.json`.  When an OAuth token expires, each process will independently
notice and try to refresh.  File locking ensures only one process performs the
refresh while others wait and read the updated result.

## Data Structures

```
// ---- Credential variants ----

record ApiKeyCredential:
    type       = "api_key"
    key        : string            // raw key or "$ENV_VAR" / "file:/path"

record OAuthCredential:
    type         = "oauth"
    accessToken  : string
    refreshToken : string
    expires      : integer         // epoch millis
    tokenType    : string          // e.g. "Bearer"
    scope        : string          // space-separated scopes
    // ... provider-specific extra fields

union AuthCredential = ApiKeyCredential | OAuthCredential

// ---- Storage ----

type AuthStorageData = Map<string, AuthCredential>
//   key = provider name (e.g. "anthropic", "openai")

// ---- Lock result ----

record LockResult<T>:
    result : T                      // value to return to caller
    next   : string | undefined     // if set, new file content to write

// ---- Backend interface ----

interface AuthStorageBackend:
    // Synchronous lock + callback. Used for simple reads/writes.
    withLock<T>(fn: (currentFileContent: string?) -> LockResult<T>) -> T

    // Async lock + callback. Used when the callback must do I/O (token refresh).
    withLockAsync<T>(fn: (currentFileContent: string?) -> Promise<LockResult<T>>) -> Promise<T>
```

## Algorithm / Flow

### 1. Initialization

```
function createAuthStorage(path):
    backend = FileAuthStorageBackend(path)
    storage = AuthStorage(backend)
    storage.reload()
    return storage

function reload(storage):
    backend.withLock(current =>
        storage.data = parseJSON(current) or {}
        return { result: undefined }
    )
```

### 2. Reading a credential (getApiKey)

```
function getApiKey(storage, providerId, options?):
    // Priority 1: runtime override
    if storage.runtimeOverrides.has(providerId):
        return storage.runtimeOverrides.get(providerId)

    credential = storage.data[providerId]

    // Priority 2: stored API key
    if credential?.type == "api_key":
        return resolveConfigValue(credential.key)

    // Priority 3: stored OAuth token
    if credential?.type == "oauth":
        provider = getOAuthProvider(providerId)
        if provider is null:
            return undefined

        if currentTimeMillis() >= credential.expires:
            // Token expired -- refresh under lock
            try:
                result = refreshOAuthTokenWithLock(storage, providerId)
                if result is not null:
                    return result.apiKey
            catch error:
                storage.recordError(error)
                // Another process may have refreshed -- re-read file
                storage.reload()
                updated = storage.data[providerId]
                if updated?.type == "oauth" and currentTimeMillis() < updated.expires:
                    return provider.getApiKey(updated)
                return undefined
        else:
            return provider.getApiKey(credential)

    // Priority 4: environment variable
    envKey = getEnvApiKey(providerId)
    if envKey is not null:
        return envKey

    // Priority 5: fallback resolver
    if options?.includeFallback != false:
        return storage.fallbackResolver?(providerId)

    return undefined
```

### 3. Locked OAuth token refresh

```
function refreshOAuthTokenWithLock(storage, providerId):
    provider = getOAuthProvider(providerId)
    if provider is null:
        return null

    return storage.backend.withLockAsync(async current =>
        // Re-read under lock -- another process may have refreshed already
        currentData = parseJSON(current) or {}
        storage.data = currentData

        credential = currentData[providerId]
        if credential?.type != "oauth":
            return { result: null }

        // Double-check expiry under lock
        if currentTimeMillis() < credential.expires:
            return { result: { apiKey: provider.getApiKey(credential),
                               newCredentials: credential } }

        // Collect all OAuth credentials (some providers share refresh state)
        oauthCreds = {}
        for (key, value) in currentData:
            if value.type == "oauth":
                oauthCreds[key] = value

        // Call the OAuth provider's refresh endpoint
        refreshed = await getOAuthApiKey(providerId, oauthCreds)
        if refreshed is null:
            return { result: null }

        // Merge refreshed credential back into data
        merged = copy(currentData)
        merged[providerId] = { type: "oauth", ...refreshed.newCredentials }
        storage.data = merged

        return { result: refreshed,
                 next: toJSON(merged) }   // <-- triggers file write
    )
```

### 4. File-based lock backend (FileAuthStorageBackend)

```
function withLock(fn):
    ensureParentDir()    // mkdir -p with mode 0o700
    ensureFileExists()   // create "{}" with mode 0o600 if missing

    release = acquireLockWithRetry(authPath)
    try:
        current = readFile(authPath)
        { result, next } = fn(current)
        if next is not undefined:
            writeFile(authPath, next)
            chmod(authPath, 0o600)
        return result
    finally:
        release()

function acquireLockWithRetry(path):
    MAX_ATTEMPTS = 10
    DELAY_MS     = 20

    for attempt in 1..MAX_ATTEMPTS:
        try:
            return lockFile(path)
        catch error:
            if error.code != "ELOCKED" or attempt == MAX_ATTEMPTS:
                throw error
            busyWait(DELAY_MS)

    throw "Failed to acquire auth storage lock"

function withLockAsync(fn):
    ensureParentDir()
    ensureFileExists()

    lockCompromised = false
    release = await lockFileAsync(path,
        retries: { count: 10, factor: 2, minTimeout: 100ms, maxTimeout: 10s },
        stale: 30s,
        onCompromised: (err) => lockCompromised = true
    )

    try:
        throwIfCompromised()
        current = readFile(authPath)
        { result, next } = await fn(current)
        throwIfCompromised()
        if next is not undefined:
            writeFile(authPath, next)
            chmod(authPath, 0o600)
        throwIfCompromised()
        return result
    finally:
        release()
```

### 5. In-memory backend (for tests)

```
class InMemoryAuthStorageBackend:
    value : string?

    function withLock(fn):
        { result, next } = fn(this.value)
        if next is not undefined:
            this.value = next
        return result

    // withLockAsync is identical but async
```

### 6. Environment variable resolution

```
function getEnvApiKey(provider):
    // Special cases
    if provider == "anthropic":
        return env.ANTHROPIC_OAUTH_TOKEN or env.ANTHROPIC_API_KEY

    if provider == "google-vertex":
        if env.GOOGLE_CLOUD_API_KEY:
            return env.GOOGLE_CLOUD_API_KEY
        if hasVertexADCCredentials() and env.GOOGLE_CLOUD_PROJECT and env.GOOGLE_CLOUD_LOCATION:
            return "<authenticated>"    // sentinel: real auth happens in SDK

    if provider == "amazon-bedrock":
        if env.AWS_PROFILE or (env.AWS_ACCESS_KEY_ID and env.AWS_SECRET_ACCESS_KEY)
           or env.AWS_BEARER_TOKEN_BEDROCK or env.AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
           or env.AWS_WEB_IDENTITY_TOKEN_FILE:
            return "<authenticated>"

    // Standard mapping
    ENV_MAP = {
        "openai":       "OPENAI_API_KEY",
        "google":       "GEMINI_API_KEY",
        "groq":         "GROQ_API_KEY",
        "xai":          "XAI_API_KEY",
        "openrouter":   "OPENROUTER_API_KEY",
        "mistral":      "MISTRAL_API_KEY",
        "huggingface":  "HF_TOKEN",
        // ... etc.
    }

    envVar = ENV_MAP[provider]
    return envVar ? env[envVar] : undefined
```

### 7. Login and logout

```
function login(storage, providerId, callbacks):
    provider = getOAuthProvider(providerId)
    if provider is null:
        throw "Unknown OAuth provider"
    credentials = await provider.login(callbacks)
    storage.set(providerId, { type: "oauth", ...credentials })

function logout(storage, provider):
    storage.remove(provider)

function set(storage, provider, credential):
    storage.data[provider] = credential
    persistProviderChange(storage, provider, credential)

function remove(storage, provider):
    delete storage.data[provider]
    persistProviderChange(storage, provider, undefined)
```

### 8. Persistence helper

```
function persistProviderChange(storage, provider, credential):
    if storage.loadError:
        return    // don't write to a file we couldn't read

    storage.backend.withLock(current =>
        currentData = parseJSON(current) or {}
        merged = copy(currentData)
        if credential is not undefined:
            merged[provider] = credential
        else:
            delete merged[provider]
        return { result: undefined, next: toJSON(merged) }
    )
```

## Key Design Decisions

1. **Atomic read-modify-write via lock callbacks.**  The lock function reads the
   current file, passes it to the callback, and only writes back if the callback
   returns a `next` value.  This eliminates TOCTOU races that would occur with
   separate read/lock/write steps.

2. **File permissions 0o600 / 0o700.**  Credentials are sensitive.  The auth
   file is created with owner-only read/write (0o600) and the parent directory
   with owner-only access (0o700).

3. **Compromised lock detection.**  The async lock variant monitors for lock
   compromise (another process forcibly taking the lock).  If detected, the
   operation aborts rather than writing potentially stale data.

4. **Stale lock timeout (30 seconds).**  If a process crashes while holding the
   lock, other processes can reclaim it after 30 seconds.

5. **Runtime overrides are ephemeral.**  Keys set via `--api-key` live only in
   memory and never touch the file.  This prevents accidental persistence of
   one-time keys.

6. **Graceful degradation on refresh failure.**  If token refresh fails, the
   system re-reads the file (in case another process succeeded) before giving
   up.  If the re-read also shows an expired token, `undefined` is returned so
   the model discovery layer can skip the provider.

7. **Config value indirection.**  API key strings support `$ENV_VAR` and
   `file:/path` prefixes, resolved at read time.  This lets users store keys
   in secret managers or environment without changing auth.json.

## Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| Two processes refresh the same token simultaneously | First process wins; second sees the refreshed token under lock and skips the refresh |
| Refresh token itself has expired | OAuth provider returns an error; `undefined` is returned; user must `/login` again |
| Network failure during refresh | Error is caught, file is re-read for a peer-refreshed token, else `undefined` |
| Corrupt auth.json (invalid JSON) | `parseJSON` returns `{}`, effectively treating all providers as unconfigured; error is recorded |
| auth.json deleted while process is running | `ensureFileExists` re-creates it on next lock; in-memory data becomes authoritative |
| Lock file left stale after crash | Stale timeout (30s) allows other processes to reclaim; `onCompromised` fires if original process wakes up |
| Missing parent directory | `ensureParentDir` creates it recursively with 0o700 |
| Provider not in env map and not in auth.json | Falls through to fallback resolver; if that also returns undefined, caller gets undefined |
| `resolveConfigValue("$MISSING_VAR")` | Returns undefined or empty string depending on implementation; caller treats as missing key |
| Concurrent `set()` from different threads in same process | `withLock` is re-entrant-safe because each call acquires/releases independently; last writer wins for the same provider |

## Testing Strategy

### Unit tests (in-memory backend)

```
test "getApiKey returns runtime override first":
    storage = AuthStorage.inMemory({ "openai": { type: "api_key", key: "stored" } })
    storage.setRuntimeApiKey("openai", "override")
    assert await storage.getApiKey("openai") == "override"

test "getApiKey falls through to env var when no stored credential":
    setEnv("OPENAI_API_KEY", "from-env")
    storage = AuthStorage.inMemory({})
    assert await storage.getApiKey("openai") == "from-env"

test "getApiKey refreshes expired OAuth token":
    storage = AuthStorage.inMemory({
        "anthropic": { type: "oauth", accessToken: "old", refreshToken: "rt",
                       expires: pastTimestamp() }
    })
    mockOAuthRefresh(returns: { accessToken: "new", expires: futureTimestamp() })
    key = await storage.getApiKey("anthropic")
    assert key == "new"
    assert storage.get("anthropic").accessToken == "new"

test "getApiKey returns undefined when refresh fails and no peer refreshed":
    storage = AuthStorage.inMemory({
        "anthropic": { type: "oauth", expires: pastTimestamp(), refreshToken: "bad" }
    })
    mockOAuthRefresh(throws: Error("invalid_grant"))
    key = await storage.getApiKey("anthropic")
    assert key is undefined

test "set persists to backend":
    storage = AuthStorage.inMemory({})
    storage.set("openai", { type: "api_key", key: "k" })
    assert storage.has("openai")
    // Reload to verify persistence
    storage.reload()
    assert storage.get("openai").key == "k"

test "remove deletes credential":
    storage = AuthStorage.inMemory({ "openai": { type: "api_key", key: "k" } })
    storage.remove("openai")
    assert not storage.has("openai")
```

### Integration tests (file backend)

```
test "file permissions are 0o600":
    storage = AuthStorage.create(tempPath)
    storage.set("openai", { type: "api_key", key: "k" })
    assert fileMode(tempPath) == 0o600

test "concurrent refresh across two processes":
    // Fork two processes that both call getApiKey with an expired OAuth token
    // Assert: only one refresh call was made to the OAuth endpoint
    // Assert: both processes get the refreshed token

test "stale lock recovery":
    // Acquire lock, simulate crash (don't release)
    // After stale timeout, another process should acquire the lock
```

### Environment variable tests

```
test "anthropic prefers ANTHROPIC_OAUTH_TOKEN over ANTHROPIC_API_KEY":
    setEnv("ANTHROPIC_OAUTH_TOKEN", "oauth-val")
    setEnv("ANTHROPIC_API_KEY", "api-val")
    assert getEnvApiKey("anthropic") == "oauth-val"

test "google-vertex returns sentinel when ADC credentials present":
    mockADCCredentials(exist: true)
    setEnv("GOOGLE_CLOUD_PROJECT", "proj")
    setEnv("GOOGLE_CLOUD_LOCATION", "us-central1")
    assert getEnvApiKey("google-vertex") == "<authenticated>"

test "unknown provider returns undefined":
    assert getEnvApiKey("nonexistent") is undefined
```

## References

- OAuth 2.0 Token Refresh: RFC 6749, Section 6
- File locking: `proper-lockfile` (npm), `flock(2)` (POSIX)
- Source: `packages/coding-agent/src/core/auth-storage.ts`
- Source: `packages/ai/src/env-api-keys.ts`
