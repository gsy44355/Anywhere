# Implementation Plan: Performance & Security Hardening

**Branch**: `001-perf-security-fixes` | **Date**: 2026-03-31 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-perf-security-fixes/spec.md`

## Summary

Fix 5 XSS vulnerabilities across the content rendering pipeline (ChatMessage, append_selector, help docs, prompt descriptions), optimize 3 critical streaming performance bottlenecks (MutationObserver, deep watcher, computed markdown rendering), fix 4 resource leaks (autoSaveInterval, TextSearchUI listeners, backgroundShells, abort signal), reduce idle polling overhead, and add safety mechanisms for config import/export (pre-import backup, API key masking).

## Technical Context

**Language/Version**: JavaScript (ES2020+), Vue 3 (Composition API)
**Primary Dependencies**: Vue 3, Element Plus, DOMPurify, marked, highlight.js, XMarkdown (vue-element-plus-x), OpenAI SDK
**Storage**: uTools DB (key-value document store), local filesystem (JSON chat files), WebDAV (remote sync)
**Testing**: Manual testing (no automated test framework in project)
**Target Platform**: Electron (via uTools), cross-platform (macOS/Windows/Linux)
**Project Type**: Desktop plugin (Electron-based uTools plugin)
**Performance Goals**: 30+ FPS during streaming responses, <100ms input-to-render latency
**Constraints**: Must work within uTools plugin sandbox, no native module additions
**Scale/Scope**: 3 sub-projects (Anywhere_main, Anywhere_window, backend), ~12 files to modify

## Constitution Check

*GATE: No project-specific constitution defined. Using default best practices.*

- No new dependencies except `dompurify` added to `Anywhere_main` (already used in `Anywhere_window`)
- All changes are modifications to existing files, no new files created in source
- Changes are scoped to security fixes, performance optimizations, and bug fixes — no feature additions

## Project Structure

### Documentation (this feature)

```text
specs/001-perf-security-fixes/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0: Technical research
├── data-model.md        # Phase 1: Data model changes
├── quickstart.md        # Phase 1: Development quickstart
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
Anywhere_main/           # Settings/management Vue 3 frontend
├── src/
│   ├── App.vue                        # Help docs XSS fix (FR-003)
│   ├── components/
│   │   ├── Prompts.vue                # Prompt description XSS fix (FR-002)
│   │   └── Setting.vue                # Config import/export safety (FR-013, FR-014)
│   └── main.js
├── package.json                       # Add dompurify dependency

Anywhere_window/         # Chat window Vue 3 frontend
├── src/
│   ├── App.vue                        # MutationObserver + watcher + autoSave fixes (FR-005, FR-007, FR-008)
│   ├── components/
│   │   └── ChatMessage.vue            # Sanitization pipeline + render perf (FR-001, FR-004, FR-006, FR-015)
│   └── utils/
│       └── TextSearchUI.js            # Event listener cleanup (FR-009)

Fast_window/
├── append_selector.html               # innerHTML XSS fix (FR-002)

