# Signal Deck — UI prototype

A dark-mode music player UI for gamers, built as plain HTML/CSS/JS so you can
drop it straight into an Electron renderer. No backend, no database —
everything lives in the browser (`localStorage` for playlist metadata, native
`<audio>` + Web Audio API for playback and the waveform visualizer).

## Files

- `index.html` / `app.js` — the main player window (library, playlists, upload, transport bar).
- `overlay.html` / `overlay.js` — the compact in-game overlay window.
- `style.css` — shared design tokens ("Signal Deck" theme: near-black panels, amber HUD accent, HUD corner brackets as the signature motif).

Open `index.html` in a browser to try it standalone. Click "Overlay" in the transport bar to pop the overlay window and see it sync (title, play state) via `postMessage`.

## Wiring into Electron

**Main window**
```js
const win = new BrowserWindow({
  width: 1180, height: 720,
  backgroundColor: '#0a0b0e',
  webPreferences: { contextIsolation: true }
});
win.loadFile('index.html');
```

**Overlay window** — the part that makes it a real gaming overlay:
```js
const overlay = new BrowserWindow({
  width: 372, height: 130,
  transparent: true,
  frame: false,
  alwaysOnTop: true,
  resizable: false,
  skipTaskbar: true,
  webPreferences: { contextIsolation: true }
});
overlay.setAlwaysOnTop(true, 'screen-saver'); // stay above fullscreen games
overlay.loadFile('overlay.html');
```
`-webkit-app-region: drag` is already set on the overlay card so users can
reposition it; buttons inside opt out with `no-drag` so they stay clickable.

**Replace `postMessage` with IPC.** The prototype uses `window.opener.postMessage`
so the two windows can talk to each other in a plain browser tab. In Electron,
swap that for `ipcRenderer`/`ipcMain` so it works regardless of window focus
and multi-display setups:
- main window → `ipcRenderer.send('now-playing', state)` on every play/pause/track change
- main process relays it → `overlayWin.webContents.send('now-playing', state)`
- overlay's `prev/play/next` buttons → `ipcRenderer.send('transport-control', action)` → forwarded back to the main window

## Local file uploads without a database

Uploaded tracks now persist across app restarts — no database, no copying
files around, just remembering where they live:

- On upload, `preload.js` exposes Electron's `webUtils.getPathForFile()` to
  the renderer (the modern, secure replacement for the old `File.path`
  property). `app.js` resolves each uploaded file's real path and saves it
  alongside the track's metadata in `localStorage`.
- On the next launch, any track with a saved path is pointed straight at
  `file://<path>` and is playable immediately — no re-upload needed.
- If you open `index.html` in a plain browser tab (no Electron, no
  `preload.js` bridge) there's no way to get a real filesystem path, so those
  tracks fall back to a session-only blob URL and won't survive a reload.
  That's a browser security limitation, not a bug — it only matters if you're
  testing outside the actual Electron app.

One caveat worth knowing: if the user moves, renames, or deletes the original
file on disk, the saved path breaks and that track can no longer play. If you
want the library to survive that too, copy files into
`app.getPath('userData')/library/` on import instead of just remembering
their original location — happy to add that if you want it.

## Theme tokens (for extending the UI)

| Token | Value | Use |
|---|---|---|
| `--bg-void` | `#0a0b0e` | app background |
| `--bg-panel` | `#14161b` | panels, sidebar, transport bar |
| `--amber` | `#ffb454` | primary accent — active states, waveform, HUD brackets |
| `--violet` | `#8c7bff` | secondary accent, available for future features |
| Display font | Chakra Petch | headings, track titles — condensed/technical, HUD feel |
| Body font | Inter | everything else |
| Mono font | JetBrains Mono | timestamps, counts, status text |
