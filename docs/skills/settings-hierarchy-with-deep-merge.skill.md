# Settings Hierarchy with Deep Merge

## Purpose

Implement a layered configuration system where settings from multiple sources (defaults, global config, project config, runtime overrides) are deep-merged together. Users can override specific nested fields without replacing entire objects. File locking prevents corruption from concurrent processes.

## Dependencies

None — this is a foundational pattern.

## Core Concepts

### Deep Merge

Unlike shallow merge (which replaces entire nested objects), deep merge recursively combines nested objects. If both the base and override have an object at the same key, their fields are merged rather than the override replacing the base entirely.

```
// Shallow merge: override replaces entire "compaction" object
base:     { compaction: { enabled: true, reserveTokens: 16384, keepRecentTokens: 20000 } }
override: { compaction: { reserveTokens: 8192 } }
result:   { compaction: { reserveTokens: 8192 } }  // Lost enabled and keepRecentTokens!

// Deep merge: only overridden fields change
result:   { compaction: { enabled: true, reserveTokens: 8192, keepRecentTokens: 20000 } }
```

### Settings Layers

Settings are resolved from multiple sources with increasing priority:

1. **Built-in defaults** — Hardcoded in the application
2. **Global settings** — User-wide config file (e.g., `~/.config/app/settings.json`)
3. **Project settings** — Per-project config file (e.g., `.app/settings.json` in project root)
4. **Runtime overrides** — CLI flags, environment variables, or programmatic overrides

### File Locking

When multiple instances of the application might read/write the same settings file concurrently, file locking prevents race conditions (lost updates, corrupt reads).

### Schema Validation

Settings are validated against a schema after loading and before saving. Unknown keys are preserved (forward compatibility) but known keys must match their expected types.

### Settings Migration

When settings files contain deprecated keys or outdated structure, migrations transform them to the current format on load.

## Data Structures

```
Settings {
    // Provider & Model
    defaultProvider: string?
    defaultModel: string?
    defaultThinkingLevel: enum { off, minimal, low, medium, high, xhigh }?

    // Context Management
    compaction: CompactionSettings {
        enabled: bool           // default: true
        reserveTokens: int      // default: 16384
        keepRecentTokens: int   // default: 20000
    }

    // Session
    branchSummary: BranchSummarySettings {
        enabled: bool           // default: true
    }

    // Network
    retry: RetrySettings {
        maxRetries: int         // default: 3
        initialDelayMs: int     // default: 1000
    }
    transport: enum { sse, websocket, auto }    // default: auto

    // Appearance
    theme: string?

    // Extensions
    extensions: list of ExtensionConfig {
        name: string
        enabled: bool
        path: string?
        config: map?
    }

    // Skills, Prompts, Packages
    skills: list of SkillConfig
    prompts: list of PromptConfig
    packages: list of PackageConfig

    // Terminal
    terminal: TerminalSettings {
        // Terminal-specific overrides
    }

    // Media
    images: ImageSettings {
        enabled: bool       // default: true
        maxWidth: int       // default: 800
        maxHeight: int      // default: 600
    }
    markdown: MarkdownSettings {
        enabled: bool       // default: true
    }
}

SettingsSource {
    path: string        // File path
    layer: enum { global, project, override }
    data: Settings      // Parsed settings
}
```

## Algorithm / Flow

### Deep Merge

```
function deepMerge(base, override):
    if override is null or undefined:
        return base

    result = shallowCopy(base)

    for key, value in override:
        if key not in result:
            // New key from override
            result[key] = value
        elif isPlainObject(result[key]) and isPlainObject(value):
            // Both are objects: recurse
            result[key] = deepMerge(result[key], value)
        else:
            // Scalar, array, or type mismatch: override replaces
            result[key] = value

    return result

function isPlainObject(value):
    // Returns true for plain objects (not arrays, dates, null, etc.)
    return value is not null
       and typeof value == "object"
       and not isArray(value)
       and not isDate(value)
       and not isRegExp(value)
```

