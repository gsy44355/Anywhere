# Data Model: Performance & Security Hardening

**Branch**: `001-perf-security-fixes` | **Date**: 2026-03-31

## Entities

### Configuration Backup

A timestamped snapshot of the full configuration, created before destructive operations.

**Fields:**
- `_id`: `"config_backup"` (uTools DB document ID)
- `timestamp`: ISO 8601 string (when the backup was created)
- `config`: Full configuration object (same structure as assembled by `getConfig()`)
- `reason`: String describing why the backup was created (e.g., `"import"`, `"webdav_restore"`)

**Lifecycle:**
- Created before config import or WebDAV restore
- Only the most recent backup is kept (overwrites previous)
- Can be restored manually via a "Restore last backup" action

**Relationships:**
- Contains the same structure as the split config documents (`config`, `prompts`, `providers`, `mcpServers`, `tasks`)

---

### Background Shell Entry (existing, modified)

An entry in the in-memory `backgroundShells` Map tracking a spawned background process.

**Fields (current):**
- `process`: ChildProcess instance
- `command`: String (the command that was run)
- `startTime`: ISO 8601 string
- `logs`: String (accumulated stdout/stderr, max 1MB)
- `pid`: Number
- `active`: Boolean

**Fields (added):**
- `cleanupTimer`: Timer ID (the setTimeout handle for auto-cleanup after completion)

**State transitions:**
- `active: true` → process running
- `active: false` → process exited, `cleanupTimer` set for 5-minute auto-delete
- Entry deleted → removed from Map (after cleanup timer fires, or explicit `kill_background_shell`)

---

### Sanitization Placeholder (existing, modified)

Internal mapping used during the ChatMessage content sanitization pipeline.

**Current format:** `__PROTECTED_CONTENT_0__`, `__PROTECTED_CONTENT_1__`, ...
**New format:** `__PC_<uuid>__` where `<uuid>` is generated via `crypto.randomUUID()`

**Rationale:** Sequential integer-based placeholders can collide with user content. UUID-based placeholders are collision-resistant.

---

### Export Config (existing, modified)

The configuration object written to a JSON file during export.

**Current behavior:** Contains all fields including plaintext API keys.
**New behavior:** `providers[*].api_key` values are masked by default (show last 4 characters as `****...xxxx`). A user opt-in flag controls whether full keys are included.

## No New Database Tables/Collections

All changes operate on existing storage mechanisms:
- uTools DB documents (for config backup)
- In-memory Map (for background shells)
- Transient in-memory Map (for sanitization placeholders)
