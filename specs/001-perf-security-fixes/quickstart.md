# Quickstart: Performance & Security Hardening

**Branch**: `001-perf-security-fixes` | **Date**: 2026-03-31

## Overview

This feature addresses 5 XSS vulnerabilities, 3 critical performance bottlenecks, and 4 resource leak bugs across the Anywhere uTools plugin. Changes span 3 sub-projects: `Anywhere_window` (chat UI), `Anywhere_main` (settings UI), and `backend` (Node.js preload scripts).

## Modified Files Summary

### Security Fixes (XSS)
| File | Change |
|------|--------|
| `Anywhere_window/src/components/ChatMessage.vue` | Fix sanitization pipeline: remove `&gt;→>` replacement, use UUID placeholders, remove `style` from allowed attrs |
| `Fast_window/append_selector.html` | Replace `innerHTML` with safe DOM API for prompt names |
| `Anywhere_main/src/App.vue` | Add DOMPurify sanitization to `marked.parse()` output |
| `Anywhere_main/src/components/Prompts.vue` | Change `v-html` to `v-text` for prompt descriptions |

### Performance Fixes
| File | Change |
|------|--------|
| `Anywhere_window/src/App.vue` | Throttle MutationObserver with rAF, replace deep watcher with targeted approach, clear autoSaveInterval |
| `Anywhere_window/src/components/ChatMessage.vue` | Debounce `renderedMarkdownContent` during streaming |

### Resource Leak Fixes
| File | Change |
|------|--------|
| `Anywhere_window/src/utils/TextSearchUI.js` | Fix `destroy()` to remove all global listeners |
| `backend/src/mcp_builtin.js` | Auto-cleanup completed background shell entries |
| `backend/src/mcp.js` | Use `{ once: true }` for abort signal listeners |
| `backend/src/preload.js` | Reduce scheduler polling to 15s |

### Config Safety
| File | Change |
|------|--------|
| `Anywhere_main/src/components/Setting.vue` | Add pre-import backup, mask API keys in exports |
| `backend/src/data.js` | Add `backupConfig()` backend function |

## Development Setup

```bash
# The project has 3 sub-projects, each with their own package.json
cd Anywhere_main && pnpm install && cd ..
cd Anywhere_window && pnpm install && cd ..
cd backend && pnpm install && cd ..

# Add dompurify to Anywhere_main (currently only in Anywhere_window)
cd Anywhere_main && pnpm add dompurify && cd ..
```

## Testing Approach

### XSS Testing
Test each rendering path with payloads:
- `<script>alert(1)</script>`
- `<img src=x onerror=alert(1)>`
- `<svg onload=alert(1)>`
- `<div style="position:fixed;z-index:99999;width:100vw;height:100vh;background:red">`
- Content containing literal `__PROTECTED_CONTENT_0__`

### Performance Testing
- Open a chat with 50+ messages
- Trigger a streaming response with 500+ words and 3+ code blocks
- Observe for UI freezing/jank during streaming
- Compare frame rate before/after changes

### Resource Leak Testing
- Open chat window → use text search → close search → verify no orphaned listeners via DevTools
- Close chat window → verify autoSaveInterval is cleared
- Run background shells → wait for completion → verify cleanup after 5 minutes

### Config Safety Testing
- Export config → verify API keys are masked
- Import config → verify backup exists before overwrite
- Restore from WebDAV → verify backup exists before overwrite
