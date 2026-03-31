# Tasks: Performance & Security Hardening

**Input**: Design documents from `/specs/001-perf-security-fixes/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, quickstart.md

**Tests**: Not included — project has no automated test framework. Manual testing procedures are documented in quickstart.md.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Anywhere_main**: Settings/management Vue 3 frontend (`Anywhere_main/src/`)
- **Anywhere_window**: Chat window Vue 3 frontend (`Anywhere_window/src/`)
- **Fast_window**: Quick input HTML windows (`Fast_window/`)
- **backend**: Node.js preload scripts (`backend/src/`)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Add the only new dependency needed for this feature

- [ ] T001 Add `dompurify` dependency to `Anywhere_main/package.json` by running `cd Anywhere_main && pnpm add dompurify`

---

## Phase 2: User Story 1 — Secure Chat Content Rendering (Priority: P1) 🎯 MVP

**Goal**: Eliminate all 5 identified XSS vulnerabilities across content rendering paths so that no AI response, prompt name, or external markdown can execute scripts or inject harmful content.

**Independent Test**: Send messages containing `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>`, `<svg onload=alert(1)>`, and CSS injection payloads through each rendering path. Verify none execute. Test prompt names with HTML in append_selector. Test help docs viewer. Test prompt description display.

### Implementation for User Story 1

- [ ] T002 [P] [US1] Fix ChatMessage sanitization pipeline — remove post-sanitization `&gt;` to `>` replacement (line ~449), remove `ADD_ATTR: ['style']` from DOMPurify config (line ~445), replace sequential `__PROTECTED_CONTENT_N__` placeholders with UUID-based `__PC_${crypto.randomUUID()}__` placeholders, and update the restoration regex to match the new UUID format in `Anywhere_window/src/components/ChatMessage.vue`
- [ ] T003 [P] [US1] Fix append_selector innerHTML injection — replace `div.innerHTML` template literal (line ~149) with safe DOM API using `document.createElement` + `textContent` for `item.name` to prevent HTML execution in `Fast_window/append_selector.html`
- [ ] T004 [P] [US1] Fix help docs XSS — add `import DOMPurify from 'dompurify'` and wrap `marked.parse(text)` output with `DOMPurify.sanitize()` before assigning to `currentDocContent.value` (line ~215) in `Anywhere_main/src/App.vue`
- [ ] T005 [P] [US1] Fix prompt description XSS — change `v-html="formatDescription(item.prompt)"` to `v-text="formatDescription(item.prompt)"` (line ~920) in `Anywhere_main/src/components/Prompts.vue`
- [ ] T006 [US1] Manual XSS verification — test all 5 rendering paths with OWASP XSS payloads: ChatMessage (script tags, style injection, placeholder collision with literal `__PROTECTED_CONTENT_0__`), append_selector (HTML in prompt names), help docs (script in markdown), prompt descriptions (HTML tags). Verify zero execution across all paths.

**Checkpoint**: All XSS vectors eliminated. User Story 1 is independently testable and complete.

---

## Phase 3: User Story 2 — Smooth Streaming Response Experience (Priority: P1)

**Goal**: Eliminate UI jank during streaming by throttling the MutationObserver, replacing the deep watcher, and debouncing markdown rendering so the plugin feels like a fast, responsive assistant.

**Independent Test**: Open a conversation with 50+ messages, trigger a streaming response with 500+ words including 3+ code blocks, and verify smooth rendering with no perceptible freezing.

### Implementation for User Story 2

- [ ] T007 [US2] Throttle MutationObserver auto-scroll — wrap the `scrollTop = scrollHeight` assignment in a `requestAnimationFrame` guard (coalesce multiple mutations to 1 scroll per frame), add `cancelAnimationFrame` cleanup in `onBeforeUnmount` in `Anywhere_window/src/App.vue` (lines ~1373-1386)
- [ ] T008 [US2] Replace deep watcher on `chat_show` with targeted approach — change `watch(chat_show, ..., { deep: true })` to `watch(() => chat_show.value.length, ...)` for message add/remove detection, add a secondary watcher on streaming completion state to trigger `addCopyButtonsToCodeBlocks()`, debounce `addCopyButtonsToCodeBlocks` with 500ms delay, and scope it to only scan the last message's DOM element instead of `document.querySelectorAll` in `Anywhere_window/src/App.vue` (lines ~1188-1190, ~725-748)
- [ ] T009 [US2] Debounce markdown rendering during streaming — convert `renderedMarkdownContent` from `computed` to a `ref` with manual `watchEffect`, add 150ms debounce during active streaming (detect via content change frequency), ensure immediate final recomputation when streaming ends in `Anywhere_window/src/components/ChatMessage.vue` (lines ~419-460)
- [ ] T010 [US2] Manual streaming performance verification — test with a 500-word streaming response containing code blocks and math formulas, verify smooth scrolling and no UI freezes, test that copy buttons appear correctly after streaming completes, test that mixed content (markdown + code + math) renders correctly with debouncing

**Checkpoint**: Streaming responses render smoothly. User Story 2 is independently testable and complete.

---

## Phase 4: User Story 3 — Reliable Resource Cleanup (Priority: P2)

**Goal**: Fix all identified resource leaks (timers, event listeners, background processes) so that memory usage stays stable during extended use.

**Independent Test**: Open chat window, use text search, run background shells, close window, verify zero orphaned timers/listeners via DevTools.

### Implementation for User Story 3

- [ ] T011 [P] [US3] Clear autoSaveInterval on unmount — add `if (autoSaveInterval) { clearInterval(autoSaveInterval); autoSaveInterval = null; }` to the `onBeforeUnmount` handler in `Anywhere_window/src/App.vue` (lines ~1824-1838)
- [ ] T012 [P] [US3] Fix TextSearchUI event listener cleanup — refactor `_bindEvents()` to store anonymous global listeners as named instance properties (`this._handleDocKeydown`, `this._handleDocMousemove`, `this._handleDocMouseup`), update `destroy()` to call `this.clear()` and `document.removeEventListener` for all three global listeners plus the existing `window.removeEventListener('resize', this._handleResize)` in `Anywhere_window/src/utils/TextSearchUI.js` (lines ~192-273, ~463-468)
- [ ] T013 [P] [US3] Auto-cleanup background shell entries — in the `child.on('close')` handler, set `proc.process = null` to release the ChildProcess reference, add a `setTimeout` (5 minutes) to auto-delete the entry from `backgroundShells` Map, store the timer ID as `proc.cleanupTimer`, update `kill_background_shell` to clear the cleanup timer via `clearTimeout` if it exists before deletion in `backend/src/mcp_builtin.js` (lines ~2047-2057, ~2199-2232)
- [ ] T014 [P] [US3] Fix MCP abort signal listener leak — change `signal.addEventListener('abort', () => controller.abort())` to `signal.addEventListener('abort', () => controller.abort(), { once: true })` so the listener auto-removes after firing in `backend/src/mcp.js` (lines ~373-376)
- [ ] T015 [US3] Manual resource cleanup verification — open chat window with search active, close window, check DevTools for orphaned listeners; run background shell, wait for completion, verify auto-cleanup after 5 minutes; check autoSave timer is cleared on unmount

**Checkpoint**: All resource leaks fixed. User Story 3 is independently testable and complete.

---

## Phase 5: User Story 4 — Reduced Idle Resource Consumption (Priority: P2)

**Goal**: Reduce idle CPU/IO usage by lowering the task scheduler polling frequency from 1 second to 15 seconds.

**Independent Test**: Run plugin with no active conversations, monitor CPU usage, compare before/after polling interval change.

### Implementation for User Story 4

- [ ] T016 [US4] Reduce scheduler polling interval — change `setInterval(..., 1000)` to `setInterval(..., 15000)` in the task scheduler setup in `backend/src/preload.js` (lines ~637-774)
- [ ] T017 [US4] Manual idle verification — run plugin idle for 60 seconds, verify task scheduler fires ~4 times (every 15s) instead of ~60 times, confirm scheduled tasks still trigger correctly at minute boundaries

**Checkpoint**: Idle overhead reduced. User Story 4 is independently testable and complete.

---

## Phase 6: User Story 5 — Safe Configuration Import/Export (Priority: P3)

**Goal**: Add pre-import config backup and API key masking on export to prevent data loss and credential exposure.

**Independent Test**: Export config and verify API keys are masked. Import a config file and verify backup was created. Restore from WebDAV and verify backup was created.

### Implementation for User Story 5

- [ ] T018 [US5] Add `backupConfig()` function to backend — implement a function that calls `getConfig()` to assemble the full config, stores it as a `config_backup` document in uTools DB with `timestamp` and `reason` fields, expose as `backupConfig(reason)` on the preload API (`window.api`) in `backend/src/data.js`
- [ ] T019 [US5] Wire `backupConfig` through preload — expose the new `backupConfig` function via `window.api` in both `backend/src/preload.js` and `backend/src/window_preload.js` so it is accessible from the settings frontend
- [ ] T020 [US5] Add pre-import backup and confirmation dialog — in `importConfig()`, add `ElMessageBox.confirm` before applying import (similar to existing `restoreFromWebdav` pattern), call `await window.api.backupConfig('import')` before `window.api.updateConfig()`. In `restoreFromWebdav()`, call `await window.api.backupConfig('webdav_restore')` before `window.api.updateConfig()` in `Anywhere_main/src/components/Setting.vue` (lines ~205-257, ~486-549)
- [ ] T021 [US5] Mask API keys in config exports — in `exportConfig()`, after deep-cloning config, iterate `configToExport.providers` and replace each `api_key` with masked version (last 4 chars: `'****' + key.slice(-4)`), mask `webdav.password` as `'****'`. Add `ElMessageBox.confirm` asking whether to include full API keys before export, defaulting to masked in `Anywhere_main/src/components/Setting.vue` (lines ~170-203)
- [ ] T022 [US5] Manual config safety verification — export config and verify API keys show as `****xxxx`, import a config file and verify backup document exists in uTools DB, restore from WebDAV and verify backup was created, test import with confirmation dialog

**Checkpoint**: Config import/export is safe. User Story 5 is independently testable and complete.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final verification across all user stories

- [ ] T023 Build all three sub-projects (`Anywhere_main`, `Anywhere_window`, `backend`) and verify no build errors
- [ ] T024 End-to-end smoke test — open plugin, send a message with XSS payload, verify sanitization; trigger streaming response, verify smooth rendering; use text search then close, verify cleanup; check idle CPU; export/import config with backup verification

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — install dompurify first
- **US1 XSS (Phase 2)**: Depends on Phase 1 (dompurify dependency) — BLOCKS nothing
- **US2 Performance (Phase 3)**: No dependency on Phase 1 or 2 — can run in parallel with US1
- **US3 Resource Cleanup (Phase 4)**: No dependency on Phases 1-3 — can run in parallel
- **US4 Idle (Phase 5)**: No dependency on Phases 1-4 — can run in parallel
- **US5 Config Safety (Phase 6)**: No dependency on Phases 1-5 — can run in parallel (T018→T019→T020,T021 is sequential within)
- **Polish (Phase 7)**: Depends on all user stories being complete

### User Story Dependencies

- **US1 (P1 XSS)**: Depends only on T001 (dompurify install). No cross-story dependencies.
- **US2 (P1 Performance)**: Fully independent. No cross-story dependencies.
- **US3 (P2 Cleanup)**: Fully independent. No cross-story dependencies.
- **US4 (P2 Idle)**: Fully independent. No cross-story dependencies.
- **US5 (P3 Config)**: T018→T019 must complete before T020, T021. No cross-story dependencies.

### Within Each User Story

- Tasks marked [P] within the same story can run in parallel
- Verification tasks (T006, T010, T015, T017, T022) must run after all implementation tasks in their story

### Parallel Opportunities

All 5 user stories touch different files and can be worked on simultaneously:

| Story | Files Modified |
|-------|---------------|
| US1 | ChatMessage.vue, append_selector.html, App.vue (main), Prompts.vue |
| US2 | App.vue (window), ChatMessage.vue* |
| US3 | App.vue (window)*, TextSearchUI.js, mcp_builtin.js, mcp.js |
| US4 | preload.js |
| US5 | data.js, preload.js*, window_preload.js, Setting.vue |

*Note: US2 and US3 both modify `App.vue (window)` but in different sections (streaming logic vs unmount cleanup). US2 also modifies `ChatMessage.vue` but in a different section than US1 (rendering pipeline vs sanitization config). These can be parallelized with care, or done sequentially to avoid merge conflicts.*

---

## Parallel Example: User Story 1 (XSS Fixes)

```bash
# All 4 implementation tasks touch different files — run in parallel:
Task: "T002 Fix ChatMessage sanitization pipeline in Anywhere_window/src/components/ChatMessage.vue"
Task: "T003 Fix append_selector innerHTML in Fast_window/append_selector.html"
Task: "T004 Fix help docs XSS in Anywhere_main/src/App.vue"
Task: "T005 Fix prompt description XSS in Anywhere_main/src/components/Prompts.vue"

