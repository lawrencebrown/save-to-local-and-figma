# Screenshot Drop — Agent Reconstruction Spec

Complete technical specification for rebuilding this project from scratch. Implement every file exactly as described.

---

## Build Order & Verification

Work in this order. Write each file completely before moving to the next. After each phase, do a quick self-check (listed below) — then continue without stopping unless something is broken.

**Phase 1 — Cloudflare Worker** (`figma-plugin/worker.js`)
Write the full Worker. Self-check: all four routes present, `json()` helper used throughout, `request.json()` for POST body, chunks validated, CORS headers on all responses, TTL on all KV puts.

**Phase 2 — Figma Plugin** (`figma-plugin/manifest.json`, `code.js`, `ui.html`)
Write all three files. Self-check: `code.js` uses `figma.currentPage`, no page-ID logic anywhere. `ui.html` fetches manifest on load, renders list, decodes base64 chunks before posting to `code.js`, deletes after place. Config section uses `figma.clientStorage`.

**Phase 3 — Chrome Extension scaffolding** (`manifest.json`, `offscreen.html`, `offscreen.js`, `icon.png`)
Write the manifest and offscreen files. Self-check: `default_popup` set, all permissions listed, `host_permissions` includes Figma API and Google Fonts domains, `offscreen.html` is one line.

**Phase 4 — background.js**
Write the full service worker. Self-check: `restorePage()` helper called in both success path and catch block, scrollbar hiding uses both CDP and CSS injection, 500ms sleep after each scroll, last segment placed at its `scrollY` not `drawY`, `convertToBlob({ type: 'image/png' })`, chunks stripped of data-URL prefix before upload, `captureState` success written before clipboard/download, Figma URL uses `/design/`.

**Phase 5 — popup.html + popup.js**
Write both files. Self-check: all five state divs present, second tab label is "Send to draft", `#fileUrlOverride` is inside `#panel-saved` only, `saveAs: false` on download, `watchCaptureState()` removes its own listener on success/error, `escapeHtml()` used on draft names, `savedDrafts` capped at 20.

Only pause between phases if something in the self-check fails and you need to fix it. Otherwise write all five phases continuously.

---

## What This Is

A personal-use tool with three components that work together:

1. **Chrome Extension (MV3)** — click the icon on any webpage, pick a destination Figma file, click "Send It →". The extension takes a full-page screenshot and uploads it to a Cloudflare relay.
2. **Cloudflare Worker** — a simple JSON relay with a KV-backed queue. Holds screenshots for up to 24 hours. Accepts and serves multiple screenshots (queue, not single-slot).
3. **Figma Plugin ("Screenshot Drop")** — runs inside Figma. Fetches the queue from the Worker, shows a list, user picks one and clicks "Place Selected". It places the screenshot as a frame on the current Figma page.

---

## Architecture

```
User clicks extension icon
  → popup.html opens (default_popup)
  → popup.js reads chrome.storage.local for config + last state
  → if configured: show picker (project/file cascade from Figma API)
  → user picks file, clicks "Send It →"
  → popup.js sends { type: 'startCapture', tabId, fileKey } to background.js
  → popup switches to CAPTURING state, watches chrome.storage.onChanged

background.js (service worker)
  → attaches Chrome Debugger Protocol to the tab
  → forces 1x DPR via Emulation.setDeviceMetricsOverride
  → hides scrollbars (CDP + CSS injection)
  → scrolls page in viewport-height steps, captures PNG each step
  → stitches segments into OffscreenCanvas
  → slices canvas into 2000px-tall chunks (Figma has 4096px image limit)
  → POSTs chunks to Cloudflare Worker /screenshot
  → writes captureState = { status: 'success', figmaUrl, timestamp } to chrome.storage.local
  → writes pendingFigma = { chunks, fullDataUrl, ... } for "Save to disk"
  → copies full PNG to clipboard via offscreen document

popup.js (storage.onChanged listener)
  → success → show SUCCESS state with "Open in Figma" link
  → error   → show ERROR state with message + retry

--- later, in Figma ---

Figma Plugin (ui.html)
  → GET /screenshot → receives manifest array of pending screenshots
  → shows list, user selects one
  → GET /screenshot/:id → fetches full data
  → DELETE /screenshot/:id (fire and forget)
  → postMessage to code.js with chunks + dimensions

Figma Plugin (code.js)
  → creates container Frame (pageWidth × pageHeight)
  → stacks chunk Frames inside it, each with IMAGE fill
  → appends to figma.currentPage
  → scrolls viewport to show the new frame
```