backend/
├── src/
│   ├── preload.js                     # Scheduler polling interval (FR-012)
│   ├── mcp.js                         # Abort signal listener fix (FR-011)
│   ├── mcp_builtin.js                 # Background shell cleanup (FR-010)
│   └── data.js                        # Config backup function (FR-013)
```

**Structure Decision**: Existing multi-project structure is preserved. Changes are distributed across all 3 sub-projects plus Fast_window. No structural changes needed.

## Implementation Phases

### Phase A: XSS Security Fixes (P1 — FR-001 through FR-004, FR-015)

**Goal**: Eliminate all 5 identified XSS vectors.

#### Task A1: Fix ChatMessage sanitization pipeline
**File**: `Anywhere_window/src/components/ChatMessage.vue` (lines 419-460)
**Changes**:
1. Replace `__PROTECTED_CONTENT_${placeholderIndex++}__` with `__PC_${crypto.randomUUID()}__` (FR-015)
2. Remove line 449: `sanitizedPart = sanitizedPart.replace(/&gt;/g, '>');` (FR-001)
3. Remove `ADD_ATTR: ['style']` from DOMPurify config at line 445 (FR-004)
4. Update placeholder restoration regex to match new UUID format

**Risk**: Removing `&gt;` → `>` may affect markdown blockquote rendering. XMarkdown handles markdown `>` syntax natively, so this should not be an issue since the `>` in markdown source is processed by the markdown parser before HTML generation.

**Risk**: Removing `style` attribute may break some AI-generated content with inline styles. This is acceptable — CSS injection is a greater risk than cosmetic loss.

#### Task A2: Fix append_selector innerHTML injection
**File**: `Fast_window/append_selector.html` (line 149)
**Changes**:
Replace:
```javascript
div.innerHTML = `<img src="${iconSrc}" ...><span>${item.name}</span>`;
```
With safe DOM API:
```javascript
const img = document.createElement('img');
img.src = iconSrc;
img.onerror = function() { this.src = 'logo.png'; };
const span = document.createElement('span');
span.textContent = item.name;
div.appendChild(img);
div.appendChild(span);
```

#### Task A3: Fix help docs rendering
**File**: `Anywhere_main/src/App.vue` (lines 215, 551)
**Prerequisite**: Add `dompurify` to `Anywhere_main/package.json`
**Changes**:
1. Add `import DOMPurify from 'dompurify';`
2. Change `currentDocContent.value = marked.parse(text);` to `currentDocContent.value = DOMPurify.sanitize(marked.parse(text));`

#### Task A4: Fix prompt description rendering
**File**: `Anywhere_main/src/components/Prompts.vue` (line 920)
**Changes**:
Change `v-html="formatDescription(item.prompt)"` to `v-text="formatDescription(item.prompt)"`

### Phase B: Streaming Performance Optimization (P1 — FR-005 through FR-007)

**Goal**: Eliminate UI jank during streaming responses.

#### Task B1: Throttle MutationObserver auto-scroll
**File**: `Anywhere_window/src/App.vue` (lines 1373-1386)
**Changes**:
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
Also cancel pending rAF in `onBeforeUnmount`: `if (scrollRAF) cancelAnimationFrame(scrollRAF);`

#### Task B2: Replace deep watcher with targeted approach
**File**: `Anywhere_window/src/App.vue` (lines 1188-1190)
**Changes**:
1. Replace `watch(chat_show, ..., { deep: true })` with `watch(() => chat_show.value.length, ...)`
2. Add a secondary watcher that triggers `addCopyButtonsToCodeBlocks()` when streaming ends (watch the streaming state flag)
3. Debounce `addCopyButtonsToCodeBlocks` with 500ms delay
4. Scope `addCopyButtonsToCodeBlocks` to only scan the last message's DOM element instead of the entire document

#### Task B3: Debounce markdown rendering during streaming
**File**: `Anywhere_window/src/components/ChatMessage.vue` (lines 419-460)
**Changes**:
1. Convert `renderedMarkdownContent` from `computed` to a `ref`
2. Use `watchEffect` with a debounce mechanism (150ms during streaming, immediate on final update)
3. Track streaming state by observing whether `props.message.content` is still changing

### Phase C: Resource Leak Fixes (P2 — FR-008 through FR-012)

**Goal**: Eliminate all identified resource leaks and reduce idle overhead.

#### Task C1: Clear autoSaveInterval on unmount
**File**: `Anywhere_window/src/App.vue` (lines 1824-1838)
**Changes**: Add to `onBeforeUnmount`:
```javascript
if (autoSaveInterval) { clearInterval(autoSaveInterval); autoSaveInterval = null; }
```

#### Task C2: Fix TextSearchUI event listener cleanup
**File**: `Anywhere_window/src/utils/TextSearchUI.js` (lines 192-273, 463-468)
**Changes**:
1. Store listener references as instance properties:
   ```javascript
   this._handleDocKeydown = (e) => { ... };
   this._handleDocMousemove = (e) => { ... };
   this._handleDocMouseup = () => { ... };
   ```
2. Update `destroy()`:
   ```javascript
   destroy() {
     this.clear();
     document.removeEventListener('keydown', this._handleDocKeydown);
     document.removeEventListener('mousemove', this._handleDocMousemove);
     document.removeEventListener('mouseup', this._handleDocMouseup);
     window.removeEventListener('resize', this._handleResize);
     if (this.container) this.container.remove();
   }
   ```

#### Task C3: Auto-cleanup background shells
**File**: `backend/src/mcp_builtin.js` (lines 2047-2057)
**Changes**: In the `close` handler:
```javascript
child.on('close', (code) => {
  const proc = backgroundShells.get(shellId);
  if (proc) {
    proc.active = false;
    proc.process = null; // Release child_process reference
    proc.cleanupTimer = setTimeout(() => {
      backgroundShells.delete(shellId);
    }, 5 * 60 * 1000); // 5 minutes
  }
  cleanupTempFile();
});
```
Also update `kill_background_shell` to clear the cleanup timer if it exists.

#### Task C4: Fix MCP abort signal listener
**File**: `backend/src/mcp.js` (lines 373-376)
**Changes**: Use `{ once: true }`:
```javascript
signal.addEventListener('abort', () => controller.abort(), { once: true });
```

#### Task C5: Reduce scheduler polling interval
**File**: `backend/src/preload.js` (lines 637-774)
**Changes**: Change `setInterval(..., 1000)` to `setInterval(..., 15000)`

### Phase D: Config Import/Export Safety (P3 — FR-013, FR-014)

**Goal**: Prevent data loss on import and protect API keys in exports.

#### Task D1: Add config backup function to backend
**File**: `backend/src/data.js`
**Changes**: Add a `backupConfig()` function that:
1. Calls `getConfig()` to get the full assembled config
2. Stores it as a `config_backup` document in uTools DB with timestamp and reason
3. Expose via preload API

#### Task D2: Add pre-import backup to Setting.vue
**File**: `Anywhere_main/src/components/Setting.vue` (lines 205-257, 486-549)
**Changes**:
1. Before `window.api.updateConfig()` in `importConfig()`: call `await window.api.backupConfig('import')`
2. Before `window.api.updateConfig()` in `restoreFromWebdav()`: call `await window.api.backupConfig('webdav_restore')`
3. Add confirmation dialog to `importConfig()` (similar to `restoreFromWebdav()`)

#### Task D3: Mask API keys in exports
**File**: `Anywhere_main/src/components/Setting.vue` (lines 170-203)
**Changes**: In `exportConfig()`, after deep-cloning config:
```javascript
// Mask API keys by default
if (configToExport.providers) {
  Object.values(configToExport.providers).forEach(provider => {
    if (provider.api_key) {
      const key = provider.api_key;
      provider.api_key = key.length > 4 ? '****' + key.slice(-4) : '****';
    }
  });
}
if (configToExport.webdav?.password) {
  configToExport.webdav.password = '****';
}
```
Add an `ElMessageBox.confirm` before export asking whether to include full API keys.

## Implementation Order

```
Phase A (XSS) ──→ Phase B (Performance) ──→ Phase C (Leaks) ──→ Phase D (Config)
   A1,A2,A3,A4       B1,B2,B3                C1,C2,C3,C4,C5      D1,D2,D3
   (parallel)         (sequential)            (parallel)           (sequential: D1→D2,D3)
```

Tasks within Phase A are independent and can be done in parallel. Phase B tasks should be done sequentially (B1→B2→B3) as they interact with the same streaming state. Phase C tasks are independent. Phase D requires D1 (backend) before D2 (frontend integration).
