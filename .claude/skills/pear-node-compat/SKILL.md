---
name: pear-node-compat
description: Make Node.js code, libraries, and APIs work under the Bare runtime that powers Pear — by substituting `bare-*` modules, wiring `package.json` `imports` maps, applying the `with.imports` pragma, handling the missing `global.process`, and avoiding `bare-pack` failures from conditional `require()`. Use when the user gets `Bare is not defined`, `require.addon is not a function`, `MODULE_NOT_FOUND` for `node:*` specifiers, `bare-pack` errors, or is converting a Node CLI / library to Pear-compatible. Trigger terms include "bare vs node", "bare-fs", "bare-http1", "bare-subprocess", "imports map", "with.imports", "bare-pack module not found", "node:crypto not found", "global process undefined", "convert node to pear".
metadata:
  tags: pear, bare, node, compatibility, imports-map, bare-pack, bare-fs, bare-http1
---

# Node.js compatibility on Bare

Pear runs on Bare, not Node.js. Bare exposes a minimal `global.Bare` and nothing else by default — no `process`, no `fs`, no `http`. Standard-library functionality lives in `bare-*` modules. Most Node code can be made to run by wiring an `imports` map, occasionally with a `with.imports` pragma.

The authoritative substitution table is in [bare-node](https://github.com/holepunchto/bare-node).

## When to use

- The user sees `Bare is not defined`, `require.addon is not a function`, `MODULE_NOT_FOUND` for `node:*`, or `bare-pack` errors.
- The user wants to consume a Node-only npm package from a Pear app.
- The user is writing a hybrid module that should run on both Node and Bare.
- The user is wondering "why doesn't `process` exist?".

For storage/swarm questions, use `pear-data` / `pear-swarm`. For build/distribution issues, `pear-staging-release`.

## Substitution table (most common)

| Node                    | Bare                                                  |
| ----------------------- | ----------------------------------------------------- |
| `fs`, `fs/promises`     | `bare-fs`                                             |
| `path`                  | `bare-path`                                           |
| `os`                    | `bare-os`                                             |
| `process`               | `bare-process` (or `bare-process/global` for the global) |
| `events`                | `bare-events`                                         |
| `stream`                | `bare-stream`                                         |
| `buffer`                | `bare-buffer`                                         |
| `crypto`                | `bare-crypto`                                         |
| `url`                   | `bare-url`                                            |
| `tty`                   | `bare-tty`                                            |
| `readline`              | `bare-readline`                                       |
| `net`                   | `bare-tcp`                                            |
| `dgram`                 | `bare-dgram`                                          |
| `tls`                   | `bare-tls`                                            |
| `http`                  | **`bare-http1`** (different name)                     |
| `https`                 | `bare-https`                                          |
| `child_process`         | **`bare-subprocess`** (use `bare-daemon` for `detached`) |
| `worker_threads`        | `bare-worker`                                         |
| `zlib`                  | `bare-zlib`                                           |
| `dns`                   | `bare-dns`                                            |
| `assert`                | `bare-assert`                                         |
| `fetch` (global)        | `bare-fetch`                                          |
| `inspector`             | `bare-inspector`                                      |

For anything not in this list, check [bare-node](https://github.com/holepunchto/bare-node).

## Strategy 1 — write code in `bare-*` directly

For brand-new Pear/Bare apps, just import from `bare-*`. There is no benefit to pretending you're in Node.

```js
const fs = require('bare-fs')
fs.readFile('./config.json', (err, buf) => {})
```

## Strategy 2 — package.json `imports` map (hybrid modules)

Use when:

- You are publishing a library that should run on Node *and* Bare.
- You depend on someone else's Node module and want to override its `fs` import.

```jsonc
{
  "imports": {
    "node:fs": { "fs": { "bare": "bare-fs", "default": "fs" } },
    "fs":      { "bare": "bare-fs",  "default": "fs" },
    "fs/*":    { "bare": "bare-fs/*", "default": "fs/*" }
  },
  "dependencies": {
    "bare-fs": "^2.1.5"
  }
}
```

The `bare` condition resolves under Bare; `default` resolves elsewhere (Node). No code changes needed in the consuming module.

Important: **`imports` maps only apply to the package they're declared in.** They do not override a dependency's transitive imports.

## Strategy 3 — `with.imports` pragma (apply a map to a dep)

When a dependency you depend on uses `node:fs` internally and you want to rewrite *its* imports tree:

```js
// CommonJS
const dep = require('dep-that-uses-fs', { with: { imports: './package.json' } })

// ESM
import dep from 'dep-that-uses-fs' with { imports: './package.json' }
```

The `./package.json` referenced contains the `imports` map; Bare applies it to the entire subtree under `dep`. Use sparingly — it can mask incompatibilities and is overhead per import.

## Strategy 4 — `npm` alias for one-off rewrites

If only one or two deps reach for a Node builtin, install a Bare-shim under the Node name:

```sh
npm i bare-net net@npm:bare-node-net
```

Now `require('net')` resolves to `bare-node-net` (which proxies to `bare-tcp`) throughout the dep tree.

## The missing `global.process`

Bare does not set `global.process`. Two options:

1. **Prefer:** `const process = require('bare-process')` in code you control.
2. **Last resort:** at startup, `require('bare-process/global')` — sets `global.process` once. Use when a dep references `process` as a global and you can't fix it.

```js
require('bare-process/global')
console.log(process.platform)
```

## `bare-pack` constraints (mobile + advanced bundling)

`bare-pack` is a static analyser — it does not run your code. So this fails:

```js
// BAD
const { runtime } = require('which-runtime')
const crypto = runtime === 'bare' ? require('bare-crypto') : require('node:crypto')
```

Error: `Bail: ModuleTraverseError: MODULE_NOT_FOUND: Cannot find module 'node:crypto'`. `bare-pack` saw both branches and tried to resolve both.

Fix by declaring an `imports` map:

```jsonc
{
  "imports": {
    "node:crypto": { "bare": "bare-crypto", "default": "node:crypto" },
    "crypto":      { "bare": "bare-crypto", "default": "crypto" }
  }
}
```

Then write the code without branching:

```js
// GOOD
const crypto = require('crypto')   // resolves to bare-crypto on Bare, node:crypto elsewhere
```

If a dynamic dependency genuinely cannot be declared statically, mark it in `pear.stage.defer`:

```jsonc
"pear": { "stage": { "defer": ["optional-only-on-node-thing"] } }
```

## `require.addon is not a function` / `Bare is not defined` in Pear desktop

These appear when Bare-only code runs inside Electron's Node context. Pear v1 desktop apps booted Electron directly, so a Bare module loaded in the renderer / main process won't find Bare.

Solutions:

- Move the Bare-only code to a Bare process and talk to it via `pear-pipe` / `pear-run`.
- For Pear v2 (`pear-electron` + `pear-bridge`), the application boot is Bare and Electron is a UI subprocess — Bare modules work in the application code but still not in renderer JS.
- Provide both implementations via `imports` map (`bare` + `default`).

## Hybrid module checklist

When publishing a module intended for Node *and* Bare:

- [ ] Declare an `imports` map for every Node built-in you use.
- [ ] List the `bare-*` versions in `dependencies`.
- [ ] Test on both runtimes — `node test/index.js` and `bare test/index.js`.
- [ ] Do not use `if (typeof Bare !== 'undefined')` branches — use the map.
- [ ] If you need both `process` and a globalless API, prefer the API; only `require('bare-process/global')` if you must.

## Common pitfalls

- **`require('node:crypto')` works on Node but not Bare.** Add to the `imports` map.
- **`http` ≠ `bare-http`.** The Bare HTTP/1 implementation is `bare-http1`. There is no `bare-http`.
- **`child_process.spawn(..., { detached: true })`.** `bare-subprocess` does not implement detached spawning. Use `bare-daemon` instead.
- **`process.env`.** Available via `bare-env` (set/get) or `bare-process` (`process.env`). Different from Node when env was set after process start.
- **Top-level `await` in CJS.** Bare supports ESM with top-level await; CJS does not. Don't mix.
- **`Buffer.from(string, 'utf-8')`.** `bare-buffer` is Buffer-compatible but prefer `b4a` (the cross-runtime helper) when sharing code with Node and Bare.
- **`fetch` to arbitrary hosts.** Even with `bare-fetch`, Pear's HTTP guard blocks the request unless the host is in `pear.links`.
