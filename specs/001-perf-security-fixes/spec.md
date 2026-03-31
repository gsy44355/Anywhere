# Feature Specification: Performance & Security Hardening

**Feature Branch**: `001-perf-security-fixes`
**Created**: 2026-03-31
**Status**: Draft
**Input**: User description: "根据代码审阅分析，修复XSS等高危安全漏洞，优化流式响应性能瓶颈（MutationObserver/deep watcher/Markdown渲染管线），修复内存泄漏与定时器未清理等Bug，提升快速响应AI助手的整体稳定性和响应速度。"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Secure Chat Content Rendering (Priority: P1)

As a user chatting with an AI assistant, I want to be confident that AI responses (including those from untrusted sources or manipulated prompts) cannot execute malicious scripts or inject harmful content into the plugin interface, so that my system and data remain safe.

**Why this priority**: XSS vulnerabilities are the highest-severity security issue found. An adversarial AI response or crafted prompt name could steal API keys, hijack the user session, or execute arbitrary code in the Electron context.

**Independent Test**: Can be tested by sending messages containing HTML/script payloads through the chat interface and verifying none execute.

**Acceptance Scenarios**:

1. **Given** an AI response containing `<script>alert('xss')</script>`, **When** the message is rendered in the chat window, **Then** no JavaScript executes and the raw text is displayed as escaped content.
2. **Given** an AI response containing CSS injection via `style` attribute (e.g., `position:fixed; z-index:99999; width:100vw; height:100vh`), **When** the message is rendered, **Then** the dangerous CSS properties are stripped or the style attribute is disallowed.
3. **Given** a Prompt with a name containing `<img onerror=alert(1)>`, **When** the append selector window shows that prompt, **Then** the name is displayed as escaped text, not executed as HTML.
4. **Given** externally-fetched Markdown documentation (from GitHub/Gitee), **When** rendered in the help viewer, **Then** the HTML output is sanitized before insertion and no embedded scripts execute.
5. **Given** a user-authored Prompt description containing HTML tags, **When** displayed in the Prompts management list, **Then** the content is escaped and no HTML is rendered as active elements.
6. **Given** the content sanitization pipeline in ChatMessage, **When** content passes through DOMPurify, **Then** no post-processing step re-introduces raw HTML angle brackets that were previously escaped.

---

### User Story 2 - Smooth Streaming Response Experience (Priority: P1)

As a user asking the AI a question, I want the streaming response to render smoothly without UI freezing or visible lag, so that the plugin feels like a fast, responsive assistant rather than a sluggish application.

**Why this priority**: The current rendering pipeline fires hundreds of times per second during streaming due to unthrottled observers and watchers. This is the primary cause of perceived UI sluggishness and directly contradicts the "quick response AI assistant" positioning.

**Independent Test**: Can be tested by initiating a long streaming response (500+ words with code blocks) and measuring UI responsiveness during streaming.

**Acceptance Scenarios**:

1. **Given** a streaming AI response in progress, **When** each text chunk arrives, **Then** the auto-scroll behavior updates at most once per animation frame (throttled), not on every character mutation.
2. **Given** a streaming AI response with code blocks, **When** chunks are being rendered, **Then** copy buttons are only injected after streaming is complete or on a debounced interval, not on every reactive change.
3. **Given** a conversation with 50+ messages and a new streaming response, **When** streaming is in progress, **Then** the content rendering pipeline processes at a debounced or batched rate rather than on every single chunk.
4. **Given** a conversation with 100+ messages, **When** scrolling through the history, **Then** the scroll performance remains fluid with no visible jank.

---

### User Story 3 - Reliable Resource Cleanup (Priority: P2)

As a user who keeps chat windows open for extended periods, I want the plugin to properly clean up timers, event listeners, and background processes, so that memory usage stays stable and the plugin does not degrade over time.

**Why this priority**: Memory leaks from uncleaned intervals, event listeners, and background shell entries cause gradual performance degradation. For a tool used throughout the workday, this directly impacts long-term reliability.

**Independent Test**: Can be tested by opening a chat window, using features (search, background shells), closing the window, and verifying no orphaned timers or listeners remain.

**Acceptance Scenarios**:

1. **Given** a chat window with auto-save enabled, **When** the window is closed, **Then** the auto-save interval timer is cleared and no further save attempts occur.
2. **Given** a text search session in progress, **When** the search UI is destroyed (via close button or window close), **Then** all keyboard and mouse event listeners are removed from the document.
3. **Given** background shell processes that have completed execution, **When** they remain in the background shells registry, **Then** completed entries are automatically cleaned up after a reasonable period.
4. **Given** an MCP tool invocation using an external abort signal, **When** the invocation completes, **Then** the abort event listener is removed from the signal.

---

### User Story 4 - Reduced Idle Resource Consumption (Priority: P2)

As a user running the plugin in the background, I want minimal CPU and I/O usage when the plugin is idle, so that it does not impact system performance or battery life.

**Why this priority**: The 1-second task scheduler polling with full config reads is wasteful for a minimum 1-minute scheduling granularity. Reducing idle overhead is important for a background-running plugin.

**Independent Test**: Can be tested by monitoring CPU and I/O activity when the plugin is running with no active conversations.

**Acceptance Scenarios**:

1. **Given** the plugin is running with no active conversations, **When** the task scheduler is polling for due tasks, **Then** the polling interval is at most once every 30 seconds.
2. **Given** the task scheduler polling, **When** the configuration is needed, **Then** the result is cached for at least the polling interval to avoid redundant database reads.

---

### User Story 5 - Safe Configuration Import/Export (Priority: P3)

