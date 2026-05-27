---
name: pear-desktop-app
description: Build, run, and debug Pear desktop applications using pear-electron, pear-bridge, the `pear.gui` config, HTML/JS UI, and the v2 entrypoint pattern (Runtime + Bridge). Use when the user runs `pear init ui`, edits an HTML/JS UI inside a Pear app, configures `pear.gui` (height, width, backgroundColor), uses `<pear-ctrl>` for window controls, wires live reload with `Pear.updates(() => Pear.reload())` or `pear-hotmods`, or migrates a Pear v1 desktop app to v2. Trigger terms include "pear desktop", "pear-electron", "pear-bridge", "pear init ui", "pear.gui", "pear-ctrl", "pear migrate v2", "Pear.Window", "Pear.View".
metadata:
  tags: pear, desktop, electron, pear-electron, pear-bridge, ui, gui
---

# Pear desktop applications

Pear desktop apps render their UI in Electron, but the application code is a Pear (Bare) program that *boots* Electron via `pear-electron` and exposes an HTTP-over-pipe surface via `pear-bridge`. This is the v2 pattern. Pear v1 apps booted directly into Electron — see "Migration" below.

## When to use

- `pear init ui` or any project where `pear.type === "desktop"` (or unset with an HTML entrypoint).
- Editing `ui/index.html`, `ui/app.js`, or any browser-visible code in a Pear app.
- Wiring `pear.gui` settings (window size, background, frame).
- Live reload (`Pear.updates(() => Pear.reload())` or `pear-hotmods`).
- Migrating from `Pear.Window`/`Pear.View`/`Pear.tray` (v1) to the `ui` import from `pear-electron` (v2).

For headless apps, switch to `pear-terminal-app`. For mobile, `pear-mobile-app`.

## Quick start

```sh
mkdir myapp && cd myapp
pear init ui --yes
npm install
pear run --dev .
```

Generated files:

```
myapp/
  package.json
  index.js                  # boots pear-electron + pear-bridge
  ui/index.html             # the actual UI
  ui/app.js                 # UI-side JS (Hyperswarm, etc.)
  test/index.test.js
```

`package.json`:

```jsonc
{
  "name": "myapp",
  "main": "index.js",
  "type": "module",
  "pear": {
    "name": "myapp",
    "type": "desktop",
    "pre":  "pear-electron/pre",
    "gui":  { "height": 600, "width": 800, "backgroundColor": "#1F2430" }
  },
  "dependencies": {
    "pear-electron": "^1",
    "pear-bridge":   "^1"
  }
}
```

## The v2 entrypoint pattern

`index.js` is small and consistent across apps:

```js
import Runtime from 'pear-electron'
import Bridge  from 'pear-bridge'

const bridge = new Bridge()
await bridge.ready()

const runtime = new Runtime()
const pipe = await runtime.start({ bridge })
pipe.on('close', () => Pear.exit())
```

- `pear-bridge` runs a localhost HTTP server backed by the application drive.
- `pear-electron` spawns Electron, which loads `index.html` through the bridge.
- The bridge supports SPA routing via `pear.routes`.

## A complete desktop chat