---

## File Structure

```
browser-screenshot/
  manifest.json           Chrome extension manifest
  popup.html              Extension popup UI
  popup.js                Popup logic
  background.js           Service worker (capture + upload)
  offscreen.html          Minimal HTML for offscreen document
  offscreen.js            Clipboard writer
  icon.png                Extension icon (128×128)

  figma-plugin/
    manifest.json         Figma plugin manifest
    code.js               Plugin main thread
    ui.html               Plugin UI (self-contained HTML with inline JS)
    worker.js             Cloudflare Worker source (deploy manually)
```

---

## manifest.json (Chrome Extension)

```json
{
  "manifest_version": 3,
  "name": "Screenshot Tab",
  "version": "1.0",
  "description": "Full-page screenshot of the current tab, saved with URL + date as filename.",
  "permissions": ["downloads", "debugger", "tabs", "clipboardWrite", "offscreen", "storage", "unlimitedStorage"],
  "host_permissions": [
    "https://api.figma.com/*",
    "https://fonts.googleapis.com/*",
    "https://fonts.gstatic.com/*"
  ],
  "web_accessible_resources": [{ "resources": ["popup.html"], "matches": ["<all_urls>"] }],
  "action": {
    "default_popup": "popup.html",
    "default_title": "Capture full-page screenshot",
    "default_icon": { "128": "icon.png" }
  },
  "background": { "service_worker": "background.js" },
  "icons": { "128": "icon.png" }
}
```

---

## popup.html

Full-page popup UI. Five state divs, only one visible at a time. Controlled entirely by popup.js.

States: `state-setup`, `state-picker`, `state-capturing`, `state-success`, `state-error`.

Design: Geist font (Google Fonts), off-white `#f5f5f5` background, white header, bold black borders (`2px solid #050505`), black pill buttons. Header has 5 coloured dots (red/orange/purple/blue/green), title "Save to Figma & Local", Lucide settings SVG icon.

The `.modal` wrapper carries a brutalist drop shadow: `box-shadow: 4px 4px 0 #050505`. Tab buttons: first tab label is "Projects", second tab label is exactly **"Send to draft"** (not "Saved" or "Saved files"). The "Open in Figma" button uses Figma blue: `background: #18a0fb`. The error box is a bordered card with a pink/red background (`background: #ffe6e6`), not just a plain text block.

All element IDs that popup.js references:
- `#gearBtn` — settings toggle
- `#pat`, `#teamUrl`, `#workerUrl`, `#workerSecret`, `#saveConfigBtn`, `#setup-status` — setup form
- `#tabProjects`, `#tabSaved` — tab toggle buttons
- `#panel-projects`, `#panel-saved` — tab panels
- `#projectSelect`, `#fileSelect` — project/file dropdowns
- `#draft-list` — container for saved draft radio buttons
- `#fileUrlOverride` — paste-a-Figma-URL input
- `#picker-status` — status text below picker
- `#captureBtn` — main CTA (starts disabled)
- `#openFigmaBtn`, `#success-title`, `#success-sub` — success state
- `#saveDiskBtn`, `#sendAgainBtn`, `#newCaptureBtn` — success actions
- `#error-msg`, `#retryBtn`, `#errorNewCaptureBtn` — error state

CSS classes: `.modal` (border wrapper), `.header`, `.header-dots`, `.header-dot`, `.state`, `.tab-toggle`, `.draft-list`, `.draft-list.scrollable`, `.draft-item`, `.btn`, `.btn-outline`, `.btn-ghost`, `.btn-figma`, `.divider`, `.section-label`, `.spinner-wrap`, `.spinner`.

---

## popup.js

### Storage keys read/written

