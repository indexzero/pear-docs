---
name: pear-data
description: Use the Holepunch storage primitives correctly inside a Pear application — Hypercore (append-only log), Hyperbee (ordered K/V on a core), Hyperdrive (P2P filesystem), Autobase (multi-writer linearisation), and Corestore (one factory per app). Use when the user is persisting state, replicating between peers, hitting RocksDB LOCK contention from multiple Corestores, designing a bootstrap pattern for sharing keys, choosing between Hyperbee and Hyperdrive, batching writes, or wiring `core.replicate(conn)` into a swarm. Trigger terms include "hypercore", "hyperbee", "hyperdrive", "corestore", "autobase", "append-only", "p2p storage", "RocksDB LOCK", "discoveryKey vs key", "store.replicate", "core.update".
metadata:
  tags: pear, hypercore, hyperbee, hyperdrive, autobase, corestore, storage, replication
---

# Storage primitives for Pear apps

All Holepunch storage is **append-only** and built on Hypercore. Replication is **sparse by default** — peers fetch only the blocks they request. Discovery happens through Hyperswarm (see `pear-swarm`).

## When to use

- Any persistence inside a Pear app.
- "Store X in a way that syncs to peers."
- Picking between Hyperbee (K/V), Hyperdrive (files), Autobase (multi-writer log), or raw Hypercore.
- Debugging RocksDB LOCK errors, "core not ready", missing keys, divergent replication.

For the network side (discovery, connection establishment), switch to `pear-swarm`.

## Core principle: one Corestore per app

A Corestore is a factory + replication multiplexer. Inside a single application:

```js
import Corestore from 'corestore'
const store = new Corestore(Pear.app.storage)   // ← one and only one
```

Multiple Corestore instances against the same storage cause:

- `Error: While lock file ... LOCK: Resource temporarily unavailable` (RocksDB).
- Duplicate on-disk storage for the same key.
- Multiple replication streams per peer.

If you need scoped namespaces, use:

```js
const chatStore  = store.namespace('chat')
const inboxStore = store.namespace('inbox')
const chatCore   = chatStore.get({ name: 'log' })
const inboxCore  = inboxStore.get({ name: 'log' })
```

Namespacing avoids name collisions; the two `name: 'log'` cores get different keys.

## Hypercore

Append-only signed log. Foundation for everything else.

```js
import Hypercore from 'hypercore'

const writer = store.get({ name: 'log' })       // writable; deterministic from name+device
await writer.ready()
await writer.append('block 0')
await writer.append('block 1')
console.log(writer.length, writer.fork)
console.log(writer.key.toString('hex'))         // read capability
console.log(writer.discoveryKey.toString('hex'))// DHT lookup

// elsewhere
const reader = otherStore.get({ key: writer.key })
await reader.ready()
await reader.update()                           // fetch latest metadata before querying
const block = await reader.get(0)
for await (const b of reader.createReadStream({ start: reader.length, live: true })) { /* tail */ }
```

Critical:

- **Always `await core.ready()`** before reading `.key`, `.discoveryKey`, `.length`.
- **`key` vs `discoveryKey`.** `key` grants read access. `discoveryKey` (derived) finds peers without revealing the read capability.
- **`.update()` for readers.** Without it, a fresh reader may not know about appended blocks.
- **Sparse by default.** A reader who fetches `get(1000)` downloads only the blocks needed to verify and serve that block; not the whole log.

Replication:

```js
swarm.on('connection', conn => store.replicate(conn))    // one call covers ALL cores in the store
```

Never call `core.replicate(conn)` directly when you have a Corestore — let the store multiplex.

## Hyperbee — ordered K/V on a core

```js
import Hyperbee from 'hyperbee'
const core = store.get({ name: 'users' })
const bee  = new Hyperbee(core, { keyEncoding: 'utf-8', valueEncoding: 'json' })
await bee.ready()

await bee.put('user/42', { name: 'Mafintosh' })
const node = await bee.get('user/42')           // { seq, key, value } or null
await bee.del('user/42')

// range queries (lexicographic)
for await (const n of bee.createReadStream({ gte: 'user/', lt: 'user/~' })) {
  console.log(n.key, n.value)
}

// batches: atomic at flush
const batch = bee.batch()
for (const [k, v] of pairs) await batch.put(k, v)
await batch.flush()
```

