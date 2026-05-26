# AGENTS.md

Operational context for AI coding agents working in a Pear runtime project. This file is the entry point — read it before touching any code. The full doc set lives at https://docs.pears.com and a single-document condensed bundle at [`llms-full.txt`](./llms-full.txt).

This repository is the canonical [Pear](https://github.com/holepunchto/pear) documentation. Many of the rules below also apply verbatim to **any** Pear application repository; treat the bottom half of this file as a template `AGENTS.md` to copy into Pear apps.

---

## Project overview

- **Pear is a peer-to-peer runtime and deployment platform** built by Holepunch. Applications are Hyperdrives identified by `pear://<key>` links, replicated over the Hyperswarm DHT, and executed by the Pear runtime.
- **Pear is not Node.js.** The runtime is [Bare](https://github.com/holepunchto/bare) — a small V8 + libuv runtime. Node-only APIs (`fs`, `child_process`, `process`, `worker_threads`) are not present and must be replaced with `bare-*` modules or wired through `package.json` `imports` maps. See [`reference/node-compat.md`](./reference/node-compat.md).
- **Apps target three surfaces:** terminal (Bare + `bare-tty` + `bare-readline`), desktop (Bare + Electron via `pear-electron` + `pear-bridge`), and mobile (Bare via `react-native-bare-kit` + `bare-expo`).
- **There is no central server.** Distribution is `pear stage <channel>` → `pear seed <channel>`. There is no "package upload"; users pull from peers.

---

## Commands you will actually use

Authoring docs in this repo:

```sh
# This repo is plain Markdown rendered by GitBook. No build step.
# Validate links after editing SUMMARY.md or README.md links by visual inspection.
git diff --stat                  # confirm scope
```

For Pear apps (the audience of the docs):

```sh
# install runtime once per machine
npm i -g pear
pear                             # finishes setup; pulls platform from peers

# create
pear init --yes --type terminal           # terminal app
pear init ui --yes                        # desktop (UI) app

# develop
pear run --dev .                          # devtools, updates-diff, no trust prompts
pear run --dev . --some-arg               # args land in Pear.app.args

# stage / share / release
pear stage <channel>                      # writes drive → outputs pear:// link
pear seed  <channel>                      # keep running to serve peers
pear release <channel>                    # mark current length as production
pear info  pear://<key>                   # inspect length, fork, key
pear dump  pear://<key> ./out             # write the drive's files to disk

# inspect / clean
pear data apps                            # list installed apps + storage paths
pear gc releases|sidecars|assets|cores
pear drop app                             # DESTRUCTIVE reset of an app's storage
pear sidecar --log-level 3                # take over as sidecar and tail logs
```

---

## Runtime constraints — do not violate

These are the bugs novices file. Internalise them before suggesting code.

1. **Bare, not Node.** Don't suggest `require('fs')`, `process.cwd()`, `child_process`, `node:*` specifiers, or `global.fetch` against arbitrary origins. Reach for `bare-fs`, `bare-process`, `bare-subprocess` (use `bare-daemon` for `detached`), `bare-fetch`. Full mapping in [`reference/node-compat.md`](./reference/node-compat.md).
2. **One Corestore per application.** Multiple Corestores against the same storage cause RocksDB LOCK contention (`Error: While lock file ... Resource temporarily unavailable`). Subdivide with `store.namespace('chat')`, not by constructing a second Corestore.
3. **One Hyperswarm per application.** Joins multiplex across topics; instances do not. Always `Pear.teardown(() => swarm.destroy())` — leaked instances stall future DHT joins for tens of seconds.
4. **HTTP and HTTPS are blocked by default.** Allowlist trusted hosts and `pear://` links in `package.json` `pear.links`. Do not suggest a `fetch()` workaround.
5. **Never load JavaScript over HTTP(S).** This is a security boundary: Pear apps have native access. Code arrives only via the application drive.
6. **`Pear.exit(code)` runs `Pear.teardown` handlers. `Bare.exit(code)` does not.** Prefer `Pear.exit` for app shutdown.
7. **`pear.stage.ignore` replaces the default ignore list — re-add `.git`** if you set it. Otherwise `.git` ends up in the drive.
8. **Run `npm prune --omit=dev` before `pear stage`.** Dev dependencies pad the drive and slow first-run replication.
9. **`bare-pack` does not evaluate code.** Conditional `require()` based on a runtime check (`if (runtime === 'bare') require('bare-crypto') else require('node:crypto')`) will fail to bundle. Use `package.json` `imports` maps.
10. **`.key` and `.discoveryKey` are undefined before `.ready()`.** Always `await core.ready()` / `await drive.ready()` before reading them.
11. **Corestore replication does not carry keys.** Publish keys to peers either out-of-band or by writing them into a primary core that peers fetch first.

---

## File layout (this repo)

```
README.md                  Top-level showcase + module index. Rendered as docs home.
SUMMARY.md                 GitBook table of contents. Update this when adding/removing a page.
llms.txt                   Curated agent index (llms.txt standard).
llms-full.txt              Single-document condensed bundle for whole-doc LLM ingestion.
AGENTS.md                  THIS FILE — agent operational context.

reference/                 API + CLI + config reference (authoritative spec).
guide/                     Step-by-step tutorials (terminal, desktop, mobile, releasing).
howto/                     Task-oriented recipes against the building blocks.
building-blocks/           Hypercore, Hyperbee, Hyperdrive, Autobase, HyperDHT, Hyperswarm.
helpers/                   Corestore, Localdrive, Mirrordrive, Secretstream, etc.
tools/                     CLI tools (Hypershell, Hyperbeam, Drives, ...).
assets/                    SVG icons referenced by README tables.
.claude/skills/            Bundled skills for Claude Code agents (pear-*, see below).
evals/                     Eval suites for the skills and for the agent-facing docs.
```

When adding a new doc page, update `SUMMARY.md` **and** the relevant table in `README.md`. Do not duplicate content across pages; cross-link.

---

## Style conventions for doc changes

- Use Markdown. Tables are GitHub-flavored. Code blocks must have a language tag (` ```js`, ` ```sh`, ` ```jsonc`).
- Anchor headings get an explicit `<a name="...">` for cross-doc linking (e.g. `## pear stage<a name="pear-stage"></a>`). Don't rename anchors — many docs link to them.
- Stability badges are inline `<mark>` HTML:
  - `<mark style="background-color: #80ff80;">**stable**</mark>`
  - `<mark style="background-color: #8484ff;">**experimental**</mark>`
  - `<mark style="background-color: #ffffa2;">**deprecated**</mark>`
  - `<mark style="background-color: #ff4242;">**unstable**</mark>`
- Use real `pear://` keys in examples (e.g. `pear://keet`, `pear://runtime`) only when illustrating a published app. Never invent keys.
- The `pear` CLI block convention: a triple-backtick console block, no `$` prefix. Each flag column-aligned. Match the format already used in `reference/cli.md`.

---

## Skills bundled in this repo

The `.claude/skills/` directory ships Claude Code skills tuned for Pear development. They are organised as follows. When the conversation reaches into one of these domains, the corresponding skill is the source of truth — prefer it over re-deriving steps from the reference docs.

| Skill | Use when |
| ----- | -------- |
| `pear-runtime`         | Any work touching a Pear app. Router into the other skills. |
| `pear-terminal-app`    | Building or debugging Bare-based terminal apps. |
| `pear-desktop-app`     | Building or debugging Pear desktop apps (pear-electron, pear-bridge). |
| `pear-mobile-app`      | Building Bare-on-mobile apps (bare-expo, react-native-bare-kit). |
| `pear-staging-release` | `pear stage`, `pear release`, `pear seed`, `pear dump`, channel/version mechanics. |
| `pear-data`            | Hypercore, Hyperbee, Hyperdrive, Corestore, Autobase patterns. |
| `pear-swarm`           | Hyperswarm + HyperDHT discovery and connection patterns. |
| `pear-node-compat`     | Wiring `bare-*` modules, import maps, `with.imports` pragma, `bare-pack` constraints. |
| `pear-debug`           | `pear-inspect`, sidecar logs, crash logs, common errors. |
| `pear-init-template`   | Authoring `pear init` templates with `_template.json`. |

Each skill has its own `evals/evals.json` and a `tile.json` for registry metadata. The aggregate doc-effectiveness eval set lives in `evals/docs/`.

---

## Recommended reading order for common scenarios

| If the user is… | Read in this order |
| --- | --- |
| New to Pear | `guide/getting-started.md` → `reference/cli.md` → `reference/configuration.md` → `reference/api.md` |
| Building a terminal app | `pear-terminal-app` skill → `guide/starting-a-pear-terminal-project.md` → `guide/making-a-pear-terminal-app.md` → `guide/debugging-a-pear-terminal-app.md` |
| Building a desktop app | `pear-desktop-app` skill → `guide/starting-a-pear-desktop-project.md` → `guide/making-a-pear-desktop-app.md` → `reference/migration.md` (if porting from v1) |
| Building a mobile app | `pear-mobile-app` skill → `guide/making-a-bare-mobile-app.md` |
| Shipping a version | `pear-staging-release` skill → `guide/sharing-a-pear-app.md` → `guide/releasing-a-pear-app.md` |
| Stuck on Node-isms | `pear-node-compat` skill → `reference/node-compat.md` → `reference/troubleshooting.md` |
| Storage / data flow | `pear-data` skill → `howto/replicate-and-persist-with-hypercore.md` → `howto/work-with-many-hypercores-using-corestore.md` → topic-specific howto |
| Network connectivity | `pear-swarm` skill → `howto/connect-two-peers-by-key-with-hyperdht.md` → `howto/connect-to-many-peers-by-topic-with-hyperswarm.md` |
| Crashing / hanging | `pear-debug` skill → `reference/troubleshooting.md` → crash logs in the platform `current/` directory |

---

## Template for an application-level `AGENTS.md`

Copy the block below into a Pear application repository's root `AGENTS.md` and edit. This is the minimum that lets a coding agent ship correct code on first contact.

```markdown
# AGENTS.md

## Project
- Pear application (terminal | desktop | mobile). Runtime is Bare, not Node.js.
- Entry: <path>. Pear config in package.json under "pear".
- Storage lives at Pear.app.storage. One Corestore for the whole app.

## Commands
- Develop: pear run --dev .
- Stage:   pear stage <channel>
- Release: pear release <channel>
- Seed:    pear seed <channel>
- Test:    <test command>

## Runtime constraints (do not violate)
- Bare ≠ Node. Use bare-fs, bare-path, bare-os, bare-tcp, bare-tty, bare-readline, bare-subprocess, bare-process. Wire Node-named imports via package.json "imports" maps.
- One Corestore instance per app. Use store.namespace(...) to subdivide.
- One Hyperswarm instance per app. Always Pear.teardown(() => swarm.destroy()).
- HTTP/HTTPS blocked by default. Allowlist in pear.links.
- Never load JavaScript over HTTP(S).
- Prune dev deps: npm prune --omit=dev before pear stage.
- If pear.stage.ignore is set, re-add ".git" explicitly.
- Use Pear.exit(code) — it runs Pear.teardown; Bare.exit does not.

## Pear-specific facts
- App identity: pear://<key>, with current <fork>.<length>.<key>.
- Pear.app.key === null when running from disk.
- core.key / drive.key are undefined before .ready().
- Corestore replication does not carry keys — exchange out-of-band or via a primary core.
- bare-pack scans static imports only; never branch on runtime — use "imports" maps.

## References
- Docs:    https://docs.pears.com
- llms.txt: https://docs.pears.com/llms.txt
- API:     https://docs.pears.com/reference/api
- CLI:     https://docs.pears.com/reference/cli
- Bare:    https://docs.pears.com/reference/bare-overview
- Node-compat: https://docs.pears.com/reference/node-compat
- Recommended practices: https://docs.pears.com/reference/recommended-practices
```

---

## When in doubt

- Don't invent `pear://` keys.
- Don't downgrade to Node assumptions — if you don't know whether `X` exists in Bare, grep [`reference/node-compat.md`](./reference/node-compat.md) or `https://github.com/holepunchto/bare-node`.
- If a feature is `experimental`, `unstable`, or `deprecated` in the docs, surface that to the user before recommending it.
- If the user is migrating from v1, the answer almost always starts with [`reference/migration.md`](./reference/migration.md).
- If a fix involves `--no-verify`, `--force`, `--unsafe-clear-app-storage`, `pear drop`, or any destructive flag — stop and confirm with the user first.