| Key | Type | Purpose |
|-----|------|---------|
| `figmaPat` | string | Figma Personal Access Token |
| `figmaTeamId` | string | Figma Team numeric ID |
| `workerUrl` | string | Cloudflare Worker base URL (no trailing slash) |
| `workerSecret` | string | X-Secret header value |
| `captureState` | object | `{ status, fileKey, figmaUrl, message, timestamp }` |
| `pendingFigma` | object | `{ chunks, fullDataUrl, pageWidth, pageHeight, filename, capturedAt }` |
| `savedDrafts` | array | `[{ key, name, savedAt }]` — Figma files outside projects |
| `lastProjectId` | string | Remembered project selection |
| `lastFileKey` | string | Remembered file selection |

### Key functions

**`init()`** — reads storage, routes to correct state:
- No config → `showState('setup')`
- `captureState.status === 'capturing'` → `showState('capturing')` + `watchCaptureState()`
- `captureState.status === 'success'` AND `Date.now() - timestamp < 5min` → `showSuccess(captureState)`
- `captureState.status === 'error'` → `showError(captureState.message, captureState)`
- Otherwise → `showState('picker')` + `loadSavedDrafts()` + `loadProjects()`

**`loadProjects()`** → `GET https://api.figma.com/v1/teams/${figmaTeamId}/projects` (X-Figma-Token header) → `populateSelect('projectSelect', ...)` → `loadFiles(sel.value)`

**`loadFiles(projectId)`** → `GET https://api.figma.com/v1/projects/${projectId}/files` → `populateSelect('fileSelect', ...)` → enables `captureBtn`

**`populateSelect(selectEl, items, placeholder)`** — when `items` is empty, append a single disabled `<option>` with the placeholder text (e.g. "No projects found") and set `select.disabled = true`. Without this, an empty `<select>` renders as a blank box with no affordance. When items exist, populate normally and set `select.disabled = false`.

**`fetchAndSaveDraft(fileKey)`** → `GET https://api.figma.com/v1/files/${fileKey}?depth=1` → calls `saveDraftFile(key, name)`

**`saveDraftFile(key, name)`** — upserts into `savedDrafts` array in storage. Calls `loadSavedDrafts()` after.

**`loadSavedDrafts()`** — renders radio buttons in `#draft-list`. Adds `.scrollable` class when > 5 items.

**`switchTab(tab)`** — shows/hides `panel-projects` vs `panel-saved`. Clears the opposite side's state. When switching to projects, re-enables `captureBtn` if a file is selected.

**`captureBtn` click handler**:
```js
const fileKey = fileKeyOverride || fileSelect.value;
chrome.storage.local.set({ lastFileKey: fileKey, lastProjectId: projSel.value });
const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
showState('capturing');
watchCaptureState();
chrome.runtime.sendMessage({ type: 'startCapture', tabId: tab.id, fileKey });
```

**`watchCaptureState()`** — sets up `chrome.storage.onChanged` listener. On `captureState.newValue.status === 'success'` → `showSuccess(state)`. On `'error'` → `showError(state.message, state)`.

**CRITICAL**: The listener MUST be removed after it fires via a `stopWatchingCaptureState()` cleanup function. Pattern:
```js
let _captureListener = null;
function watchCaptureState() {
  if (_captureListener) return;
  _captureListener = (changes, area) => {
    if (area !== 'local' || !changes.captureState?.newValue) return;
    const state = changes.captureState.newValue;
    if (state.status === 'success') { stopWatchingCaptureState(); showSuccess(state); }
    if (state.status === 'error')   { stopWatchingCaptureState(); showError(state.message, state); }
  };
  chrome.storage.onChanged.addListener(_captureListener);
}
function stopWatchingCaptureState() {
  if (_captureListener) { chrome.storage.onChanged.removeListener(_captureListener); _captureListener = null; }
}
```
Using a boolean flag like `captureWatcherAttached` without removing the listener causes a leak — the listener fires on every future capture in the same popup session.

**`retryBtn` click** — must trigger a full re-capture via `startCapture()`, NOT `sendAgain()`. If the original capture failed, `pendingFigma` does not exist in storage — calling `sendAgain()` silently does nothing. The retry path must re-attach the debugger and re-capture from scratch.