`ui/index.html`:

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>chat</title>
    <style>
      body { margin: 0; font-family: system-ui; background: #1F2430; color: #fff; }
      header { -webkit-app-region: drag; padding: 8px; display: flex; align-items: center; gap: 8px; }
      header > * { -webkit-app-region: no-drag; }
      #log { padding: 12px; height: calc(100vh - 96px); overflow: auto; }
      #send { padding: 8px 12px; display: flex; gap: 8px; }
      input { flex: 1; padding: 6px; }
    </style>
  </head>
  <body>
    <header>
      <pear-ctrl></pear-ctrl>
      <button id="create">create</button>
      <input id="topic" placeholder="topic" />
      <button id="join">join</button>
    </header>
    <div id="log"></div>
    <form id="send"><input id="msg" autocomplete="off" /><button>send</button></form>
    <script type="module" src="./app.js"></script>
  </body>
</html>
```

`ui/app.js`:

```js
import Hyperswarm from 'hyperswarm'
import crypto from 'hypercore-crypto'
import b4a from 'b4a'

const swarm = new Hyperswarm()
Pear.teardown(() => swarm.destroy())
Pear.updates(() => Pear.reload())  // dev: reload on file changes

const log = document.getElementById('log')
const append = (line) => {
  const div = document.createElement('div'); div.textContent = line
  log.appendChild(div); log.scrollTop = log.scrollHeight
}

swarm.on('connection', peer => {
  peer.on('data', d => append(`${b4a.toString(peer.remotePublicKey, 'hex').slice(0, 6)}: ${d}`))
  peer.on('error', () => {})
})

document.getElementById('create').onclick = () => {
  const topic = crypto.randomBytes(32)
  swarm.join(topic, { client: true, server: true })
  document.getElementById('topic').value = b4a.toString(topic, 'hex')
}
document.getElementById('join').onclick = () => {
  const t = document.getElementById('topic').value.trim()
  swarm.join(b4a.from(t, 'hex'), { client: true, server: true })
}
document.getElementById('send').onsubmit = (e) => {
  e.preventDefault()
  const m = document.getElementById('msg').value
  for (const peer of swarm.connections) peer.write(m)
  append(`me: ${m}`)
  document.getElementById('msg').value = ''
}
```

## Window controls

`<pear-ctrl></pear-ctrl>` is the cross-platform window button widget provided by `pear-electron`. To get a draggable titlebar, set `-webkit-app-region: drag` on the parent and `-webkit-app-region: no-drag` on any clickable children. Without this, the window cannot be moved (on macOS this is especially obvious).

## Live reload

- **Built-in:** `Pear.updates(() => Pear.reload())` — fires when the drive updates. In dev this is bundle-level, not module-level.
- **Hot module reload:** `pear-hotmods` for module-granular swaps without losing state. Framework-agnostic; integrates with Vue, React, Alpine, etc.

## Configuration cheat sheet

```jsonc
"pear": {
  "type": "desktop",
  "pre":  "pear-electron/pre",
  "routes": ".",                 // SPA: all paths render entrypoint
  "unrouted": ["/api/*"],

  "gui": {
    "height": 720,
    "width":  1280,
    "backgroundColor": "#1F2430",
    "userAgent": "myapp",        // v2: was pear.userAgent in v1
    "closeHides": true           // v2: was pear.gui.hideable in v1
  },

  "links": {
    "host": "https://api.example.com"   // outbound allowlist (HTTP/S blocked by default)
  },

  "assets": {
    "ui": {
      "link": "pear://<fork>.<length>.<key>",
      "name": "Pear Runtime",
      "only": ["/boot.bundle", "/by-arch/%%HOST%%", "/prebuilds/%%HOST%%"]
    }
  }
}
```

See `pear-electron` README for the full `gui` schema (`pear://pear-electron`).

## Migrating from v1

1. `package.json` `main` must be unset or end in `.js`. Keep `index.html` (v1 fallback) and add `index.js` (v2 entry).
2. `npm install pear-electron pear-bridge`.
3. Add `"pre": "pear-electron/pre"`.
4. Add the v2 `index.js` boot block (above).
5. Rename APIs:
   - `Pear.config` → `Pear.app`
   - `Pear.reload` → `location.reload`
   - `Pear.Window` / `Pear.View` / `Pear.tray` / `Pear.badge` / `Pear.media` → `import ui from 'pear-electron'` then `ui.Window`, `ui.View`, `ui.app.tray`, `ui.app.badge`, `ui.media`
   - `Pear.worker.run` / `pipe` → `pear-run` / `pear-pipe`
   - `Pear.message` / `messages` / `wakeups` / `updates` → `pear-message` / `pear-messages` / `pear-wakeups` / `pear-updates`
6. Config renames:
   - `pear.userAgent` → `pear.gui.userAgent`
   - `pear.gui.hideable` → `pear.gui.closeHides`
   - When supporting both v1 and v2 simultaneously, include both forms.
7. Verify with `pear run --dev .`. If `pear -v` prints `2.x.x` you're on v2.

If migration must be incremental, opt in to compat mode in the HTML head and in any worker entrypoint:

```html
<script>Pear.constructor.COMPAT = true</script>
```

```js
Pear.constructor.COMPAT = true
```

Compat mode silences deprecation warnings and extends API lifetimes — but it also blocks new APIs. Migrate fully when you can.

## Common pitfalls

- **`fetch` to external hosts returns network errors.** HTTP/HTTPS is blocked by default; add the host to `pear.links`.
- **The window doesn't drag on macOS.** Missing `-webkit-app-region: drag` on a parent that wraps `<pear-ctrl>`.
- **`Bare is not defined` / `require.addon is not a function` in the UI.** Desktop UIs run in Electron (Node-based), not Bare. Bare-only addons must be invoked from the Bare side of the app, not the UI. Pear v2 unifies this with a "Pear-end" worker pattern; see Pear v2 examples.
- **Closing the app does not stop the process.** Something kept the loop alive — typically an undestroyed swarm or open core. Add `Pear.teardown` handlers.
- **`npm install` fails or breaks the dev session.** Close the running dev session before installing.

## Build / publish

Same as terminal apps:

```sh
npm prune --omit=dev
pear stage production
pear release production
pear seed production
```

For installable binaries (`.dmg`, `.msi`, etc.), see [`pear-appling`](https://github.com/holepunchto/pear-appling/). The binary is a thin shell that bootstraps Pear and the app drive — it almost never needs rebuilding.