# Then run verification:
Task: "T006 Manual XSS verification across all paths"
```

## Parallel Example: User Story 3 (Resource Cleanup)

```bash
# All 4 implementation tasks touch different files — run in parallel:
Task: "T011 Clear autoSaveInterval in Anywhere_window/src/App.vue"
Task: "T012 Fix TextSearchUI listeners in Anywhere_window/src/utils/TextSearchUI.js"
Task: "T013 Auto-cleanup background shells in backend/src/mcp_builtin.js"
Task: "T014 Fix MCP abort signal in backend/src/mcp.js"

# Then run verification:
Task: "T015 Manual resource cleanup verification"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001)
2. Complete Phase 2: US1 XSS Fixes (T002-T006)
3. **STOP and VALIDATE**: Test all XSS rendering paths
4. Security vulnerabilities eliminated — immediate value delivered

### Incremental Delivery

1. Setup → Install dompurify
2. US1 XSS Fixes → Eliminate security vulnerabilities (MVP!)
3. US2 Performance → Smooth streaming experience
4. US3 Resource Cleanup → Stable long-term operation
5. US4 Idle Reduction → Lower background resource usage
6. US5 Config Safety → Protected import/export
7. Polish → Final verification

### Parallel Strategy

With multiple agents/developers:
1. Complete Setup (T001)
2. Launch in parallel:
   - Agent A: US1 (XSS — T002-T006)
   - Agent B: US2 (Performance — T007-T010)
   - Agent C: US3 + US4 (Cleanup + Idle — T011-T017)
   - Agent D: US5 (Config — T018-T022)
3. Final: Polish (T023-T024)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story is independently completable and testable
- No automated test framework exists — all verification is manual
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- T002 and T009 both modify ChatMessage.vue but in different code sections — coordinate if parallelizing US1 and US2