**`sendAgain()`** — reads `captureState.fileKey`, sends `{ type: 'sendAgain', fileKey }` to background. Only used from the success state (resend already-captured data). Never call this from the error retry path.

**`showSuccess(captureState)`** — sets title "Sent to Figma", sets `#openFigmaBtn` href to `captureState.figmaUrl`. The `figmaUrl` is always `https://www.figma.com/design/${fileKey}` — never `/file/`.

**`saveDiskBtn` click** — reads `pendingFigma.fullDataUrl` from storage, calls `chrome.downloads.download({ url, filename, saveAs: false })`.

**CRITICAL**: `saveAs` MUST be `false`. Setting `saveAs: true` opens the OS file-picker dialog, which closes the popup window before the user sees the success state — they lose the "Open in Figma" link and success screen entirely.

**`fileUrlOverride` input handler** — regex matches Figma URL for file key, sets `fileKeyOverride`, calls `fetchAndSaveDraft(key)`, enables `captureBtn`. The `#fileUrlOverride` input lives inside `#panel-saved` only — it is NOT shown in the Projects tab.

**`draft-list` change handler** — sets `fileKeyOverride = e.target.value`, enables `captureBtn`.

**`escapeHtml(value)`** — MUST exist and be used when rendering draft names as innerHTML. Without it, a Figma file with `<script>` in its name would execute arbitrary code:
```js
function escapeHtml(value) {
  return String(value).replace(/[&<>"']/g, (c) => ({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;' }[c]));
}
```

**Gear button** — toggles setup state. If setup visible, calls `init()`. Otherwise prefills and shows setup.

---

## background.js

### Message handler

Listens for `startCapture` and `sendAgain` messages from popup.

### `captureAndUpload({ tabId, fileKey })`

1. `chrome.storage.local.set({ captureState: { status: 'capturing', fileKey } })`
2. `chrome.debugger.attach(target, "1.3")` + `sleep(600)` (wait for debugger bar)
3. `Page.enable` AND `Runtime.enable` — both must be called. Omitting `Runtime.enable` can cause `Runtime.evaluate` calls to fail silently on some pages.
4. `evalJson` to read `scrollHeight`, `viewportHeight`, `viewportWidth`, `dpr`, `originalY`
5. `Emulation.setDeviceMetricsOverride` with `deviceScaleFactor: 1` + `sleep(150)`
6. Re-read scroll dimensions (DPR change can cause re-layout)
7. Hide scrollbars: `Emulation.setScrollbarsHidden` (try/catch) + inject CSS `*::-webkit-scrollbar{display:none!important}*{scrollbar-width:none!important}`
8. `scrollTo({ top: 0 })` + `sleep(500)`
9. Capture loop: read `scrollY`, capture PNG, advance by `viewportHeight`, `sleep(500)` — stop when `scrollY === prevScrollY` or `scrollY >= maxScrollY`
10. `restorePage(target, scrollbarStyleId, originalY)` — extracted cleanup helper: removes injected `<style>`, calls `Emulation.setScrollbarsHidden(false)` (try/catch), calls `Emulation.clearDeviceMetricsOverride` (try/catch), restores scroll position. Call this in both the success path AND the catch block so cleanup always runs.
11. Stitch: `OffscreenCanvas(viewportWidth, effectiveScrollHeight)`, draw each segment. Last segment: draw at `y` (handles partial overlap). Others: draw at `drawY`.
12. `canvas.convertToBlob({ type: 'image/png' })` → `blobToDataUrl`
13. `buildFilename(tab.url)` — sanitized URL + date + `.png`
14. `sliceCanvas(canvas)` — 2000px-tall chunks
15. `uploadToWorker(chunks, { fileKey, filename, pageWidth, pageHeight }, workerUrl, workerSecret)`
16. `chrome.storage.local.set({ captureState: { status: 'success', fileKey, figmaUrl, timestamp }, pendingFigma: { chunks, fullDataUrl, pageWidth, pageHeight, filename, capturedAt } })`
17. Badge `"✓"` green, clear after 2s
18. `ensureOffscreen()` + send clipboard message

### `sendAgain({ fileKey })`