Notes:

- Encodings must match on the writer side and any side that reads structured values.
- Multiple Hyperbees can share a single core with sub-prefixes via `bee.sub('users')`.
- Sparse: a `bee.get('user/42')` downloads only tree nodes on the path to that key.

## Hyperdrive — P2P filesystem

Internally a Hyperbee (metadata) + a Hypercore (content). Manage via Corestore.

```js
import Hyperdrive from 'hyperdrive'
import Localdrive from 'localdrive'

const drive = new Hyperdrive(store)             // writer
await drive.ready()
await drive.put('/README.md', Buffer.from('# hi'))

// mirror to/from a local directory
const local = new Localdrive('./out')
const mirror = drive.mirror(local)              // drive → local
await mirror.done()

const mirrorBack = local.mirror(drive)          // local → drive (writer-side bulk import)
await mirrorBack.done()
```

For a reader, pass the drive key:

```js
const reader = new Hyperdrive(store, writerDrive.key)
await reader.ready()
const buf = await reader.get('/README.md')      // sparse fetch
```

Pitfalls:

- Localdrive doesn't create parent directories until you write into them.
- Mirror operations are one-way per call; bidirectional sync = two mirrors with debouncing.
- Don't construct a separate Corestore for drive content. Use the app's Corestore.

## Autobase — multi-writer linearisation

```js
import Autobase from 'autobase'

const base = new Autobase(store, /* bootstrap */ null, {
  open: (linStore) => linStore.get('view'),
  apply: async (nodes, view, base) => {
    for (const { value } of nodes) await view.append(value)
  }
})
await base.append({ op: 'set', key: 'x', value: 1 })

for await (const node of base.view.createReadStream()) {
  // ordered, agreed-upon view across all writer cores
}
```

Use when multiple peers must write into a shared log with deterministic merge. For single-writer apps, plain Hypercore is simpler.

## Choosing between primitives

| Need | Use |
| --- | --- |
| Append-only event log | **Hypercore** |
| K/V store, range queries | **Hyperbee** |
| File tree, paths, blobs | **Hyperdrive** |
| Many writers, single agreed log | **Autobase** |
| Manage many cores efficiently | **Corestore** (always) |
| Mirror filesystem ↔ drive | **Localdrive** + `drive.mirror(local)` |

## Bootstrap pattern (sharing keys)

Corestore replication **does not exchange keys**. A reader must already know the key of every core they want to read. Common pattern:

1. The writer publishes a single "primary" core's key out-of-band (QR code, link, manual paste).
2. The first block of the primary core lists the keys of secondary cores:
   ```js
   await primaryCore.append(JSON.stringify({ inbox: inboxCore.key, files: filesDrive.key }))
   ```
3. The reader fetches block 0 of the primary core, then `store.get({ key })` for each secondary.

For invite flows, see [pear-pair](https://github.com/holepunchto/blind-pairing) and Hyperbeam (one-shot passphrase pipe).

## Cleanup

Always:

```js
Pear.teardown(async () => {
  await core.close()           // or drive.close(), bee.close()
  await store.close()
})
```

For Hyperswarm, see `pear-swarm`.

## Common pitfalls

- **Multiple Corestores.** RocksDB LOCK contention. Fix: one Corestore per app, scope with `namespace`.
- **Reading `.key` before `.ready()`.** Returns undefined. `await core.ready()`.
- **Forgetting `core.update()`** on a reader before `core.get()`. Reader sees old length.
- **Per-core `core.replicate(conn)`** when you have a Corestore. Use `store.replicate(conn)` — it multiplexes.
- **Reusing the same `name`** across subsystems. Use `store.namespace('subsystem')`.
- **Treating Hyperdrive as Localdrive.** Hyperdrive operates on the drive's internal Hyperbee; create parent dirs implicitly (no `mkdir` needed). Localdrive operates on disk and does not create parent dirs.
- **Encoding mismatch.** Writer with `valueEncoding: 'json'` and reader without will see raw buffers.
- **Skipping batch for bulk writes.** Hundreds of individual `bee.put` calls are slow; batch them.
- **Trying to delete from Hypercore.** You can't. Use a Hyperbee where you can `del` a key (the underlying log still grows).
