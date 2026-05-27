---
name: pear-swarm
description: Discover and connect peers using Hyperswarm (topic-based) and HyperDHT (keypair-based) inside a Pear application — including topic generation, `swarm.join`, `discovery.flushed()`, encrypted streams, `store.replicate(conn)`, holepunching, and proper teardown. Use when the user is joining a swarm topic, exchanging a public key, debugging slow joins, leaking DHT records, choosing between Hyperswarm and HyperDHT, or wiring connections into Corestore replication. Trigger terms include "hyperswarm", "hyperdht", "swarm.join", "topic", "discoveryKey", "holepunching", "swarm.destroy", "DHT bootstrap", "peer discovery", "connect peers".
metadata:
  tags: pear, hyperswarm, hyperdht, dht, p2p, networking, discovery
---

# Discovery and connections

Two layers move peers from "they exist somewhere" to "we have an encrypted stream":

- **HyperDHT** — Kademlia DHT with NAT holepunching. Peers find each other by public key; one server, many clients.
- **Hyperswarm** — built on HyperDHT. Peers find each other by *topic* (a 32-byte buffer). Symmetric; many peers per topic; reconnects automatic.

Most apps use Hyperswarm. Use HyperDHT directly only when you have a clear server/client split (e.g. a Hypershell daemon listening on its public key).

## When to use

- Joining a chat room, swarming a Hypercore, building any multi-peer feature.
- Connecting two peers by exchanged public key (invite flow, paired devices).
- "My swarm join is slow." / "Why doesn't this peer see me?"
- Replicating cores or drives across peers (combine with `pear-data`).

For storage/replication primitives themselves, use `pear-data`.

## One Hyperswarm per app

```js
import Hyperswarm from 'hyperswarm'
const swarm = new Hyperswarm()
Pear.teardown(() => swarm.destroy())
```

Multiple instances per app are an anti-pattern: they fragment peer connections, multiply DHT records, and slow reconnects. Join more topics on the one instance instead.

## Hyperswarm — topic-based discovery

```js
import Hyperswarm from 'hyperswarm'
import crypto from 'hypercore-crypto'

const swarm = new Hyperswarm()
const topic = crypto.randomBytes(32)              // or core.discoveryKey

swarm.on('connection', conn => {
  // conn is a SecretStream — encrypted, multiplexed
  store.replicate(conn)                           // hand off to Corestore
  conn.on('error', () => {})                      // always handle to avoid uncaught
})

const discovery = swarm.join(topic, { client: true, server: true })
await discovery.flushed()                         // optional: wait for DHT announcement

Pear.teardown(() => swarm.destroy())
```

Key facts:

- `swarm.join(topic, { client, server })` returns a discovery handle. `client: true` means "look for peers on this topic"; `server: true` means "announce that I'm here". Default: both true.
- `discovery.flushed()` resolves when the DHT announcement completes. Peer connections may arrive before or after.
- `swarm.connections` is the live set of `SecretStream`s. Iterate to broadcast.
- `swarm.leave(topic)` to stop on a topic without destroying the swarm.

### Topics from cores vs random topics

- Replicating a Hypercore? `swarm.join(core.discoveryKey)` — the core's discovery key is the natural topic.
- Chat room with an arbitrary name? `swarm.join(crypto.randomBytes(32))` once, share the topic hex out of band.
- Stable topic from a passphrase? `swarm.join(crypto.hash(Buffer.from(passphrase)))`.

### Multicast / broadcast

```js
for (const conn of swarm.connections) conn.write(message)
```

Or build a small protocol over `protomux` for typed messages, see [protomux](https://github.com/holepunchto/protomux).

## HyperDHT — keypair-based discovery

When you want a fixed identity (a server) and clients connect by knowing its public key:

```js
import DHT from 'hyperdht'
const dht = new DHT()
const keyPair = DHT.keyPair()                     // persistent identity

const server = dht.createServer(conn => {
  process.stdin.pipe(conn).pipe(process.stdout)
})
await server.listen(keyPair)
console.log('public key:', keyPair.publicKey.toString('hex'))

Pear.teardown(async () => {
  await server.close()
  await dht.destroy()
})
```

Client side:

```js
const dht = new DHT()
const conn = dht.connect(serverPublicKey)
conn.once('open', () => { /* connected */ })
```

When to use HyperDHT directly:

- A long-lived server identity (Hypershell, Hyperbeam endpoint).
- You need fine-grained control over holepunching.
- You're building a higher-level discovery system that doesn't fit topics.

Otherwise default to Hyperswarm.

## Wiring into Corestore replication

Almost every Hyperswarm `connection` handler ends with:

```js
swarm.on('connection', conn => store.replicate(conn))
```

This is the only call you need — Corestore multiplexes every loaded core onto the connection's SecretStream. You do not need (and should not write) a per-core `core.replicate(conn)`.

If you have multiple unrelated topics that should not share data, namespace the Corestore (`pear-data` skill) — but keep one swarm.

## DHT bootstrap

Pear caches known DHT nodes in `Pear.app.dht.nodes`. To override bootstrap (e.g. running against a private DHT or in tests):

```js
const swarm = new Hyperswarm({ bootstrap: [{ host: 'dht.example.net', port: 49737 }] })
```

Same option works for `new DHT({ bootstrap: ... })`.

## Slow joins — diagnostic checklist

If `discovery.flushed()` takes more than a few seconds:

1. **Prior process didn't destroy the swarm.** DHT still routes to the dead instance. Wait it out or restart Pear sidecar.
2. **Firewall blocks UDP.** HyperDHT requires UDP; check the host's firewall and your router.
3. **Restrictive NAT on both peers.** Holepunching may need a relay; this is rare and improves with more swarm peers.
4. **First-run with no cached bootstrap nodes.** Subsequent joins are faster because `Pear.app.dht.nodes` caches.
5. **You called `swarm.flush()` instead of `discovery.flushed()`.** They are different — `swarm.flush()` waits for the join to be accepted; `discovery.flushed()` waits for the DHT announcement.

## Teardown

```js
Pear.teardown(() => swarm.destroy())
```

Without it:

- The swarm stops handling new peers but the DHT still routes to it for a while.
- Other peers attempting to reconnect will see slow, failed handshakes.
- Repeated dev iterations leave a trail of stale DHT records.

For HyperDHT:

```js
Pear.teardown(async () => {
  await server.close()    // unannounce
  await dht.destroy()
})
```

## Common pitfalls

- **Multiple Hyperswarms in one app.** Fragmented peer set, slow joins. One swarm; many topics.
- **Calling `swarm.flush()` thinking it waits for connections.** It waits for the join to be confirmed; peers may arrive later.
- **Forgetting `conn.on('error', () => {})`.** Closed peers emit errors; uncaught crashes the app.
- **Joining a topic with `client: true, server: false`** then wondering why no peers connect to you. Server side must be true to be discoverable.
- **Passing a string topic.** Topics are 32-byte buffers; pass a `Buffer`/`Uint8Array`, not a hex string. Convert with `b4a.from(hex, 'hex')`.
- **Replicating per-core when you have a Corestore.** Single `store.replicate(conn)` covers all loaded cores.
- **Calling `DHT.keyPair()` on every start.** That changes the server identity each run — clients lose track. Persist the key pair (e.g. in Pear app storage or a checkpoint).