Reads `pendingFigma` from storage, re-runs `uploadToWorker`, updates `captureState`.

### `uploadToWorker(chunks, payload, workerUrl, workerSecret)`

```js
POST ${workerUrl}/screenshot
Headers: { 'Content-Type': 'application/json', 'X-Secret': workerSecret }
Body: { chunks: [{ imageBase64, width, height }], fileKey, filename, pageWidth, pageHeight }
```

Strip the `data:image/...;base64,` prefix from each chunk's dataUrl before sending.

### `sliceCanvas(canvas)`

Slices into `CHUNK_HEIGHT = 2000` px tall strips. Returns `[{ dataUrl, width, height }]`.

### Helper functions

- `sleep(ms)` — Promise timeout
- `debugEval(target, expression)` — `Runtime.evaluate`, no return value needed
- `evalJson(target, expression)` — wraps in `JSON.stringify(...)`, parses result
- `evalNumber(target, expression)` — `Runtime.evaluate`, returns `result.value`
- `blobToDataUrl(blob)` — arrayBuffer → Uint8Array → binary string → btoa, chunked at 8192 bytes
- `buildFilename(url)` — sanitize URL, append date, trim to 120 chars, add `.png`
- `ensureOffscreen()` — creates offscreen document for clipboard (catches errors silently)

---

## offscreen.html + offscreen.js

`offscreen.html` — one line: `<!DOCTYPE html><script src="offscreen.js"></script>`

`offscreen.js` — listens for messages with `target: "offscreen"`:
- `type: "clipboard"` → fetch dataUrl as blob → `navigator.clipboard.write([new ClipboardItem({ "image/png": blob })])`

---

## figma-plugin/manifest.json

```json
{
  "name": "Screenshot Drop",
  "id": "screenshot-drop-personal",
  "api": "1.0.0",
  "editorType": ["figma"],
  "main": "code.js",
  "ui": "ui.html",
  "networkAccess": { "allowedDomains": ["https://*.workers.dev"] }
}
```

---

## figma-plugin/code.js

```js
figma.showUI(__html__, { width: 300, height: 360 });

figma.ui.onmessage = async (msg) => {
  if (msg.type === 'getConfig') → read workerUrl + workerSecret from figma.clientStorage → postMessage { type: 'config', workerUrl, workerSecret }
  if (msg.type === 'saveConfig') → save both to figma.clientStorage → postMessage { type: 'configSaved' }
  if (msg.type === 'placeImage'):
    - destructure { chunks, filename, pageWidth, pageHeight }
    - create container Frame: resize(pageWidth, pageHeight), fills=[], clipsContent=true, name=filename
    - loop chunks: createImage(Uint8Array(imageBytes)), createFrame(width, height), fill=IMAGE FILL, append to container, offsetY += height
    - figma.currentPage.appendChild(container)
    - figma.viewport.scrollAndZoomIntoView([container])
    - postMessage { type: 'placed' }
};
```

---

## figma-plugin/ui.html

Self-contained single-file HTML with inline CSS and JS.

**States:**
- Config section (first run / editing): Worker URL + Secret inputs + Save button
- Main section: queue list + Place Selected button + status + Refresh + Edit config link

**Flow:**
1. On load: `postMessage getConfig` → receives config → if configured, `showMain()` + `loadQueue()`
2. `loadQueue()`: `GET /screenshot` → parse manifest array → render sorted-by-timestamp list with radio buttons + time-ago + × delete button
3. Click row → `selectedId = item.id`, enable Place button
4. × delete → `DELETE /screenshot/:id` → `loadQueue()`
5. Place Selected → `GET /screenshot/:id` → decode base64 chunks → `DELETE /screenshot/:id` (fire and forget) → `postMessage placeImage`
6. On `placed` message → show "Placed ✓", reload queue

**`timeAgo(ts)`** — returns "just now" / "Xm ago" / "Xh ago" / "Xd ago"

---

## figma-plugin/worker.js (Cloudflare Worker)

### KV storage schema

- `__manifest__` → JSON array of `[{ id, filename, pageWidth, pageHeight, timestamp }]`
- `screenshot:{uuid}` → full JSON body from POST (chunks + metadata)

