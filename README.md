# Kurpio — UI Prototype

A dark-mode music player built for gamers, created with plain HTML, CSS, and JavaScript so it can be dropped directly into an Electron renderer. No backend, no database—everything runs locally using `localStorage` for playlists, the native `<audio>` element for playback, and the Web Audio API for visualization.

## Files

- `index.html` / `app.js` — Main player window (library, playlists, upload, playback controls).
- `overlay.html` / `overlay.js` — Compact in-game overlay window.
- `style.css` — Shared design system (Kurpio theme: near-black interface, neon accents, minimalist gaming aesthetic).

Open `index.html` in a browser to test it. Click **Overlay** in the playback bar to open the overlay window and watch it synchronize playback state using `postMessage`.

---

## Wiring into Electron

### Main window

```js
const win = new BrowserWindow({
  width: 1180,
  height: 720,
  backgroundColor: "#0a0b0e",
  webPreferences: {
    contextIsolation: true
  }
});

win.loadFile("index.html");
```

### Overlay window

```js
const overlay = new BrowserWindow({
  width: 372,
  height: 130,
  transparent: true,
  frame: false,
  alwaysOnTop: true,
  resizable: false,
  skipTaskbar: true,
  webPreferences: {
    contextIsolation: true
  }
});

overlay.setAlwaysOnTop(true, "screen-saver");
overlay.loadFile("overlay.html");
```

The overlay already uses `-webkit-app-region: drag` so users can reposition it, while playback buttons use `no-drag` to remain clickable.

---

## Replace `postMessage` with IPC

The prototype uses `window.opener.postMessage()` so both windows communicate inside a normal browser.

Inside Electron, replace that with IPC:

- Main window
  ```js
  ipcRenderer.send("now-playing", state);
  ```

- Main process
  ```js
  overlayWin.webContents.send("now-playing", state);
  ```

- Overlay controls
  ```js
  ipcRenderer.send("transport-control", action);
  ```

Forward the action back to the main window to control playback regardless of focus or monitor configuration.

---

## Local Music Library (No Database)

Kurpio stores imported tracks locally without requiring a server or database.

- `preload.js` exposes Electron's `webUtils.getPathForFile()` to the renderer.
- `app.js` saves each song's metadata and original file path into `localStorage`.
- On the next launch, Kurpio automatically loads the saved tracks using their stored `file://` paths.
- Running only `index.html` in a web browser cannot access real filesystem paths because of browser security restrictions, so uploaded files only last until the page reloads.

### Note

If a user moves, renames, or deletes the original music file, the saved path becomes invalid.

For a permanent music library, import each song into:

```
app.getPath("userData")/library/
```

instead of referencing its original location.

---

## Kurpio Design Tokens

| Token | Value | Purpose |
|-------|-------|---------|
| `--bg-void` | `#0a0b0e` | Main application background |
| `--bg-panel` | `#14161b` | Sidebar, panels, playback bar |
| `--primary` | `#5C7CFA` | Kurpio primary accent |
| `--secondary` | `#8B5CF6` | Secondary accent |
| `--success` | `#22C55E` | Playing/active indicators |
| `--warning` | `#F59E0B` | Notifications and highlights |
| Display Font | Chakra Petch | Titles, branding, gaming UI |
| Body Font | Inter | General interface text |
| Mono Font | JetBrains Mono | Timers, counters, technical information |

---

## About Kurpio

Kurpio is a modern desktop music player designed for gamers. It focuses on fast local playback, customizable playlists, a lightweight always-on-top overlay, and a clean interface without requiring cloud accounts or online services.

Everything stays on the user's computer, making Kurpio fast, private, and responsive.
