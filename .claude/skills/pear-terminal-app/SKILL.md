---
name: pear-terminal-app
description: Build, run, and debug Pear terminal applications on the Bare runtime using bare-tty, bare-readline, bare-process, and the Pear lifecycle (Pear.app, Pear.teardown, Pear.updates). Use when the user is initialising a terminal-only Pear project (`pear init --type terminal`), wiring stdin/stdout with raw-mode input, accepting CLI args via `Pear.app.args`, running a long-lived swarm/listener from the terminal, debugging a terminal app with `pear-inspect`, or migrating a Node CLI to Bare. Trigger terms include "pear terminal app", "bare cli", "bare-readline", "bare-tty", "pear init --type terminal", "headless pear app", "pear run --dev ." in a terminal context.
metadata:
  tags: pear, bare, terminal, cli, bare-readline, bare-tty, headless
---

# Pear terminal applications

Terminal Pear apps run on Bare, not Node. Standard input/output is wired via `bare-tty`, line editing via `bare-readline`, signals via `bare-signals`. Networking via Hyperswarm + Hypercore as usual.

## When to use

- The user runs `pear init --type terminal` or asks for a "headless" / "CLI" Pear app.
- The app reads stdin interactively (chat, REPL) or accepts piped input.
- Long-lived process: swarm peers, log replicator, daemon, watcher.
- Debugging a terminal app with `pear-inspect`.

For desktop or mobile apps, switch to `pear-desktop-app` or `pear-mobile-app`. For staging/release/seeding, switch to `pear-staging-release`.

## Quick start

```sh
mkdir myapp && cd myapp
pear init --yes --type terminal
npm install
npm i bare-readline bare-tty bare-process hyperswarm b4a hypercore-crypto
pear run --dev .
```

Generated files: `package.json`, `index.js`, `test/index.test.js`. `package.json` has `pear.type === "terminal"`.

## A complete, runnable terminal chat

```js
// index.js
const readline = require('bare-readline')
const tty      = require('bare-tty')
const crypto   = require('hypercore-crypto')
const b4a      = require('b4a')
const Hyperswarm = require('hyperswarm')

const swarm = new Hyperswarm()
Pear.teardown(() => swarm.destroy())

// last positional argument is the topic; create a new one if missing
const topicArg = Pear.app.args[Pear.app.args.length - 1]
const topic = topicArg && /^[0-9a-f]{64}$/.test(topicArg)
  ? b4a.from(topicArg, 'hex')
  : crypto.randomBytes(32)

swarm.join(topic, { client: true, server: true })

swarm.on('connection', peer => {
  const name = b4a.toString(peer.remotePublicKey, 'hex').slice(0, 6)
  peer.on('data', d => process.stdout.write(`${name}> ${d}`))
  peer.on('error', () => {})
})

const rl = readline.createInterface({
  input:  new tty.ReadStream(0),
  output: new tty.WriteStream(1)
})
rl.input.setMode(tty.constants.MODE_RAW)
rl.on('data', line => {
  for (const peer of swarm.connections) peer.write(line + '\n')
})
rl.on('close', () => Pear.exit())

console.log('topic:', b4a.toString(topic, 'hex'))
```

Run as two peers:

```sh
# terminal A
pear run --dev .
# copy the topic, then in terminal B (or a second machine)
pear run --dev . <topic>
```

## Project layout

```
myapp/
  package.json           pear.type === "terminal"
  index.js               entrypoint
  test/index.test.js     scaffold; brittle by default
```

`package.json` minimum:

```jsonc
{
  "name": "myapp",
  "main": "index.js",
  "type": "module",          // optional; CJS works equally well
  "pear": {
    "name": "myapp",
    "type": "terminal"
  }
}
```

## Lifecycle patterns

- **Pass arguments:** `pear run --dev . --port 4000` → `Pear.app.args === ['--port', '4000']`. Parse with your own loop or `paparam`.
- **Teardown:** register everything that needs closing.
  ```js
  Pear.teardown(() => swarm.destroy())
  Pear.teardown(() => core.close())
  ```
- **Persistent state across restarts:** `Pear.checkpoint({ lastSeen: id })`. Reread next start via `Pear.app.checkpoint`.
- **Exit:** `Pear.exit(code)` runs teardown handlers. `Bare.exit(code)` does not.
- **Detect dev mode:** `Pear.app.dev` (truthy when started with `--dev`). Use this to gate the `pear-inspect` block.

## Reading stdin correctly

`bare-readline.createInterface` requires `bare-tty` streams; passing Node-style `process.stdin` fails. Always:

```js
const tty = require('bare-tty')
const readline = require('bare-readline')

const rl = readline.createInterface({
  input:  new tty.ReadStream(0),
  output: new tty.WriteStream(1)
})
rl.input.setMode(tty.constants.MODE_RAW)   // essential for char-by-char
```

For piped input (not a TTY), check `tty.isatty(0)` and fall back to reading from `bare-process.stdin` directly.

## Debugging

```js
if (Pear.app.dev) {
  const { Inspector } = await import('pear-inspect')
  const inspector = await new Inspector()
  const key = await inspector.enable()
  console.log(`pear://runtime/devtools/${key.toString('hex')}`)
}
```

Then `pear run pear://runtime` → Developer Tooling → paste the key. Click "Open in Chrome" for full DevTools.

If the app exits but the process lingers, something kept the loop alive after `Pear.teardown` fired — typically an unclosed `peer-pipe`, undestroyed swarm, or open core. Full diagnostic flow is in the `pear-debug` skill.

## Common pitfalls

- **`process` is undefined.** Either `require('bare-process')` and use it explicitly, or `require('bare-process/global')` once at startup.
- **`Pear.app.dev` vs running from disk.** `--dev` only sets `Pear.app.dev`. Running from disk is `Pear.app.key === null`. They are independent.
- **Treating `Pear.app.args` as `argv`.** It contains only the app args (after `--`); it does not include the path or runtime args. The platform passes `--dev`, etc., before the dir, and Pear strips those.
- **Forgetting teardown.** Hyperswarm without `swarm.destroy()` leaves DHT records that slow future joins.
- **Using Node's `readline`.** Node's `readline` from `node:readline` or `readline` will not resolve in Bare. Use `bare-readline`.

## Testing

The scaffold ships [Brittle](https://github.com/holepunchto/brittle) as a dev dependency. `brittle` runs under Bare and is the convention.

```sh
npx brittle test/*.js
```

## Build / publish

When ready to ship:

```sh
npm prune --omit=dev
pear stage production
pear release production
pear seed production
```

See `pear-staging-release` for channels, release pointer mechanics, and rollback.