Both keys have TTL = 86400 seconds (24 hours).

### Routes (all require `X-Secret` header matching `env.SECRET_KEY`)

| Method | Path | Action |
|--------|------|--------|
| POST | `/screenshot` | Generate UUID, store full data + update manifest, return `{ id }` |
| GET | `/screenshot` | Return manifest array |
| GET | `/screenshot/:id` | Return full screenshot data |
| DELETE | `/screenshot/:id` | Delete screenshot + remove from manifest |

CORS headers on all responses: `Access-Control-Allow-Origin: *`, methods: GET/POST/DELETE/OPTIONS.

**Implementation notes:**
- Parse the POST body with `await request.json()` directly — do NOT use `await request.text()` followed by `JSON.parse()`.
- Validate that `env.SCREENSHOTS` and `env.SECRET_KEY` exist at the top of every request; return a clear error if missing.
- Validate that `body.chunks` is a non-empty array in the POST handler — return 400 if missing.
- Use a `json(data, status = 200)` helper that sets both `Content-Type: application/json` and CORS headers — avoids duplicating headers on every response.
- Use `crypto.randomUUID()` for screenshot IDs (built into Workers runtime, no import needed).

---

## Data Flow Summary

```
Extension popup.js
  → chrome.runtime.sendMessage({ type: 'startCapture', tabId, fileKey })

background.js
  → captures, stitches, slices
  → POST /screenshot { chunks: [{imageBase64, width, height}], fileKey, filename, pageWidth, pageHeight }
  → writes captureState + pendingFigma to chrome.storage.local

Cloudflare Worker KV
  → manifest: [{ id, filename, pageWidth, pageHeight, timestamp }]
  → screenshot:{id}: { chunks, fileKey, filename, pageWidth, pageHeight }

Figma Plugin ui.html
  → GET /screenshot → show queue list
  → GET /screenshot/:id → full data
  → DELETE /screenshot/:id
  → postMessage { type: 'placeImage', chunks: [{imageBytes, width, height}], filename, pageWidth, pageHeight }

Figma Plugin code.js
  → places on figma.currentPage as stacked Frame structure
```

---

## Key Implementation Constraints

- `default_popup` in manifest means `chrome.action.onClicked` never fires — capture must be triggered by message from popup
- Screenshot stitching: last segment is placed at its actual `scrollY` position (handles partial overlap); all other segments placed at `drawY` (accumulated bitmap heights)
- Canvas `convertToBlob({ type: 'image/png' })` for lossless quality — never use UPNG or JPEG
- Scrollbar hiding: use BOTH `Emulation.setScrollbarsHidden` (wrapped in try/catch) AND CSS injection with `*::-webkit-scrollbar{display:none!important}` — neither alone is reliable
- Per-scroll sleep of 500ms is required for Mac overlay scrollbars to fade before capture
- `captureState` success must be written to storage BEFORE any download is triggered — prevents OS save dialog from closing the popup before it shows the success state
- `chrome.downloads.download` must use `saveAs: false` — `saveAs: true` opens the OS file-picker which instantly closes the popup window
- Figma URL must use `/design/` path: `https://www.figma.com/design/${fileKey}` — the older `/file/` path still works but is deprecated and will eventually break
- `watchCaptureState()` MUST clean up its own listener after success or error — a bare boolean flag without `removeListener()` causes the listener to accumulate and fire on every future capture
- `escapeHtml()` MUST be used for any draft file name rendered via innerHTML — Figma file names are user-controlled and can contain `<script>` tags
- `savedDrafts` array should be capped at 20 entries — `savedDrafts.slice(0, 20)` after unshift
- Chunk base64 prefix (`data:image/png;base64,`) must be stripped before sending to Worker
- `blobToDataUrl` must chunk the binary string at 8192 bytes to avoid stack overflow on large images
- Figma image stacking: each chunk is its own child Frame within a container Frame, not a single image node — this avoids Figma's 4096px per-image dimension limit
- `restorePage(target, styleId, originalY)` should be extracted as a helper called in both the success path and the error catch block — avoids duplicating cleanup code and ensures scrollbars/overrides are always restored even if capture fails mid-way