### Settings Resolution

```
function resolveSettings(cwd, runtimeOverrides?):
    // Layer 1: Built-in defaults
    settings = copy(DEFAULT_SETTINGS)

    // Layer 2: Global settings
    globalPath = join(getConfigDir(), "settings.json")
    if fileExists(globalPath):
        globalSettings = loadAndValidate(globalPath)
        settings = deepMerge(settings, globalSettings)

    // Layer 3: Project settings
    projectPath = findProjectSettings(cwd)  // Walk up from cwd looking for .app/settings.json
    if projectPath:
        projectSettings = loadAndValidate(projectPath)
        settings = deepMerge(settings, projectSettings)

    // Layer 4: Runtime overrides
    if runtimeOverrides:
        settings = deepMerge(settings, runtimeOverrides)

    return settings

function findProjectSettings(cwd):
    // Walk up directory tree looking for project config
    dir = cwd
    while dir != parentOf(dir):  // Stop at filesystem root
        candidate = join(dir, ".app", "settings.json")
        if fileExists(candidate):
            return candidate
        dir = parentOf(dir)
    return null
```

### Loading with Validation

```
function loadAndValidate(path):
    raw = readFile(path)
    data = parseJSON(raw)

    // Validate known keys against schema
    errors = validateSchema(data, SETTINGS_SCHEMA)
    if errors:
        for error in errors:
            warn("Settings validation: " + error.path + " - " + error.message)
            // Remove invalid values so defaults apply
            deleteAtPath(data, error.path)

    // Migrate deprecated keys
    data = migrateSettings(data)

    // Unknown keys are preserved (forward compatibility)
    return data
```

### Saving Settings

```
function saveSettings(path, settings):
    withFileLock(path, fn():
        // Read current file to preserve unknown keys
        current = {}
        if fileExists(path):
            current = parseJSON(readFile(path))

        // Merge new settings into current (preserving unknowns)
        merged = deepMerge(current, settings)

        // Validate before writing
        errors = validateSchema(merged, SETTINGS_SCHEMA)
        if errors:
            error("Cannot save invalid settings: " + errors)
            return

        // Write atomically (write to temp, then rename)
        tempPath = path + ".tmp"
        writeFile(tempPath, serializeJSON(merged, indent=2))
        renameFile(tempPath, path)
    )
```

### File Locking

```
function withFileLock(path, fn):
    lockPath = path + ".lock"
    maxAttempts = 10
    delayMs = 20

    for attempt in 1 to maxAttempts:
        try:
            acquireLock(lockPath)
            try:
                result = fn()
                return result
            finally:
                releaseLock(lockPath)
        catch ELOCKED:
            if attempt == maxAttempts:
                throw "Failed to acquire lock after " + maxAttempts + " attempts"
            sleep(delayMs)

function withFileLockAsync(path, fn):
    // Same pattern but with async/await
    // Uses advisory file locks (e.g., lockfile.lock() or flock())
    ...
```

### Settings Migration

```
function migrateSettings(data):
    // Example migrations:

    // v1 → v2: "apiKey" moved to auth.json
    if "apiKey" in data:
        // Migrate to separate auth storage
        migrateApiKeyToAuth(data.apiKey)
        delete data.apiKey

    // v2 → v3: "model" string → { provider, modelId }
    if typeof data.defaultModel == "string" and ":" in data.defaultModel:
        parts = data.defaultModel.split(":")
        data.defaultProvider = parts[0]
        data.defaultModel = parts[1]

    // v3 → v4: Renamed fields
    if "maxContextTokens" in data:
        if "compaction" not in data:
            data.compaction = {}
        data.compaction.reserveTokens = data.maxContextTokens
        delete data.maxContextTokens

    return data
```

### Environment Variable Overrides

