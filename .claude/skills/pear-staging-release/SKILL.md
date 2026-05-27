---
name: pear-staging-release
description: Operate the Pear distribution workflow — `pear stage`, `pear seed`, `pear release`, `pear dump`, `pear info`, `pear touch`, channels, the `<fork>.<length>.<key>` version triplet, the released vs staged pointer, rollbacks, and the dump-stage-release pattern. Use when the user is staging a build for the first time, picking a channel name (`dev` / `production` / `internal`), distributing a `pear://<key>` link, rolling back a release, switching the release pointer with `--checkout`, debugging "peer can't see my update", or producing distributable binaries via `pear-appling`. Trigger terms include "pear stage", "pear release", "pear seed", "pear dump", "channel", "release pointer", "checkout staged", "rollback pear", "distribute pear app", "seeding".
metadata:
  tags: pear, stage, release, seed, dump, distribution, channels, versions
---

# Pear staging, releasing, and seeding

A Pear app's distributable artefact is a Hyperdrive, identified by `pear://<key>` and versioned by `<fork>.<length>.<key>`. Three CLI verbs move state between disk, drive, and DHT:

```
       pear stage <channel>      pear seed <channel>
  disk ─────────────────────► drive ─────────────────────► DHT (peers)
                              │
                              │   pear release <channel>
                              ▼
                          release pointer
                          (length marked as production)
```

`pear dump` reverses `pear stage` (drive → disk). `pear run pear://<key>` resolves the drive locally and executes it.

## When to use

- The user is shipping a build for the first time, or for the Nth time.
- "I staged but peers don't see the change." / "I released but `pear run` still loads the old version."
- Picking channels (`dev`, `production`, `internal`) and understanding their key derivation.
- Setting up a CI/CD-like dump-stage-release pipeline.
- Producing binaries with `pear-appling`.

For runtime / app code, use `pear-terminal-app`, `pear-desktop-app`, or `pear-mobile-app`. For peer discovery, `pear-swarm`. For storage, `pear-data`.

## Mental model

- A **channel** is a name (`dev`, `production`, ...). The first `pear stage <channel>` from a project derives a Hyperdrive key from `(app name, channel, device corestore key)`. Subsequent stages append to the same drive.
- A **version** is a length within the drive's Hypercore — the number of appended blocks. Versions are immutable.
- A **release pointer** is a length the project author marks as "production". After `pear release`, `pear run pear://<key>` resolves to that length, **even if a higher length has been staged**.
- A **fork** happens when the append-only log is truncated (rare). Most apps stay at fork `0`.

## The five commands

### `pear stage <channel> [dir]`

Synchronises disk → drive. First stage creates the channel key; subsequent stages append.

```sh
pear stage dev                # first time: outputs pear://<key>
pear stage --dry-run dev      # preview changes, write nothing
pear stage --compact dev      # tree-shake via static analysis
pear stage --ignore tmp/ dev  # comma-separated path ignores
pear stage --purge dev        # remove previously-staged files that are now ignored
pear stage --json dev         # newline-delimited JSON output
```

Outputs:

```
🍐 Staging: myapp [ dev ]
   ...
   pear://abc123def456...
   ^_^ done
```

Save that link. Re-run `pear stage dev` and the same link appears — the key is deterministic per channel.

### `pear seed <channel|link> [dir]`

