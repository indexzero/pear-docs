# Agent-Friendly Project Practices

AI coding agents (Claude Code, Cursor, Copilot, Codex, Aider, etc.) work better when a project gives them a curated, predictable place to read context from. This page collects the conventions that have emerged across the ecosystem and shows how to apply them to a Pear application.

The goal is to make Pear projects legible to an agent on first contact — so the agent does not invent answers about Bare vs Node, stage vs run, or which P2P module does what.

## AGENTS.md

[`AGENTS.md`](https://agents.md) is a vendor-neutral Markdown file placed at the project root. It is the agent equivalent of a README: a predictable file that coding agents are trained or configured to look for.

Keep `README.md` for humans (what the app does, how to install) and put agent-specific operational detail in `AGENTS.md`. For a Pear application, recommended sections include:

- **Project overview** — one paragraph naming the app, that it is a Pear application, and whether it targets desktop, terminal, or mobile.
- **Runtime** — state explicitly that the app runs on [Bare](./bare-overview.md), not Node.js, and link to [Node.js Compatibility](./node-compat.md). This single fact prevents most wrong suggestions (e.g. reaching for `fs`, `child_process`, or npm packages that depend on Node-only APIs).
- **Entry point and configuration** — point at `package.json` `pear` field, see [Pear Application Configuration](./configuration.md).
- **Commands** — the actual `pear` CLI invocations the agent should use: `pear run --dev .`, `pear stage <channel>`, `pear release <channel>`, `pear seed <channel>`. See [Pear CLI](./cli.md).
- **Testing** — how tests are run (Brittle, `bare` test runner, etc.) and any required setup.
- **Code style / conventions** — formatter, lint command, import style.
- **Constraints to respect** — link to [Recommended Practices](./recommended-practices.md). At minimum call out: one [Corestore](../helpers/corestore.md) per app, one [Hyperswarm](../building-blocks/hyperswarm.md) per app, never load JS over HTTP(S), `npm prune --omit=dev` before staging.
- **Do-not-touch list** — generated files, `.git`, application storage paths.

For monorepos, nested `AGENTS.md` files override the root one for their subtree.

## `llms.txt` and `llms-full.txt`

The [llms.txt](https://llmstxt.org) standard is for documentation sites, not application repos. A site publishes:

- `/llms.txt` — a short, curated index: an H1, a blockquote summary, then H2 sections with `[name](url): description` links to the canonical Markdown for each major topic.
- `/llms-full.txt` (or `/llms-small.txt`) — a single concatenated Markdown bundle suitable for pasting into a context window.

Site authors who want agents to ground answers on their docs should publish both at the root of the docs host. The index lets an agent decide what to fetch; the bundle lets it ingest everything at once when context allows.

When working inside a Pear app, agents should be pointed at the Pear documentation's `llms.txt` (when available on `pears.com`) rather than scraping HTML. List the URL in `AGENTS.md` under a **References** section so the agent picks it up automatically.

## Skills and tool-specific config

Most coding agents also support a private, tool-specific configuration directory that is checked into the repo. Treat these as supplements to `AGENTS.md`, not replacements:

- **Claude Code** — `.claude/` (project-level settings, permissions, hooks) and `skills/` directories containing invocable skills.
- **Cursor** — `.cursor/rules/*.mdc`.
- **Copilot** — `.github/copilot-instructions.md`.
- **Aider** — `CONVENTIONS.md` referenced via `--read`.
- **Codex / OpenAI** — `AGENTS.md` is the primary surface.

For a Pear project, useful skill or rule files include: launching the app with `pear run --dev`, building a stage bundle, generating a new `pear init` template, and running a `bare` script directly without invoking Node.

Keep these files small and reference `AGENTS.md` for the authoritative content so the same guidance does not have to be maintained in five places.

## Pear-specific facts worth pinning

These are the corrections agents most often need when working on Pear code. Including them verbatim in `AGENTS.md` removes the guesswork:

- The runtime is Bare. Node built-ins are available only through the [Node.js compatibility layer](./node-compat.md); prefer Bare modules where they exist (`bare-fs`, `bare-path`, `bare-os`, ...).
- An application is identified by a `pear://` key, a `length`, and a `fork`. See [`Pear.app`](./api.md#pear-app).
- HTTP and HTTPS are blocked by default in Pear apps. Do not suggest `fetch` against arbitrary origins as a fix.
- Use `pear run --dev .` during development, `pear stage <channel>` to bundle, `pear release <channel>` to publish, `pear seed <channel>` to host.
- The `.git` directory is excluded by default, but only if `stage.ignore` is not overridden — if it is, re-add `.git` explicitly.
- Persistent storage lives under the Pear application data directory, not the project directory.

## Minimal `AGENTS.md` template

```markdown
# AGENTS.md

## Overview
This is a Pear desktop application. It runs on the Bare runtime, not Node.js.

## Commands
- Develop: `pear run --dev .`
- Stage:   `pear stage dev`
- Release: `pear release dev`
- Test:    `npm test`

## Runtime constraints
- One Corestore instance for the whole app.
- One Hyperswarm instance for the whole app.
- Never load JavaScript over HTTP(S).
- Run `npm prune --omit=dev` before staging.

## References
- Pear docs: https://docs.pears.com
- Recommended practices: https://docs.pears.com/reference/recommended-practices
- Bare overview: https://docs.pears.com/reference/bare-overview
- Node compatibility: https://docs.pears.com/reference/node-compat
```
