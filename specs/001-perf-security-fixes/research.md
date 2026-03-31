# Research: Performance & Security Hardening

**Branch**: `001-perf-security-fixes` | **Date**: 2026-03-31

## R1: XSS Sanitization Pipeline Fix

### Decision
Remove the post-sanitization `&gt;` → `>` replacement, use UUID-based placeholders, and sanitize the final assembled content rather than pre-assembly content.

### Rationale
The current pipeline in `ChatMessage.vue:419-460` has a critical flaw: it runs DOMPurify on content with placeholders, then replaces `&gt;` back to `>` globally, then restores unsanitized protected content. This creates two bypass vectors:
1. The `&gt;` → `>` replacement undoes DOMPurify's escaping of angle brackets
2. Protected content (code blocks, math) is never sanitized and is restored raw into the output

### Alternatives Considered
- **Sanitize each protected block individually**: Rejected — would break code block rendering (code blocks legitimately contain HTML-like syntax)
- **Remove `allow-html` from XMarkdown**: Rejected — breaks legitimate HTML rendering in AI responses (tables, etc.)
- **Keep current pipeline, just remove `&gt;` line**: Insufficient — the placeholder restoration bypass remains

### Implementation Details
**Files to modify:**
- `Anywhere_window/src/components/ChatMessage.vue` (lines 419-460)
  - Replace sequential `__PROTECTED_CONTENT_N__` with UUID-based placeholders (e.g., `__PC_${crypto.randomUUID()}__`)
  - Remove line 449: `sanitizedPart = sanitizedPart.replace(/&gt;/g, '>');`
  - Remove `ADD_ATTR: ['style']` from DOMPurify config (line 445)
  - The blockquote `>` issue can be handled by the markdown renderer (XMarkdown) which processes markdown syntax AFTER our sanitization

- `Fast_window/append_selector.html` (line 149)
  - Replace `innerHTML` with DOM API (`createElement` + `textContent`)

- `Anywhere_main/src/App.vue` (line 215, 551)
  - Add `import DOMPurify from 'dompurify'` (need to add dependency to `Anywhere_main/package.json`)
  - Wrap `marked.parse(text)` with `DOMPurify.sanitize()`

- `Anywhere_main/src/components/Prompts.vue` (line 920)
  - Change `v-html="formatDescription(item.prompt)"` to `v-text="formatDescription(item.prompt)"`

## R2: Streaming Performance Optimization

### Decision
Throttle MutationObserver with `requestAnimationFrame`, debounce `addCopyButtonsToCodeBlocks`, and throttle `renderedMarkdownContent` recomputation during streaming.

### Rationale
Three independent hotspots create a multiplicative performance problem during streaming:
1. **MutationObserver** (`App.vue:1373-1386`): Fires on every `characterData` mutation, forcing synchronous reflow via `scrollTop = scrollHeight` read-write cycle
2. **Deep watcher** (`App.vue:1188-1190`): `watch(chat_show, ..., { deep: true })` triggers `addCopyButtonsToCodeBlocks()` which scans ALL `pre.hljs` elements on every token
3. **Computed property** (`ChatMessage.vue:419-460`): `renderedMarkdownContent` re-executes full DOMPurify + regex pipeline on every `props.message.content` change (every token)

### Alternatives Considered
- **Virtual scrolling for messages**: Too invasive for this scope; deferred to separate feature
- **Incremental markdown rendering (delta-only)**: Too complex; the markdown parser needs full context for correct rendering
- **Web Worker for sanitization**: Overhead of message passing negates benefit for small chunks

### Implementation Details
**MutationObserver fix** (`App.vue:1373-1386`):
```javascript
let scrollRAF = null;
chatObserver = new MutationObserver(() => {
  if (isSticky.value && !scrollRAF) {
    scrollRAF = requestAnimationFrame(() => {
      chatMainElement.scrollTop = chatMainElement.scrollHeight;
      scrollRAF = null;
    });
  }
});
```

**Deep watcher fix** (`App.vue:1188-1190`):
- Replace `{ deep: true }` watcher with a `watch` on `chat_show.length` (shallow — triggers on message add/remove)
- Add a one-shot call when `isStreaming` transitions from `true` to `false` (streaming complete)
- Debounce the function with a 500ms delay as fallback

**Computed property fix** (`ChatMessage.vue:419-460`):
- Convert `renderedMarkdownContent` from a `computed` to a manual `ref` + `watchEffect` with debounce
- During streaming (detect via `props.message.isStreaming` or similar flag), debounce recomputation to every 150ms
- On streaming complete, do one final immediate recomputation

## R3: Resource Cleanup

### Decision
Fix all identified resource leaks: autoSaveInterval, TextSearchUI listeners, backgroundShells map, and MCP abort signal listeners.

### Rationale
Each leak is independent and has a clear, low-risk fix.

### Implementation Details
**autoSaveInterval** (`App.vue:1824-1838`):
- Add `if (autoSaveInterval) { clearInterval(autoSaveInterval); autoSaveInterval = null; }` to `onBeforeUnmount`

**TextSearchUI** (`TextSearchUI.js:192-273, 463-468`):
- Store anonymous listener references as instance properties in `_bindEvents()`
- Remove all three global listeners in `destroy()`: `document.removeEventListener('keydown', this._handleKeydown)`, same for `mousemove` and `mouseup`
- Call `this.clear()` in `destroy()` to remove highlight marks

**backgroundShells** (`mcp_builtin.js:14-25, 2047-2057`):
- In the `close` handler, set a 5-minute cleanup timer: `setTimeout(() => backgroundShells.delete(shellId), 5 * 60 * 1000)`
- Null out `proc.process` reference when `active = false` to free the child_process object

**MCP abort signal** (`mcp.js:373-376`):
- Use `{ once: true }` option: `signal.addEventListener('abort', handler, { once: true })`

## R4: Task Scheduler Polling

### Decision
Increase polling interval from 1 second to 15 seconds.

### Rationale
The scheduler has minute-level resolution (checks hours and minutes, never seconds). The early-exit guard (`currentMinute <= lastCheckMinute`) means 59/60 ticks per second are pure waste. 15 seconds ensures worst-case 15-second delay on task trigger, which is acceptable for minute-granularity scheduling.

### Implementation Details
**`preload.js:637-774`**: Change `setInterval(..., 1000)` to `setInterval(..., 15000)`

## R5: Config Import/Export Safety

### Decision
Add pre-import backup and API key masking on export.

### Rationale
Config import is a destructive operation with no undo. API keys in exports create a data exposure risk when sharing configs.

### Implementation Details
**Pre-import backup** (`Setting.vue`):
- Before `window.api.updateConfig()` in both `importConfig()` (line 240) and `restoreFromWebdav()` (line 531):
  - Call `window.api.getConfig()` to get current config
  - Store as `config_backup_TIMESTAMP` in uTools DB via a new `window.api.backupConfig()` method
  - Backend `data.js`: add `backupConfig()` that saves full config to a `config_backup` DB document

**API key masking** (`Setting.vue:exportConfig()`):
- After deep-cloning config, iterate `providers` and replace `api_key` with masked version (last 4 chars)
- Add a checkbox/dialog option "Include full API keys" before export
- Default: masked. Opt-in: full keys with warning.

**Import confirmation** (`Setting.vue:importConfig()`):
- Add `ElMessageBox.confirm()` before applying import, similar to `restoreFromWebdav()`
