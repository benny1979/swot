# Handoff: DrawPen iPad Remote

Parked here only because the DrawPen fork did not exist yet. Unrelated to the swot repo; delete this folder once the patch has been applied to the fork.

## Goal
Annotate live over a Windows screen from an iPad with Apple Pencil. The annotations are drawn on the Windows machine (an overlay), and the iPad controls them.

- Use: presenting in a room. One iPad, personal use (so GPL code would be fine, though none is used).
- Connection: over the internet by default, so the iPad doesn't need to be on the same network as the PC. Paired with a QR code.
- Content: mostly static (slides, documents), so screenshots are used instead of live video.

## Decisions (agreed with the user)
- Build on **DrawPen** (github.com/DmytroVasin/DrawPen, MIT licence, Electron + React, active). It already has the overlay, tools, undo, multi-monitor support and screen capture.
- Fork it to `benny1979/DrawPen`. **Not done yet:** the cloud session could not fork it. The user must fork it on GitHub.
- Relay: a **Cloudflare Worker + Durable Object** (free plan), which the user agreed to. Both devices connect outward to it.
- iPad side: a **Safari web page** served by the relay, not a native app (no App Store or developer account needed).
- The iPad shows a JPEG screenshot about once a second instead of video.
- DrawPen's auto-update is disabled in the fork, because official releases would replace the build and remove the feature.

## Status
- Code is in `0001-ipad-remote.patch` (one commit on top of upstream `899dbdc "Update Changelog"`, 26 Sep 2026).
  Apply it: clone the fork, then `git am path/to/0001-ipad-remote.patch`.
- Tested in the cloud: the relay (`wrangler dev`) and the iPad page, end to end in headless Chromium with a simulated PC (Node WebSocket).
  - Passed: pairing, tool/colour/width messages, down/move/up, frame JPEG display, local preview removed when a frame with a matching `seq` arrives, undo, overlay toggle, "PC offline" message.
  - main + renderer webpack bundles compile without errors.
- **Not tested:** Electron on Windows, a real iPad with Pencil, and deploying to Cloudflare.

## Architecture
```
DrawPen (Windows, Electron)        Relay (CF Worker + DO)            iPad (Safari page)
 src/main/remote.js  <---wss--->  remote/relay/src/index.js <---wss---> remote/relay/public/index.html
 - forwards input to renderer      - one DO per room id               - Pencil input, local preview ink
 - screenshots every 1s + after    - 1 host + 1 pad per room,          - toolbar (tools/colours/widths/
   each stroke (desktopCapturer,     newcomer replaces old               undo/redo/clear/overlay/board/
   cropped to workArea, <=1600px,  - peer presence messages              finger/refresh)
   JPEG q65, md5 dedupe)           - ping/pong auto-response           - wakeLock, PWA meta
```

### Files changed in the DrawPen fork
- `src/main/remote.js` (new): `createRemoteBridge()`. Handles the WebSocket to the relay (Node global WebSocket), reconnect backoff, ping every 20 s, room id (32 hex characters, stored), QR code (`qrcode` npm package), and screenshot capture/send.
- `src/main/index.js`:
  - Store keys `remote_enabled`, `remote_relay_url`, `remote_room_id`.
  - `remoteOverlay` flag, `showRemoteOverlay` / `hideRemoteOverlay`. The overlay is shown with `setIgnoreMouseEvents(true)` and `showInactive`, so the PC stays usable.
  - `enablePointerMode` keeps the overlay visible and click-through when `remoteOverlay` is set. `enableDrawMode` calls `setIgnoreMouseEvents(false)`. `hideApp` clears `remoteOverlay`.
  - Tray items "iPad Remote..." and "Hide iPad Overlay".
  - Pairing window `createRemoteWindow`.
  - IPC handlers `remote_get_state`, `remote_set_relay_url`, `remote_set_enabled`, `remote_new_room`.
  - `updateApp()` returns early (auto-update off).
- `src/renderer/app_page/components/Application.js`: an `onRemoteInput` listener calls the existing `handleMouseDown`, `handleMouseMove`, `handleMouseUp`, `handleChangeTool`, `handleChangeColor` and `handleChangeWidth` through a ref, each inside `flushSync`, so every event sees fresh state the way real pointer events do. Undo/redo pass a synthetic Ctrl+Z / Ctrl+Shift+Z to `handleKeyDown`.
- `constants.js`: `remoteToolList`. `app_page/preload.js`: `onRemoteInput`.
- `src/renderer/remote_page/*` (new): pairing window with relay URL, Connect, QR code and "New pairing code".
- `tools/forge/forge.config.js`: `remote_window` entry point.
- `remote/relay/*` (new): `wrangler.toml`, Worker, iPad page, README. `remote/README.md` has the setup steps and protocol.

### Protocol (JSON text; screenshots are binary JPEG, each after a `frame` message)
- iPad → PC:
  - `down {p:[x,y]}`, `move {pts:[[x,y]...]}`, `up {p, seq}`. Coordinates are normalised 0..1 to the drawing area (the work area of the monitor DrawPen is on, i.e. the screen minus the taskbar).
  - `tool {tool}`, `color {index}` (0 = rainbow, 1..6 = palette), `width {index 0..3}`, `undo`, `redo`.
  - `clear`, `whiteboard`, `overlay {visible}`, `frame` (request a screenshot), `hi`.
- PC → iPad: `hello {w, h, colors, overlay}`, `overlay_state {visible}`, `frame {seq, w, h}` followed by the JPEG.
- Relay: `peer {role, connected}`. Clients send text `ping` and the relay replies `pong` without waking the Durable Object.

## User setup (once the fork exists)
1. `cd remote/relay && npm install && npx wrangler login && npx wrangler deploy` → note the workers.dev URL.
2. In the repo root: `npm install`, then `npx electron-forge start`.
   - Not `npm start`: it uses Unix-style env syntax and opens a 500x500 development window.
3. Tray → iPad Remote... → paste the URL → Save → Connect → scan the QR code with the iPad.

## Most likely to need fixing after first real test
1. Focus: after leaving draw mode with the overlay kept, `mainWindow.blur()` may not return keyboard focus to PowerPoint on Windows. Alternatives: `setFocusable(false)` while in overlay mode, or focus the previous window.
2. CPU: `desktopCapturer.getSources` every second at full resolution. If heavy, capture less often or at a smaller thumbnail size.
3. The first iPad stroke can be lost if `updateMainWindowPosition` reloads the renderer because the monitor changed.
4. Pencil pressure is ignored (DrawPen has no pressure support).
5. The iPad local-preview width is approximate (simple quadratic smoothing, not perfect-freehand).

## Constraints
- DrawPen `AGENTS.md`: agents must never run npm scripts such as start or package_no_sign.
- Pairing link = full access (can see the screen and draw). "New pairing code" revokes it.
- The relay WebSocket message limit (about 1 MiB) is the reason for the 900 KB JPEG cap.

## User preferences
Blunt and concise; say so if you disagree; ask clarifying questions before big steps; don't create PRs unless asked.
