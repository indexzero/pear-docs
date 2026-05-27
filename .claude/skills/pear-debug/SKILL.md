---
name: pear-debug
description: Diagnose and fix Pear application failures — `Pear.teardown` callback fires but the app keeps running, sidecar exits without running the app, `Error: While lock file ... Resource temporarily unavailable`, slow Hyperswarm joins, `AddonError: ADDON_NOT_FOUND`, `Bare is not defined`, `bare-pack` MODULE_NOT_FOUND, missing builtin modules — plus how to attach the V8 inspector with `pear-inspect` and read crash logs. Use when the user reports an unexpected hang, crash, or behavior in a Pear app, asks how to attach a debugger to a terminal app, or hits any of the documented error strings. Trigger terms include "pear hangs", "pear teardown not exiting", "RocksDB LOCK", "Resource temporarily unavailable", "ADDON_NOT_FOUND", "Bare is not defined", "pear-inspect", "pear sidecar log", "pear crash log", "swarm join slow".
metadata:
  tags: pear, debug, troubleshoot, pear-inspect, sidecar, crash-log, teardown
---

# Debugging Pear applications

Most Pear bugs fall into one of five buckets: lifecycle (teardown), storage (RocksDB LOCK), runtime mismatch (Bare vs Node), discovery (Hyperswarm), and packaging (`bare-pack`, native addons). This skill is a diagnostic decision tree per bucket.

## When to use

- Any reported failure in a Pear app or in the `pear` CLI.
- "How do I attach a debugger to a terminal Pear app?"
- An error message matches one in the table below.

For correct day-to-day usage, see `pear-terminal-app` / `pear-desktop-app` / `pear-mobile-app`. For storage architecture, `pear-data`. For network architecture, `pear-swarm`.

## Decision tree

```
Symptom → first check
─────────────────────────────────────────────────────────────
"my app exits but the process lingers"
  → outstanding handles. Add Pear.teardown handlers for every
    swarm, core, drive, pipe, server, dht, interval, timer.

"Pear.teardown callback fires but process keeps running"
  → typically an unclosed pear-pipe (call pipe.end()) or an
    open hyperswarm (call swarm.destroy()), or a setInterval
    that wasn't cleared.

"pear CLI exits without running the application"
  → run `pear --log <command>` for verbose CLI logs.
  → run `pear sidecar --log-level 3` to take over as sidecar
    and tail logs (closes other pear apps including Keet).
  → if sidecar prints "Closing any current Sidecar clients..."
    and hangs, an existing sidecar is stuck — `ps aux | grep
    pear`, then kill stragglers.
  → check crash logs in the platform `current/` directory:
    sidecar.crash.log, electron-main.crash.log, cli.crash.log.

"Error: While lock file ... LOCK: Resource temporarily unavailable"
  → two processes opened the same storage.
  → either two app instances are running (`ps aux | grep
    pear`), or the app constructed more than one Corestore.
  → fix: one Corestore per app; use store.namespace(...) for
    scoping. See pear-data.

"Joining Hyperswarm topic takes a long time"
  → swarm.destroy() was missed on a prior exit → stale DHT
    records. Wait it out or restart the sidecar.
  → firewall blocking UDP.
  → in NAT-restrictive environments holepunching may require
    a relay; this improves with more peers.

"AddonError: ADDON_NOT_FOUND: Cannot find addon"
  → addon not built for current Bare.platform / Bare.arch.
    Log Bare.platform and Bare.arch to confirm.
  → on mobile, the addon wasn't linked into the build. Check
    node_modules/react-native-bare-kit/(ios|android)/addons.
  → clear native build cache and rebuild.

"Bare is not defined" or "require.addon is not a function"
  → Bare-only code ran inside Electron (Node context).
  → see pear-node-compat for import maps.

"Bail: ModuleTraverseError: MODULE_NOT_FOUND ... node:crypto"
  → bare-pack saw a runtime-conditional require. Replace
    with imports maps. See pear-node-compat.

"updates don't reach peers"
  → see pear-staging-release: did you stage, seed, release?

"app data is gone after I uninstalled"
  → there is no uninstall. `pear drop app` destroys storage
    irreversibly. There is no built-in backup; back up
    ~/Library/Application Support/pear (or platform
    equivalent) before testing destructive commands.
```