Announces the channel/link to the DHT and serves blocks to connecting peers. **Must keep running** to be reachable. Re-seeders (other peers who've connected to your drive) can take over once they've synced.

```sh
pear seed dev
pear seed pear://<key>          # reseed someone else's drive
pear seed --verbose dev
```

### `pear release <channel|link>`

Marks the current drive length as the release pointer. After this, `pear run pear://<key>` loads from that length.

```sh
pear release production                     # mark the latest length
pear release --checkout 500 production      # mark a specific length (rollback)
pear release --checkout current production  # mark current — same as no flag
```

To opt out and run the latest staged version regardless of release pointer:

```sh
pear run --checkout=staged pear://<key>
```

### `pear dump <link> <dir>`

Drive → disk. Reverse of stage. Use to inspect, audit, or build a "production from internal" pipeline.

```sh
pear dump pear://<key> ./out
pear dump pear://<key>/CHANGELOG.md ./snap     # subset by path
pear dump --checkout 500 pear://<key> ./old    # specific length
pear dump --list pear://<key>                  # list paths instead of writing
pear dump pear://<key> -                       # to stdout
```

### `pear info [link|channel]`

Read drive metadata. With no argument: platform information.

```sh
pear info pear://<key>            # length, fork, manifest, metadata
pear info --key pear://<key>      # just the key
pear info --metadata pear://<key> # raw metadata
pear info --manifest pear://<key> # post-pre-script manifest
pear info dev                     # channel info (local)
```

### `pear touch [channel]`

Generates a `pear://<key>` link without staging anything. Useful for automation that wants to reference a link before the first stage.

```sh
pear touch                  # random channel
pear touch production       # deterministic key for "production"
pear touch --json
```

## Choosing channels

| Channel name | Convention |
| --- | --- |
| `dev`         | Local-only dev iteration. May or may not be seeded. |
| `internal`    | Team-facing builds. Seeded for review. |
| `production`  | Public builds. Always seeded; release pointer maintained. |

There's nothing magic about these names — the key derives from the name, so changing the name produces a different drive. Pick early and don't churn.

## Rollback strategies

**Method 1 — repoint the release:**

```sh
pear release --checkout 500 production
```

Lightweight; no new drive content. Peers running `pear://<key>` load length 500. The downside: no diff is appended to the drive, so listeners on `Pear.updates` may not see the change as a "release update".

**Method 2 — dump-stage-release:**

```sh
pear dump --checkout 500 pear://<production-key> /tmp/snap
cd /path/to/production-project
# replace contents with /tmp/snap, ensuring git or backup
pear stage production
pear release production
pear seed production
```

Heavier; appends new content. Peers see a real update via `Pear.updates`. Use this when consistency with the update stream matters.

## The `--checkout` flag

`pear run --checkout=<v>` chooses what to load:

| Value           | Meaning |
| --------------- | ------- |
| (default)       | `released` if a release pointer exists, otherwise `staged` |
| `released`      | Explicitly the release pointer (default once released) |
| `staged`        | Latest staged length — overrides the release pointer |
| `<integer>`     | A specific length |

Test against staged after a fresh stage but before release:

```sh
pear stage production
pear run --checkout=staged pear://<key>
```

## Why "peers don't see my update"

Run this checklist:

1. Did you actually `pear stage <channel>` after editing? (Disk → drive is not automatic.)
2. Is `pear seed <channel>` running? If not, peers cannot pull new blocks. Re-seeders may have an older length cached.
3. Did you `pear release <channel>` afterwards? Without a release pointer, peers see the staged length; with one, they're pinned to the release unless they pass `--checkout=staged`.
4. Are peers running `pear run pear://<key>` recently? They may be on an older locally-cached length and need to come online for the swarm to push updates.
5. Are you seeding from the same machine that staged? Other re-seeders only have the lengths they've already pulled; if the latest length never reached them, they cannot serve it.

## CI / automation tips

- `pear touch <channel> --json` to pre-create a link in scripts.
- `pear stage --json <channel>` for parseable output.
- Stage from an ephemeral working tree (clone, `npm ci`, `npm prune --omit=dev`, `pear stage`) to keep dev cruft out.
- Run `pear seed` under a process manager (systemd, supervisord) on a publicly reachable host.

## Producing binaries

Use [`pear-appling`](https://github.com/holepunchto/pear-appling/) — a CMake-based shell that bootstraps Pear and your app drive. The binary is tiny and almost never needs rebuilding because it pulls the latest released drive at launch.

## Common pitfalls

- **Re-staging from a different machine produces a different key.** The key derives from the *device corestore key*. Stage from one canonical machine, or use [`pear-stage`](https://github.com/holepunchto/pear-stage) programmatically with a managed key.
- **`pear stage.ignore` replaces the default ignore list.** If you set it, re-add `.git`.
- **Forgetting `npm prune --omit=dev`** balloons the drive and slows first-run replication.
- **Closing the seeding terminal** stops your node from serving — but other re-seeders may keep the drive available.
- **Trust prompts on first run** require the user to type `TRUST` literally. `--dev` and `--no-ask` suppress it; don't recommend `--no-ask` for end users without explanation.
- **`Pear.updates` events** fire on length changes the runtime detects from peers; they do not necessarily mean "a release happened". Subscribe to `pear-updates` for richer events.
