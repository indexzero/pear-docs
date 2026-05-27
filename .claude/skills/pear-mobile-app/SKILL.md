---
name: pear-mobile-app
description: Build Bare-powered mobile applications for iOS and Android using bare-expo, react-native-bare-kit, and bare-pack — including running a Bare backend as a Worklet beside a React Native UI, bridging with bare-rpc, bundling per-architecture, and linking native addons. Use when the user is starting from `bare-expo`, building a mobile P2P app with Autopass/Hyperswarm, calling `bare-pack` to bundle Bare backend code for ios-arm64 / android-arm64, debugging `ADDON_NOT_FOUND`, or wiring a React Native screen to a Bare backend over RPC. Trigger terms include "bare mobile", "bare-expo", "react-native-bare-kit", "Worklet", "bare-pack", "ios-arm64", "android-arm64", "Autopass mobile", "pear mobile".
metadata:
  tags: pear, bare, mobile, ios, android, react-native, bare-expo, bare-pack, rpc
---

# Bare mobile applications

Mobile Pear apps embed the Bare runtime inside a React Native shell via `react-native-bare-kit`. The UI is React Native; the backend is a Bare program running on a Worklet thread. The two sides talk via `bare-rpc` over an IPC pipe. There is no `pear://` link involved — distribution is the regular App Store / Play Store process; only the runtime is shared with Pear desktop/terminal.

## When to use

- The user clones `bare-expo`, `bare-android`, or `bare-ios` to build a mobile app.
- The user has a P2P backend (Autopass, Hyperswarm, Hypercore) and needs to put a React Native UI in front of it.
- `bare-pack` errors, `ADDON_NOT_FOUND`, missing native addon linking.
- iOS / Android lifecycle: `Bare.suspend`, `Bare.resume`, suspended event.

For terminal or desktop variants of the same backend, switch to those skills. For pure RN/Expo questions unrelated to Bare, this skill does not apply.

## Quick start

```sh
git clone https://github.com/holepunchto/bare-expo myapp-mobile
cd myapp-mobile
npm install
npm i b4a bare-fs bare-rpc corestore autopass @react-native-clipboard/clipboard graceful-goodbye
npm i bare-pack @types/b4a --save-dev
```

Directory layout (after wiring a backend):

```
myapp-mobile/
  app/
    index.tsx              React Native UI (Worklet host + RPC client)
    app.bundle.mjs         OUTPUT of bare-pack — do not edit by hand
  backend/
    backend.mjs            Bare entrypoint (running as a Worklet)
  rpc-commands.mjs         RPC enum shared between UI and backend
  package.json
  ios/                     react-native project
  android/                 react-native project
```

## Bundling the Bare backend

The backend cannot be loaded directly — it must be bundled per-architecture by `bare-pack`:

```sh
npx bare-pack \
  --host ios-arm64 \
  --host ios-arm64-simulator \
  --host ios-x64-simulator \
  --host android-arm64 \
  --host android-x64 \
  --linked \
  --out app/app.bundle.mjs \
  backend/backend.mjs
```

`--linked` includes statically linked native addons. Re-run after any change to `backend/` or its dependencies.

## Wiring UI ↔ backend

`rpc-commands.mjs` (shared enum so both sides agree on opcodes):

```js
export const RPC_RESET   = 0
export const RPC_MESSAGE = 1
export const RPC_INVITE  = 2
```

`backend/backend.mjs` (runs inside the Worklet):

```js
import RPC from 'bare-rpc'
import { IPC } from 'bare-rpc/global'  // pipe injected by react-native-bare-kit
import { RPC_MESSAGE } from '../rpc-commands.mjs'

const rpc = new RPC(IPC, (req) => {
  if (req.command === RPC_MESSAGE) {
    const { text } = req.data
    // do backend work
    req.reply({ ok: true, echoed: text })
  }
})
```

`app/index.tsx` (the RN screen):

```tsx
import { Worklet } from 'react-native-bare-kit'
import bundle from './app.bundle.mjs'
import RPC from 'bare-rpc'
import { RPC_MESSAGE } from '../rpc-commands.mjs'

const worklet = new Worklet()
await worklet.start('/app.bundle', bundle, [])
const rpc = new RPC(worklet.IPC)

async function send(text: string) {
  const res = await rpc.request(RPC_MESSAGE, { text })
  console.log(res)
}
```

## Native addons & ADDON_NOT_FOUND

Bare addons (anything that needs a `.node`/`.dylib`/`.so`) must be **linked at build time** on mobile. After `npm install`:

- iOS — linked from `node_modules/react-native-bare-kit/ios/addons`
- Android — linked from `node_modules/react-native-bare-kit/android/src/main/addons`

If a backend module fails with `AddonError: ADDON_NOT_FOUND: Cannot find addon`:

1. Log `Bare.platform` and `Bare.arch` from inside the backend — confirm what the runtime actually is.
2. Confirm the addon is present in the linked directories above.
3. Clean the build cache (`npx pod-install`, `cd android && ./gradlew clean`) and rebuild.
4. If you're using `bare-expo`, ensure you re-ran `bare-pack --linked` after adding the dependency.

## Lifecycle: suspend / resume

Mobile apps must respect OS lifecycle events. Bare emits:

- `Bare.on('suspend', linger => { /* stop work; close sockets */ })`
- `Bare.on('idle',   () => { /* event loop has drained */ })`
- `Bare.on('resume', () => { /* re-open work */ })`

A backend that keeps the swarm or DHT open during suspend will leak battery and may be killed. Pattern:

```js
Bare.on('suspend', async () => {
  await swarm.flush()
  swarm.destroy()
})
Bare.on('resume', () => {
  swarm = new Hyperswarm()
  // re-join topics
})
```

## Runtime checks

| Want to know… | Read |
| --- | --- |
| iOS vs Android vs simulator | `Bare.platform` (`"ios" | "android"`), `Bare.simulator` |
| CPU arch | `Bare.arch` (`"arm64" | "x64" | ...`) |
| Process suspended? | `Bare.suspended` |

## Logs

- **iOS:** `xcrun simctl spawn booted log stream --predicate 'subsystem == "bare"'` for the simulator; Console.app for devices.
- **Android:** `adb logcat | grep bare`.

## Common pitfalls

- **`fs` / `path` / `process` `is not defined`.** Always use `bare-fs`, `bare-path`, `bare-process` in backend code. Wire via `package.json` `imports` map when reusing Node modules.
- **`bare-pack` fails on `node:crypto`.** Code branched on runtime instead of declaring an `imports` map. `bare-pack` is static — see `pear-node-compat`.
- **Worklet exits silently.** An uncaught exception in the backend; missing `'unhandledRejection'` handler. Add:
  ```js
  Bare.on('uncaughtException', (err) => { console.error('FATAL', err) })
  Bare.on('unhandledRejection', (reason) => { console.error('REJECT', reason) })
  ```
- **HMR doesn't work for the backend.** Backend code is pre-linked; rebuild for changes. RN UI has its own HMR.
- **Storage path.** `Pear.app` is not available in mobile worklets in v1. Use `bare-fs` against an app-private directory passed via worklet args.

## What this skill does not cover

- React Native UI patterns beyond the Worklet boundary — defer to RN docs.
- Building Bare itself from source — see [Bare README](https://github.com/holepunchto/bare).
- App Store / Play Store submission.
