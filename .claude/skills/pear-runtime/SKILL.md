---
name: pear-runtime
description: Routes Pear runtime questions to the right specialized skill and answers cross-cutting questions about the Pear platform — what Pear is, how Bare differs from Node.js, the CLI surface, and the application identity model. Use when the user mentions Pear, Bare, Holepunch, pear-electron, hypercore, hyperswarm, hyperdrive, autobase, pear:// links, "P2P app", "peer-to-peer app", or runs any `pear` CLI command, AND when the task does not obviously fit a more specific pear-* skill (which should be preferred). Trigger terms include "pear runtime", "build a pear app", "deploy a pear app", "how does pear work", "what is bare", "pear vs node", "holepunch", "pear sidecar".
metadata:
  tags: pear, bare, holepunch, p2p, runtime, router
---

# Pear runtime

The Pear runtime is a small, embeddable JavaScript runtime (Bare + V8 + libuv) plus a P2P distribution platform built on Hypercore. Apps are Hyperdrives identified by `pear://<key>`, replicated via Hyperswarm, run locally by the runtime, and never travel through a central server.

This skill is the entry point. For anything operational, use the specialised skill below — they have fuller context and shorter routes to the right answer.

## When to use

- Any task that touches a Pear application, the Pear runtime, the Bare runtime, or a Holepunch module (Hypercore, Hyperbee, Hyperdrive, Autobase, Hyperswarm, HyperDHT).
- The user asks "what is Pear / Bare / Holepunch?", "Pear vs Node", "Pear vs Electron", "how does P2P deployment work?".
- The task involves a `pear` CLI command and you're not sure which skill owns it.

Prefer the specialised skill below when the task is clearly within its scope. Falling through to this skill is fine — it routes.

## Routing table

| User task | Skill |
| --------- | ----- |
| Building a terminal app (bare-readline, bare-tty, interactive CLI) | `pear-terminal-app` |
| Building a desktop app (pear-electron, pear-bridge, HTML UI) | `pear-desktop-app` |
| Building a mobile app (bare-expo, react-native-bare-kit, Android/iOS) | `pear-mobile-app` |
| `pear stage`, `pear release`, `pear seed`, `pear dump`, channels, versions, rollback | `pear-staging-release` |
| Hypercore, Hyperbee, Hyperdrive, Corestore, Autobase usage patterns | `pear-data` |
| Hyperswarm, HyperDHT, topic discovery, peer connections | `pear-swarm` |
| `bare-*` module substitutions, `package.json` `imports`, `with.imports` pragma, `bare-pack` errors | `pear-node-compat` |
| `pear-inspect`, sidecar crash logs, RocksDB LOCK errors, hanging teardown | `pear-debug` |
| `pear init` templates with `_template.json`, `__locals__`, distributing templates | `pear-init-template` |

If the question spans multiple skills, start with this one's "Core principles" section, then descend.

## Core principles

- **Pear is not Node.js.** The runtime is [Bare](https://github.com/holepunchto/bare). There is no `global.process`, no Node `fs`, no `child_process` — instead `bare-fs`, `bare-process`, `bare-subprocess`, wired in via `package.json` `imports` maps. The full mapping is `reference/node-compat.md`.
- **An app is a Hyperdrive.** Its identity is `pear://<key>` and its version is `<fork>.<length>.<key>`. `Pear.app.key === null` when running from disk; otherwise it points at the drive.
- **There is no central server.** `pear stage` writes the drive, `pear seed` announces it to the DHT, peers `pear run pear://<key>`. Take seeds down and the network of peers continues to serve each other.
- **One Corestore. One Hyperswarm.** Per application. Always. Subdivide cores with `store.namespace(...)`.
- **HTTP/HTTPS is blocked.** Allowlist in `pear.links` if you really need to fetch from the network.
- **Cleanup is mandatory.** `Pear.teardown(() => swarm.destroy())`. Use `Pear.exit(code)` rather than `Bare.exit(code)` so teardown handlers run.

## Quick start: smallest possible Pear app

```sh
mkdir hello && cd hello
pear init --yes --type terminal
pear run --dev .
```

The generated `index.js` prints a banner and exits. Edit it; the dev runtime tears down and restarts on `pear run --dev .` again. To make it a live, distributable app:

```sh
pear stage dev          # outputs a pear:// link
pear seed dev           # keep this terminal open
pear run pear://<key>   # in another machine
```

## CLI surface at a glance

```
pear init     create a project (or from a pear:// template link)
pear run      run an app from a link or directory
pear stage    write disk state to the channel's Hyperdrive
pear release  mark a length as production
pear seed     announce and replicate a channel to the DHT
pear info     read drive metadata for a link
pear dump     write a link's files to a directory
pear touch    pre-create a pear:// link without staging
pear shift    migrate app storage between two app links
pear drop     destroy an app's storage (irreversible)
pear gc       remove unused releases, sidecars, assets, cores
pear data     inspect the platform's local databases
pear sidecar  take over as the local sidecar (use for log inspection)
pear versions print platform + module versions
```

Full reference: `reference/cli.md`.

## Common cross-skill questions

- *"Can I use TypeScript?"* — Compile to JavaScript and point the entrypoint at the output (`main`). Pear/Bare load JavaScript at runtime; there is no built-in TS compilation.
- *"How do I share an app?"* — `pear stage <channel>` → `pear seed <channel>`. Distribute the `pear://<key>` link. First-run peers get a trust prompt unless `--no-ask`.
- *"Can users see my IP?"* — Yes; peers exchange addresses to connect. Use a VPN if that matters.
- *"How do I run on Windows / macOS / Linux?"* — `npm i -g pear` works on all three. Mobile uses a separate path; see `pear-mobile-app`.
- *"Where does an app store its data?"* — `Pear.app.storage`. Platform-wide root: macOS `~/Library/Application Support/pear`, Linux `~/.config/pear`, Windows `%userprofile%\AppData\Roaming\pear`.

## What this skill is not

- This skill does not contain step-by-step build instructions for a specific app type — those are in `pear-terminal-app`, `pear-desktop-app`, `pear-mobile-app`.
- It does not contain Hypercore/Hyperswarm APIs — those are in `pear-data` and `pear-swarm`.
- It does not contain the `bare-*` substitution table — that is in `pear-node-compat`.