```
function getEnvironmentOverrides():
    overrides = {}

    // Specific env vars map to settings paths
    ENV_MAPPING = {
        "APP_DEFAULT_MODEL": "defaultModel",
        "APP_DEFAULT_PROVIDER": "defaultProvider",
        "APP_THEME": "theme",
        "APP_TRANSPORT": "transport",
        "APP_COMPACTION_ENABLED": "compaction.enabled",
    }

    for envVar, settingsPath in ENV_MAPPING:
        value = getEnv(envVar)
        if value is not null:
            setAtPath(overrides, settingsPath, parseValue(value))

    return overrides

function parseValue(value):
    // Attempt to parse as JSON for booleans and numbers
    if value == "true": return true
    if value == "false": return false
    if isNumeric(value): return parseNumber(value)
    return value  // String
```

## Key Design Decisions

1. **Deep merge, not shallow** — Essential for usability. Users should be able to override `compaction.reserveTokens` without knowing or specifying all other compaction fields.

2. **Project-level overrides** — Different projects may need different settings (e.g., different default models, different compaction thresholds). The project config layer enables this without changing global settings.

3. **File-based storage** — JSON files are human-readable, version-controllable (project settings can be committed), and don't require external dependencies. They work offline and are portable across platforms.

4. **Forward compatibility** — Unknown keys are preserved during load and save. This means a newer version's config file won't lose data when read by an older version. Known keys are validated; unknown keys pass through.

5. **Atomic writes** — Write to temp file, then rename. This prevents corruption if the process crashes during write. The rename operation is atomic on most filesystems.

6. **Advisory file locking** — Prevents concurrent processes from corrupting the file. Uses retry with backoff since locks are typically held very briefly.

7. **Schema validation with graceful degradation** — Invalid values are warned about and removed (falling back to defaults) rather than causing a crash. This handles hand-edited config files with typos.

## Edge Cases & Failure Modes

| Scenario | Handling |
|---|---|
| Missing config file | Skip that layer; defaults apply |
| Invalid JSON in config | Warn and use empty object for that layer |
| Invalid value for known key | Warn, remove invalid value, default applies |
| Unknown keys in config | Preserved (forward compatibility) |
| Circular reference in deep merge | Not possible with JSON-sourced data |
| Array merge semantics | Arrays are replaced, not merged (intentional — no way to "remove" an array element via merge) |
| Concurrent reads during write | File locking prevents reads during atomic write |
| Lock file left behind (stale lock) | Detect stale locks via process ID check; clean up |
| Config file permissions | Create with restrictive permissions (0o600) if contains sensitive data |
| Very deeply nested objects | Deep merge recurses; stack overflow only with adversarial input |
| Empty override file | deepMerge(base, {}) returns base unchanged |
| Null values in override | Null replaces the base value (intentional for "unset") |

## Testing Strategy

1. **Unit: Deep merge — scalars** — Override replaces base value
2. **Unit: Deep merge — nested objects** — Recursion merges nested fields
3. **Unit: Deep merge — new keys** — Override adds keys not in base
4. **Unit: Deep merge — arrays** — Arrays replaced, not merged
5. **Unit: Deep merge — null values** — Null in override replaces base
6. **Unit: Deep merge — mixed types** — Object in base, scalar in override → scalar wins
7. **Unit: Resolution order** — Global → project → override; later layers win
8. **Unit: Project settings discovery** — Walk up directory tree to find config
9. **Unit: Schema validation** — Invalid values removed with warning
10. **Unit: Unknown keys preserved** — Keys not in schema pass through
11. **Unit: Migration** — Deprecated keys transformed to current format
12. **Unit: Atomic write** — Temp file + rename; verify no partial writes
13. **Integration: File locking** — Two concurrent writers; both succeed without corruption
14. **Integration: Full round-trip** — Load → modify → save → reload → verify
15. **Integration: Environment overrides** — Env vars override file settings

## References

- `packages/coding-agent/src/core/settings-manager.ts` — SettingsManager class, deep merge, file I/O, validation, migration
- `packages/coding-agent/src/config.ts` — Config directory resolution, path utilities
- `packages/coding-agent/src/migrations.ts` — Settings migration logic