As a user importing or restoring a configuration, I want a safety net that prevents accidental data loss, so that I can recover if an import goes wrong. I also want API keys to be protected during export.

**Why this priority**: Data loss from bad imports is serious but less frequent than the above scenarios. API key exposure in exports is a secondary security concern.

**Independent Test**: Can be tested by exporting config, importing a malformed file, and verifying original config is recoverable.

**Acceptance Scenarios**:

1. **Given** the user initiates a config import, **When** the import process begins, **Then** a backup of the current configuration is automatically created before any changes are applied.
2. **Given** the user exports the configuration, **When** the export file is generated, **Then** API keys are masked by default (e.g., showing only last 4 characters), with an explicit opt-in to include full keys.
3. **Given** the user restores from WebDAV, **When** the restore begins, **Then** a local backup of the current config is created first.

---

### Edge Cases

- What happens when the sanitization pipeline encounters a message containing the literal string `__PROTECTED_CONTENT_0__`? The system must use collision-resistant placeholders (e.g., UUID-based) to prevent content corruption.
- What happens when a streaming response contains mixed markdown, code blocks, and math formulas simultaneously? All content types must render correctly without interference from debouncing.
- What happens when the user opens the text search UI during an active streaming response? The search must not corrupt the DOM or interfere with streaming updates.
- What happens when two chat windows attempt to save configuration simultaneously? The system must not silently drop one window's changes.
- What happens when a background shell process never terminates? The system should warn about long-running processes and provide visibility.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST sanitize all AI-generated content through DOMPurify without any post-processing that re-introduces raw HTML characters (specifically, no `&gt;` to `>` replacement after sanitization).
- **FR-002**: System MUST escape all user-controlled text (prompt names, descriptions) before inserting into innerHTML contexts.
- **FR-003**: System MUST sanitize externally-fetched Markdown documentation through DOMPurify before rendering via v-html.
- **FR-004**: System MUST remove the `style` attribute from the DOMPurify allowed attributes list, or restrict it to a safe subset of CSS properties that cannot be used for UI overlay attacks.
- **FR-005**: System MUST throttle the MutationObserver auto-scroll callback to at most one execution per animation frame during streaming.
- **FR-006**: System MUST debounce or batch the Markdown rendering/sanitization pipeline during streaming to avoid per-chunk full re-computation.
- **FR-007**: System MUST replace the deep watcher on `chat_show` with a targeted approach that only triggers copy button injection on message completion or addition, not on every reactive property change.
- **FR-008**: System MUST clear the auto-save interval timer when the chat window component is unmounted.
- **FR-009**: System MUST remove all event listeners (keydown, mousemove, mouseup, resize) registered by TextSearchUI when `destroy()` is called.
- **FR-010**: System MUST automatically clean up completed background shell entries from the registry after a configurable idle period.
- **FR-011**: System MUST remove abort signal event listeners after MCP tool invocations complete.
- **FR-012**: System MUST reduce the task scheduler polling interval to no more than once every 30 seconds.
- **FR-013**: System MUST create an automatic backup of the current configuration before config import or WebDAV restore operations.
- **FR-014**: System MUST mask API keys by default in config exports (showing only last 4 characters), with an explicit opt-in for full keys.
- **FR-015**: System MUST use collision-resistant placeholders (e.g., UUID-based) for the content protection mechanism in the sanitization pipeline instead of predictable sequential identifiers.

### Key Entities

- **Sanitization Pipeline**: The chain of operations that processes AI-generated content before rendering (placeholder extraction, DOMPurify pass, content restoration). Central to XSS prevention.
- **Streaming State**: The set of reactive data and observers involved during an active AI response (MutationObserver, watchers, computed properties). Central to performance.
- **Resource Registry**: Background shells, event listeners, and timers that require explicit cleanup on component/window destruction.
- **Configuration Backup**: A timestamped snapshot of the full configuration created before destructive operations (import/restore).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Zero XSS vulnerabilities remain in all content rendering paths (chat messages, prompt names, help docs, prompt descriptions, append selector). Verified by testing with standard XSS payloads from the OWASP cheat sheet.
- **SC-002**: During streaming of a 500-word response with 3+ code blocks, the UI maintains smooth rendering with no perceptible freezing or dropped frames.
- **SC-003**: Plugin idle CPU/IO usage (no active conversations) decreases by at least 80% compared to the current 1-second polling baseline.
- **SC-004**: After closing a chat window, zero orphaned timers, event listeners, or observers remain attached to the document or global objects.
- **SC-005**: Configuration import/restore operations always produce a recoverable backup, enabling 100% rollback success to the previous configuration.
- **SC-006**: Exported configuration files contain masked API keys by default, with no plaintext keys unless the user explicitly opts in.

## Assumptions

- The uTools plugin runtime environment supports standard Web APIs (requestAnimationFrame, MutationObserver, AbortController) used for performance optimizations.
- DOMPurify is already a project dependency and its API will remain stable for the required configuration changes.
- The existing auto-save functionality will continue to work correctly with the increased (30-second) scheduler polling interval, since auto-save is triggered by its own separate timer.
- Config backup files will be stored alongside the original config in the uTools data directory, using a timestamped naming convention.
- Message list virtualization (virtual scrolling) is documented as a recommendation but is NOT in scope for this specification, as it requires significant architectural changes.
- Shell command execution security (denylist vs allowlist) is NOT in scope, as it requires a separate design discussion around the MCP tool permission model.
- The `insert_content` CRLF-to-LF conversion bug and hardcoded Chinese strings are minor issues that can be fixed as standalone changes outside this specification.
- The KeepAlive/v-if pattern fix in App.vue (main) is a separate optimization not covered by this spec.