## Attaching the V8 inspector (terminal apps)

`pear-inspect` exposes the V8 inspector over Hyperswarm.

```js
if (Pear.app.dev) {
  const { Inspector } = await import('pear-inspect')
  const inspector = await new Inspector()
  const key = await inspector.enable()
  console.log(`pear://runtime/devtools/${key.toString('hex')}`)
}
```

Then:

1. `pear run --dev .`
2. `pear run pear://runtime` in another terminal (opens the Pear runtime app).
3. Developer Tooling → paste the key.
4. Click "Open in Chrome" for full Chrome DevTools.

Inspector listens only when `Pear.app.dev` is true. The key can be handed to a remote developer for over-the-wire debugging.

## CLI / sidecar logs

```sh
pear --log <command>                            # verbose CLI log for that command
pear sidecar                                    # take over as sidecar
pear sidecar --log-level 3                      # 0=off, 1=err, 2=info, 3=trace
pear sidecar --log-labels internal,ipc          # filter labels
pear sidecar --log-stacks                       # attach stacks
```

Crash logs land in the platform's `current/` directory:

| OS       | Path                                                       |
| -------- | ---------------------------------------------------------- |
| macOS    | `~/Library/Application Support/pear/current/*.crash.log`   |
| Linux    | `~/.config/pear/current/*.crash.log`                       |
| Windows  | `%userprofile%\AppData\Roaming\pear\current\*.crash.log`   |

Files of interest: `sidecar.crash.log`, `electron-main.crash.log`, `cli.crash.log`.

## Killing stale processes

```sh
ps aux | grep -i pear        # macOS/Linux
tasklist | findstr pear      # Windows
```

If `pear sidecar` hangs, kill processes named `pear`, `pear-runtime`, `bare`, then re-run `pear`.

> **Warning:** Killing pear processes will close any running Pear apps (Keet, PearPass, etc.). Save first.

## Common error → fix

| Error | First action |
| --- | --- |
| `Resource temporarily unavailable` on a RocksDB LOCK | Find the second Corestore or second app instance; consolidate to one. |
| `ADDON_NOT_FOUND` | Log `Bare.platform`/`Bare.arch`; reinstall the addon; on mobile, re-run `bare-pack --linked`. |
| `Bare is not defined` | Code ran in Electron's Node context; provide a Node-side import or move to the Bare side. |
| `require.addon is not a function` | Same as above. |
| `MODULE_NOT_FOUND: 'node:crypto'` from `bare-pack` | Replace runtime-conditional `require()` with an `imports` map. |
| `Pear.teardown` fires but process lingers | Inventory: swarms, cores, drives, dhts, servers, pipes, intervals, timers. Add teardown handlers. |
| Slow `swarm.join` | Restart sidecar (clears stale DHT); check UDP firewall; wait. |
| `pear sidecar` exits silently | Check `sidecar.crash.log`; `pear sidecar --log-level 3`. |
| `pear info` returns `null` | Wrong key, or no peers seeding — try a known link first. |

## Recovering from a hung sidecar

```sh
# attempt graceful takeover with logs
pear sidecar --log-level 3

# if that prints "Closing any current Sidecar clients..." and stops:
ps aux | grep -E 'pear|bare'
kill <PIDs>
pear         # re-bootstrap
```

## Common pitfalls

- **`Bare.exit()` does not run `Pear.teardown` handlers.** Use `Pear.exit(code)`.
- **Empty teardown handlers.** A handler with nothing in it doesn't help — list every closable: `swarm.destroy()`, `core.close()`, `drive.close()`, `dht.destroy()`, `server.close()`, `bee.close()`, `pipe.end()`, `clearInterval(...)`.
- **Killing pear processes while apps are running.** Closes everything including Keet. Save first.
- **Deleting `~/Library/Application Support/pear/app-storage/`** to "reset" an app. Use `pear drop app` (still destructive but supported).
- **Inspector key in production builds.** Gate `pear-inspect` on `Pear.app.dev`.
