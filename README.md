<p align="center">
  <img src="deep-events.svg" width="450" alt="deep-events"/>
</p>

<h1 align="center">deep-events</h1>
<p align="center">
  <em>🌳 Unified path-based event runtime for JavaScript</em>
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/deep-events">
    <img src="https://img.shields.io/npm/v/deep-events?color=blue" alt="npm">
  </a>
  <img src="https://img.shields.io/badge/dependencies-0-brightgreen" alt="zero dependencies">
  <img src="https://img.shields.io/badge/ES5-compatible-green" alt="ES5">
  <img src="https://img.shields.io/badge/status-in%20development-yellow" alt="status">
</p>

<p align="center">
  Events, state, timers, DOM, workers, and lifecycle — one primitive: <b>the path</b>.<br>
  Browser, Node.js, Web Workers, Service Workers, worker_threads, child processes — same API everywhere.
</p>

> **⚠️ Project status: _Active development_.**
> APIs may change without notice until v1.0. Use at your own risk and please report issues!

```js
// One API for everything, addressed by path:
ev.on('/chat/room-42', ['message'], handler);          // events
ev.set('/chat/room-42/users', { count: 5 });           // state
ev.interval('/chat/room-42/refresh', poll, 10000);     // timers
ev.ask('/services/db/messages', 'load', q, cb);        // request/response — maybe another thread
ev.dom('/chat/room-42/input', '#msg', 'keydown', h);   // DOM (browser)

// Room closes. One line. Listeners, state, timers, DOM
// bindings, in-flight requests — all gone. Zero leaks.
ev.clear('/chat/room-42/**');
```

## Table of Contents

**Understand it**
1. [The Problem](#-the-problem)
2. [The Core Idea: Everything Is a Path](#-the-core-idea-everything-is-a-path)
3. [The Mental Model](#-the-mental-model)
4. [If You Know Redis (or a Filesystem, or MQTT…)](#-if-you-know-redis-or-a-filesystem-or-mqtt)
5. [What It Replaces & How It Compares](#-what-it-replaces--how-it-compares)
6. [Design Philosophy](#-design-philosophy)

**Use it**
6. [Features at a Glance](#-features)
7. [Installation](#-installation)
8. [Quick Start — Node.js](#-quick-start--nodejs)
9. [Quick Start — Browser](#-quick-start--browser)

**Go deep**
10. [Architecture](#-architecture)
11. [API Reference — Core](#-core-api)
12. [Scoping: ev.create() and ev.path()](#-scoping-evcreate-and-evpath)
13. [Workers & Mounts](#-workers--mounts)
14. [DOM Binding](#-dom-binding-browser)
15. [Browser Runtime: Tabs & Service Worker](#-browser-runtime-tabs--service-worker)
16. [Cookbook — Common Scenarios](#-cookbook--common-scenarios)

**Operate it**
17. [Debugging & Troubleshooting](#-debugging--troubleshooting)
18. [Performance & Batching](#-performance--batching)
19. [Environment Detection](#-environment-detection)
20. [Testing](#-testing)
21. [Roadmap](#-roadmap)
22. [License](#-license)



## 💥 The Problem

Every non-trivial JavaScript app ends up building the same plumbing, five times, with five different libraries:

- **Events between modules** — an EventEmitter, or mitt, or a hand-rolled pub/sub. Flat string names, no ownership, listeners leak when you forget to unsubscribe.
- **State** — Redux, Zustand, signals. A separate world with its own subscription model, disconnected from your events.
- **DOM wiring** — `addEventListener` plus a hand-maintained list of things to remove on teardown. One forgotten `removeEventListener` per month, forever.
- **Workers & processes** — raw `postMessage` with hand-rolled routing, correlation IDs for request/response, restart logic, and "did the worker boot yet?" flags.
- **Cleanup** — `componentWillUnmount`-style rituals: arrays of unsubscribe functions, timers you hopefully remembered, in-flight requests you hopefully cancelled.

Each solved separately, none aware of the others. The integration tax is where the bugs live: the interval that keeps polling after the page navigated away; the worker reply that arrives after the component died; the state subscription that outlives its owner.

**The observation behind deep-events:** all five problems share one missing concept — *ownership*. And this is not a new disease. Programming has seen it before, under another name:

| Manual memory in C | Manual events in JS |
|---|---|
| Memory leak — forgot `free` | Listener leak — forgot `off` |
| Use-after-free | Zombie handler — runs after its page died |
| Double-free | Removing a listener that's already gone |
| Dangling pointer — memory with no owner | Dangling closure — holding a dead DOM node |

Same failure patterns, different objects, same root: **the concept of ownership doesn't exist in the code.** A pointer doesn't say who frees it; a listener doesn't say which page it belongs to or when it should die. That information lives in the developer's head — the one place it gets forgotten. C suffered this for decades until Rust moved ownership *into the type system*. Events are still waiting for their Rust moment — cleanup functions and lint rules (the `useEffect`-return ritual) are the static-analyzers-for-C stage: discipline aids, not structural cures. **Event leaks are the malloc of this era.**

deep-events' answer is to move ownership **into the structure**: attach everything to a hierarchical path, and ownership, addressing, routing, and cleanup all fall out of the same tree. `ev.on('/pages/dashboard/filter', …)` doesn't just register a listener — it *declares* that this listener lives under `/pages/dashboard` and dies exactly when that scope dies. The system can act on what the code states.

*(These ideas are developed in depth in the essay [The Code That Doesn't Lie](./async_philosophy.md) — on async history, the geometry of time in code, and why structure beats discipline.)*



## 🌳 The Core Idea: Everything Is a Path

A path is a slash-separated hierarchical name, like a filesystem path or a URL:

```
/chat/room-42/users
/services/db
/tabs/38213
/page/dashboard/widgets/sales
```

In deep-events the path is not a label — it plays **three roles at once**:

### 1. The path is the IDENTITY (addressing)

Like a Redis key or a URL, a path names a thing you can talk to or about. Listeners register *at* paths. State lives *at* paths. Asks are sent *to* paths. Timers are owned *by* paths. There is no separate registry of "topics" or "stores" or "channels" — the path namespace is the only namespace.

```js
ev.set('/services/db/config', { pool: 10 });      // state AT a path
ev.on('/services/db/status', ['changed'], h);      // listener AT a path
ev.ask('/services/db/users', 'get', { id: 1 }, cb); // request TO a path
```

### 2. The path is the SCOPE (structure)

Paths form a tree, and subtrees are natural module boundaries. Everything under `/services/db` *belongs to* the db service. This is what makes `ev.mount` meaningful — you graft real code (a file, a worker thread, a whole child process) onto a subtree, and the subtree becomes that code's territory:

```js
ev.mount('/services/db', { file: './db.js', type: 'thread' });
// From now on, /services/db/** IS that worker.
// Asks to /services/db/anything route there. Its listeners live there.
```

Wildcards make scopes queryable: `*` matches one segment, `**` matches any depth — in listener paths, in `ev.keys` patterns, and in cross-boundary subscriptions:

```js
ev.on('/chat/*/messages', ['new'], h);     // every room's messages
ev.on('/transfer/**', ['progress'], h);    // the whole transfer subtree
```

### 3. The path is the LIFECYCLE (ownership & cleanup)

Because everything attaches to the tree, destroying a subtree destroys everything in it — listeners, state, timers, DOM bindings, pending asks (local **and** cross-boundary), with `dispose` hooks running first:

```js
ev.clear('/page/dashboard/**');
// Think `rm -rf` for runtime resources. No unsubscribe arrays,
// no forgotten intervals, no ghost listeners. This one line is
// the reason the library exists.
```

The discipline this buys you: **when you create a resource, you decide its path — and by deciding its path, you've already decided when it dies.** A timer under `/page/edit` dies with the edit page. An ask owned by `/chat/room-42` cancels when the room closes. Lifecycle stops being a chore you perform and becomes a property of where things live.



## 🧠 The Mental Model

### A message is (path, type, data)

```js
ev.emit('/chat/room-42', 'message', { text: 'hi' });
//        └── WHERE        └── WHAT    └── DETAILS
```

- **path** — the address: which node in the tree this concerns.
- **type** — the verb: what happened there (`'message'`, `'changed'`, `'login'`).
- **data** — the payload.

Listeners subscribe to a path plus a set of types (or all types). This two-level addressing keeps the tree clean: one path per *thing*, many event types per thing — instead of the flat-emitter pattern of encoding everything into names (`'chat:room42:message'`).

### Two verbs: emit and ask

| | `ev.emit` | `ev.ask` |
|---|---|---|
| pattern | fan-out, fire-and-forget | request/response |
| listeners | 0..N fire, in priority order | first `e.reply()` wins, propagation stops |
| result | returns count of local listeners | callback gets `(err, res)` — reply, or a typed error |
| failure mode | silence is legal | `NO_HANDLER` / `TIMEOUT` / `WORKER_DEAD` / `CLEARED` |

If you'd reach for an event bus — emit. If you'd reach for an RPC call or a fetch — ask. Same tree, same addressing, same routing.

### State is events

`ev.set`/`ev.merge` don't just store — every change **emits** a `'changed'` event on the same path, carrying the new value, the previous one, and the list of changed keys. So "subscribe to state" and "listen to events" are the same operation, and `{ snapshot: true }` closes the classic gap ("I subscribed after the value was set"):

```js
ev.on('/app/user', ['changed'], render, { snapshot: true });
// fires immediately with current value, then on every change
```

### Location transparency

The caller of `ev.ask('/services/db/users', 'get', …)` does not know — and must not care — whether the handler runs in the same file, a `require`'d module, a worker thread, a child process, or (in the browser) a Service Worker. Routing is the runtime's job. This is the property that lets you **move a module between isolation levels by editing one line of `mounts.json`**, with zero code changes in the module or its callers.

The single honest exception: *inside* a worker, a path starting with `/` crosses the boundary, and a relative path stays local. That one rule is the entire cross-boundary semantics you must remember (details: [Troubleshooting](#-debugging--troubleshooting)).



## 🗺️ If You Know Redis (or a Filesystem, or MQTT…)

deep-events deliberately borrows mental models you already have. If any of these systems is familiar, you already half-know this library.

### It's like Redis — keys, values, TTL, pub/sub

The data layer is a conscious homage to Redis, with the path as the key:

| Redis | deep-events | notes |
|---|---|---|
| `GET k` / `SET k v` | `ev.get(p)` / `ev.set(p, v)` | values are real JS objects, not strings |
| `SET k v NX` / `XX` | `ev.set(p, v, { nx: true })` / `{ xx: true }` | same conditional semantics |
| `SET k v EX 60` | `ev.set(p, v, { ex: 60 })` | lazy expiration, same as Redis |
| `INCR` / `INCRBY` / `DECR` | `ev.incr(p)` / `ev.incrby(p, n)` / `ev.decr(p)` | atomic, init-to-0, TTL-aware |
| `EXPIRE` / `TTL` / `PERSIST` | `ev.expire(p, s)` / `ev.ttl(p)` / `ev.persist(p)` | same return conventions (-1, -2) |
| `MGET` / `MSET` | `ev.mget([...])` / `ev.mset({...})` | |
| `KEYS pattern` | `ev.keys('/cache/user/*')` | hierarchical patterns, not glob-on-flat |
| `HSET` (partial update) | `ev.merge(p, { field: v })` | shallow merge, per-field change tracking |
| `SUBSCRIBE` / `PUBLISH` | `ev.on` / `ev.emit` | but see the differences below |

Where it's *deliberately not* Redis: it's in-process (no network, no serialization for local calls — nanoseconds, not milliseconds); keys are hierarchical, not flat; and every write emits a `'changed'` event with `prev` + `changed` fields — Redis pub/sub and keyspace live in separate worlds, here they're the same tree.

### It's like a filesystem — with mount as literally `mount`

The name `ev.mount` is not an accident. Like a Unix filesystem: a hierarchical namespace; mounting grafts a foreign "device" (a worker, a process, a sandboxed VM) onto a directory; everything under the mount point belongs to it; unmounting removes the subtree; and `ev.clear('/x/**')` is `rm -rf`. Even `ev.status()` is your `mount`/`df`.

### It's like MQTT / NATS topics — wildcards and interest-based delivery

Subscriptions with wildcards (`/chat/*/messages`, `/transfer/**`) and forward-only-to-the-interested delivery is the topic model of MQTT (`+`/`#`) and NATS subjects — applied *inside* one app across its execution contexts, instead of between machines. If you've designed topic hierarchies, you know how to design path hierarchies.

### It's like a process supervisor — PM2/OTP-flavored mounts

Mount options read like a supervisor config because that's what the mount layer is: `restart`, `maxRestarts`, `restartDelay`, `ping` health checks, `idle` shutdown, `bootTimeout`, graceful-shutdown handshake, and blue-green `ev.reload` with drain and optional verify-before-promote. Crash isolation per subtree, Erlang-supervision-lite, without leaving your app.



## ⚖️ What It Replaces & How It Compares

### The unification table

| Problem | Typical stack | deep-events |
|---|---|---|
| Events between modules | EventEmitter / mitt | `ev.on` / `ev.emit` on paths |
| Request/response | custom callbacks, promises, RPC wrappers | `ev.ask` / `e.reply` |
| State management | Redux / Zustand / signals | `ev.get` / `set` / `merge` + `'changed'` events |
| Caching with TTL | lru-cache / hand-rolled | `ev.set(p, v, { ex })` |
| DOM binding + observers | addEventListener + Resize/Intersection/MutationObserver boilerplate | `ev.dom` — one API, path-owned |
| Worker communication | postMessage + hand-rolled protocol | `ev.mount` — transparent routing |
| Worker lifecycle | hand-rolled restart/health logic | mount options: restart, ping, idle, bootTimeout |
| Zero-downtime worker updates | usually nobody dares | `ev.reload` (blue-green, drain, keep/promote) |
| Cross-tab messaging | raw BroadcastChannel | `{ to: 'all-tabs' }` + SW-routed `/tabs/<id>` paths |
| Cleanup on navigation/teardown | unsubscribe arrays, umount rituals | `ev.clear('/scope/**')` |

### Honest comparisons

- **vs EventEmitter / mitt** — those are 50 lines and perfect for flat, single-context events. deep-events earns its size when you need hierarchy, ownership/cleanup, request-response, or more than one execution context. If you need none of those, use mitt.
- **vs Redux / Zustand / signals** — they bring mature devtools, middleware ecosystems, and framework bindings; deep-events brings state that's unified with events, workers, and lifecycle. If your app is a single-context React tree and state is your only problem, the dedicated tools are deeper. If state is one of five problems, unification wins.
- **vs Comlink** — Comlink turns one worker into elegant RPC proxies and is *lighter* for that exact job. deep-events adds what Comlink deliberately doesn't: many contexts, routing between them, restart policies, subscriptions, blue-green reload, and the same API on both sides of the wire.
- **vs PM2 / systemd** — deep-events replaces the *inner* layer people often stretch PM2 into: running many small services as separate PM2 apps that talk over HTTP/Redis becomes mounts on one bus (restart policies, health pings, blue-green — per service, finer-grained than PM2's). It does **not** replace PM2's outer role: an external supervisor that restarts your *top-level* process when it dies and starts it on boot — deep-events lives inside that process and dies with it. The right stack is both: one thin external supervisor (PM2/systemd/Docker restart policy) keeping main alive, deep-events orchestrating everything inside. PM2's cluster-mode port load-balancing across cores is also not covered — deep-events scales by service separation, not by cloning one HTTP listener.
- **vs NATS / Redis pub-sub / MQTT brokers** — same topic-and-wildcard mental model, opposite deployment: brokers connect machines over a network; deep-events connects contexts inside one app with zero network. They compose — a gateway module can bridge selected paths to a broker.
- **vs socket.io** — client↔server networking; deep-events is client-internal and server-internal plumbing. Also composes naturally.

### When NOT to use deep-events

Honesty section. Skip this library if:

- Your app is small, single-context, and an EventEmitter genuinely suffices — the concepts here would be ceremony.
- You need multi-machine messaging — that's a broker's job, not this.
- You depend on the Redux/devtools ecosystem (time-travel, RTK Query…) more than on unification.
- Your team needs TypeScript-first DX today — typings are on the roadmap but not shipped.
- You need battle-tested maturity now — this is an in-development library; the test suite is growing but the API may still move before v1.0.



## 📐 Design Philosophy

Five principles, from the essay behind the library ([full text](./async_philosophy.md)):

### 1. Events are the foundation, not an add-on

Almost every mainstream language was designed synchronous, with events bolted on later — listener *patterns* in Java, `asyncio` as a language-within-a-language in Python, coroutines reaching C++ after forty years. The result: generations trained to treat async as a complication. But most of what software does — network, disk, user — is asynchronous by nature. deep-events takes the Erlang position: the event is the first-class citizen, and everything else (state, timers, RPC, process management) is built *on* the event model rather than beside it. That's why `ask` is an event with a reply, state changes *are* events, and there are never two worlds to bridge.

### 2. Callback-first, because geometry carries meaning

This is a deliberate choice, not a compatibility artifact. Code encodes time on two axes: **vertical = sequence** (line after line — "and then"), **horizontal = nesting** ("when this arrives"). In callback style, dependency is *visible*: what's nested depends, what sits at the same indent is independent — the flowchart from the whiteboard survives as the shape of the code. `async/await` reads beautifully precisely because it flattens that information away: three `await` lines look identical whether they depend on each other or could have run in parallel (the most common performance bug in modern JS). deep-events keeps the honest geometry — and a thin Promise facade is on the roadmap for those who want it at the edges.

### 3. Cancellation is a property of scope, not an object

The `AbortController` saga is a case study in patch-upon-patch: Promises hid the listener → an external cancel object was needed → it was one-dimensional → combinators (`AbortSignal.timeout/any`, six years later) — all to reconstruct what the event model had in one line: *stop listening*. deep-events' position: a resource's lifetime is its owner's lifetime, and the owner is a path. `ev.clear('/page/**')` is cancellation as a geometric operation — listeners removed, timers killed, in-flight asks rejected with `CLEARED` — with nothing to remember, because **information that lives in structure cannot be forgotten; it requires no maintenance.**

### 4. Structural abstraction is a source of speed, not a tax

There are abstractions that add layers (they cost), and abstractions that change structure (they *win*): the event loop beat thread-per-connection not despite the abstraction but because of it — global knowledge enables scheduling no manual code can match. Same here: path-to-node caching makes exact dispatch a single lookup (millions/sec); clearing a hundred listeners is one tree walk; batching amortizes postMessage because the boundary layer sees all the traffic. You pay only for the flexibility you actually use — wildcards cost more than exact paths, and only when present.

### 5. Unification is the requirement, not the ambition

The natural first impression: *"events, state, TTL, request/response, workers, blue-green — too much in one library."* That reading assumes this is a feature collection. It isn't. It's a single guarantee — **a resource is bound to its owner at creation and dies with it; after `clear`, nothing under that scope still runs** — and the contents are exactly what the guarantee requires: *every kind of resource that lives over time.*

Listeners, timers, stored values, in-flight requests, workers — each is inside not because more is better, but because any kind left outside is a hole in the guarantee. Take the Redis-style state layer: had it been left out, a stored value would once again be something the developer must track *outside the code* — precisely the disease from the ownership table above, reintroduced through the back door. `clear` would run, something would survive, the guarantee would be false. Like an ownership system for memory, a structural guarantee is binary: cover everything in the scope, or promise nothing about it.

So the admission criterion is a test, not an appetite: **does it have a lifetime? Then it must live in the tree.** Nothing was crammed in; nothing required was left out. (And blue-green reload isn't "one more feature" — it's what falls out when the worker itself is a resource in the tree and you ask how to replace a resource without violating its guarantees.)

The honest consequence: you don't adopt a library, you adopt an approach to async-resource ownership — within a scope, all-in or not at all. Gradual adoption works along *scope* boundaries: one page, one worker, one service subtree at a time, each fully inside.



## ✨ Features

- **Zero Dependencies**
  No external packages. Optional `cbor-x` only for child-process mounts.

- **Same API Everywhere**
  Browser tab, Service Worker, Web Worker, Node.js main, worker_threads, child processes. Write once, run anywhere. Moving a module between mount types requires **zero code changes**.

- **Zero Leaks by Design**
  `ev.clear('/page/**')` kills listeners, timers, state, DOM bindings, and pending requests (local **and** cross-boundary).

- **Built-in State (Redis-style)**
  Per-path data with `get`/`set`/`merge`/`del`, atomic counters, TTL/expiration, conditional set (`nx`/`xx`), bulk ops, pattern queries, change events with `prev` + `changed`.

- **Ask / Reply**
  Request-response over the bus. First reply wins. Timeouts, error codes, owner-based cancellation, transparent cross-boundary routing.

- **Cross-Boundary Subscriptions**
  A worker calling `ev.on('/some/global/path')` automatically subscribes through its parent. Matching emits from anywhere in the tree are forwarded to it — including wildcard paths.

- **Direct Peer Channels**
  When a thread mounts another thread, they get a private `MessageChannel` and talk directly — skipping the main process entirely (2 fewer hops per round-trip).

- **Adaptive Message Batching**
  Every postMessage boundary is wrapped in a rate-detecting batcher with a **per-transport threshold** derived from measured send costs (Node threads ~8000/sec, CBOR processes ~6000, tab↔SW ~1000 — see `test/bench-transports.js` and `test/bench-browser.html` to calibrate). Below threshold: zero added latency. Above: messages coalesce automatically, flushed via `setImmediate` (Node) or a MessageChannel self-task (browser) — never a throttleable timer, so batching keeps working at full speed even in background tabs.

- **Blue-Green Reload**
  Zero-downtime hot reload for workers. New version boots, old version drains in-flight requests, optional state transfer, optional pre-promote verification.

- **DOM Binding**
  One API for events AND observers (resize, scroll, visibility, element lifecycle). Delegation, key combos, debounce/throttle — all path-owned.

- **Callback-First — by conviction**
  Nesting makes the dependency graph visible; sequence reads down, futures read right ([why](#-design-philosophy)). ES5 compatible everywhere. A thin Promise facade is planned for the edges.

- **Built-in Flow Tracing**
  `ev.trace(true)` logs every cross-boundary hop and keeps a ring buffer retrievable from any context — debugging "why didn't my event arrive" takes minutes, not hours.



## 📦 Installation

### Node.js

```bash
npm install deep-events
```

```js
var ev = require('deep-events');
```

> **Singleton** — every `require('deep-events')` in every file returns the same instance for that context (main process, each worker, each child process has its own).
> The bus IS the glue between modules. No need to pass references around.

### Browser

```html
<script src="deep-events.min.js"></script>
<script>
    ev.on('/app', ['ready'], function(e) {
        console.log('App is ready!');
    });
</script>
```

> 📂 Browser bundles ship in three flavors: **tab**, **service worker**, and **web worker** — each includes only what's needed for that environment.



## 🟢 Quick Start — Node.js

A simple service that handles requests on the bus:

```js
var ev = require('deep-events');

// Register a service
ev.on('/services/greet', ['hello'], function(e) {
    e.reply({ message: 'Hello, ' + e.data.name + '!' });
});

// Call the service from anywhere
ev.ask('/services/greet', 'hello', { name: 'Dan' }, function(err, res) {
    console.log(res.data.message);  // "Hello, Dan!"
});
```

Mount a worker for isolation:

```js
var ev = require('deep-events');

// Mount a worker thread — full isolation, blue-green reloadable
ev.mount('/services/db', { file: './db-worker.js' });

// Requests route transparently
ev.ask('/services/db/users', 'get', { id: 42 }, function(err, res) {
    console.log(res.data.user);
});

// Hot reload with zero downtime
ev.reload('/services/db', function(err) {
    console.log('DB worker reloaded — zero dropped requests');
});
```

And inside `db-worker.js`:

```js
import de from 'deep-events';
const ev = de.create();          // auto-scoped to /services/db

ev.on('users', ['get'], function(e) {     // registers /services/db/users
    e.reply({ user: findUser(e.data.id) });
});

ev.ready();                       // signal boot complete
```



## 🌐 Quick Start — Browser

```html
<script src="deep-events.min.js"></script>
<script>
    // DOM events — path-owned, auto-cleanup
    ev.dom('/ui/search', '#search-input', 'input', { debounce: 300 }, function(e) {
        console.log('Search:', e.data.value);
    });

    // State management
    ev.set('/app/user', { name: 'Dan', online: true });

    ev.on('/app/user', ['changed'], function(e) {
        console.log(e.data.changed);       // ['online']
        console.log(e.data.online);        // false
        console.log(e.data.prev.online);   // true
    });

    ev.merge('/app/user', { online: false });

    // Page navigation — clear everything when leaving
    ev.clear('/page/dashboard/**');
</script>
```



## 🏗️ Architecture

### Bird's eye view

Every execution context — main, each worker, each tab, the SW — runs **the same core**: a trie holding listeners, state, and timers, plus dispatch. Contexts are connected through **hubs**:

```
NODE                                    BROWSER
────                                    ───────
            main (hub)                        Service Worker (hub, /main)
           /    |     \                        /        |        \
     require  thread  process             tab A       tab B      tab C
      (same    │  \      (CBOR             │            │      (BroadcastChannel
       trie)   │  thread  /stdio)        Web Worker   Web Worker  for tab↔tab emits)
               │    │
               └────┘ direct MessageChannel (thread↔sub-thread peers)
```

- **In-process mounts** (require/import/vm) share the hub's trie — zero overhead, dispatch is a function call.
- **Cross-boundary mounts** (thread/process/worker/sw/port) have their own trie and talk to the hub over a wire protocol.
- **Peer channels**: when a thread mounts a thread, the hub hands both ends of a MessageChannel — they talk directly, skipping the hub.
- The Node hub is the stable anchor; the browser hub (SW) is *ephemeral by design* — killed after ~30s idle — so tabs are the source of truth and rebuild it (see [Browser Runtime](#-browser-runtime-tabs--service-worker)).

### How a message travels

**`ev.emit('/transfer/x/received', 'block', data)` on main:**

```
1. local dispatch  — trie walk (exact + * + ** nodes), listeners fire by priority
2. mount routing   — does a mount own /transfer/x/…? → T_EMIT to that worker
3. subscriptions   — which OTHER workers subscribed to a matching pattern?
                     → T_EMIT to each (this is what lets /explore, a thread,
                       receive events about /transfer, which lives elsewhere)
```

**`ev.ask('/services/db/users', 'get', q, cb)`:**

```
1. mount routing         — /services/db is mounted → T_ASK to it, correlation id,
                           timeout armed; T_REPLY resolves the callback
2. else local dispatch   — a local listener may e.reply()
3. else askSubscribers   — parallel fan-out to workers whose SUBSCRIPTIONS match;
                           first reply wins
4. else                  — cb(NO_HANDLER)
```

**Inside a worker**, one rule decides everything: absolute path (`/…`) → crosses the boundary (to a peer if one owns the path, else to the hub); relative path → local trie only.

### The wire protocol (overview)

Numbered message types over postMessage / CBOR-framed stdio (full details: `lib/wire.js`):

| range | messages | purpose |
|---|---|---|
| data | T_EMIT(1) T_ASK(2) T_REPLY(3) | the actual traffic, correlation via `a` id |
| lifecycle | T_READY(4) T_SHUTDOWN(11) T_SHUTDOWN_COMPLETE(12) | boot + graceful-shutdown handshake |
| health | T_PING(7) T_PONG(8) | liveness; PONG carries SW bootTime for restart detection |
| state | T_STATE_DUMP(5) T_STATE_LOAD(6) | blue-green state transfer |
| topology | T_MOUNT(9) T_UNMOUNT(10) T_PEER_PORT(17) T_PEER_DISCONNECT(18) | nested mounts, direct peer channels |
| interest | T_SUBSCRIBE(13) T_UNSUBSCRIBE(14) | cross-boundary subscriptions (refcounted) |
| browser | T_TAB_STATE(15) T_TAB_REQUEST(16) | tab registry maintenance |
| transport | T_BATCH(19) | envelope of coalesced messages |

Lifecycle/health messages are never batched; everything else may arrive inside a T_BATCH.

### Drivers: how mount types plug in

Every mount type is a **driver** — an object registered with `ev.mountType(name, driver)` describing boot/send/terminate/shutdown for one kind of execution context. The six Node types and five browser types are all implemented through this same contract, so a custom transport (SharedWorker, Electron ipc, a WebSocket bridge…) is a userland driver, not a fork. Drivers also declare `available()` (graceful no-op on unsupported platforms) and `batchThreshold` (measured per-transport batching entry point).

### The layer stack

| Layer | What | Key APIs |
|:---:|---|---|
| **1** | **Core** — trie dispatch, ask/reply, event pool, priorities | `on`, `emit`, `ask`, `off`, `clear` |
| **2** | **Data** — Redis-style per-path store, TTL, counters, change events | `set`, `get`, `merge`, `del`, `incr`, `expire`, `keys` |
| **3** | **Timers** — path-owned, drift-corrected, atomic replace | `timeout`, `interval`, `remainingTime` |
| **4** | **Scoping** — mount-aware auto-prefixing, `'..'` resolution | `create`, `path`, `self` |
| **5** | **Wire** — protocol, subscription tracking/tables, batching | (internal: `wire.js`, `batcher.js`) |
| **6** | **Mounts** — 11 mount types, lifecycle, blue-green, peer channels | `mount`, `reload`, `unmount`, `status`, `mountType` |
| **7** | **Browser transport** — tab registry, SW routing, subscriptions, liveness | `/tabs/*`, `/main/*`, `/system/sw` |
| **8** | **DOM** — delegation + observers + lifecycle, path-owned | `dom` |

Each layer builds only on the ones below; a build can stop at any layer (the worker bundle ships layers 1–5).



## 🔧 Core API

Quick index — every public method and where it's documented:

| area | methods |
|---|---|
| Events | `on` `once` `off` `emit` `clear` — [below](#events) |
| Request/response | `ask` + `e.reply` — [Ask / Reply](#ask--reply) |
| Data (Redis-style) | `get` `set` `merge` `del` `incr` `incrby` `decr` `decrby` `expire` `ttl` `persist` `mget` `mset` `keys` — [Data](#data-redis-style-api) |
| Timers | `timeout` `interval` `clearTimeout` `clearInterval` `remainingTime` — [Timers](#timers) |
| Cleanup | `clear` `dispose`/`onClear` — [Cleanup](#cleanup--disposal) |
| Scoping | `create` `path` `self` `ready` `shutdownComplete` — [Scoping](#-scoping-evcreate-and-evpath) |
| Mounts | `mount` `unmount` `reload` `status` `mountType` — [Workers & Mounts](#-workers--mounts) |
| DOM | `dom` — [DOM Binding](#-dom-binding-browser) |
| Debugging | `trace` `dev` `onError` — [Debugging](#-debugging--troubleshooting) |
| Meta | `env` `tabId` — [Environment Detection](#-environment-detection) |


### Events

```js
// Listen — returns a listener ID
var id = ev.on('/chat/messages', ['new', 'edit'], function(e) {
    e.data;        // payload
    e.path;        // '/chat/messages'
    e.type;        // 'new' or 'edit'
    e.source;      // 'self' | mount path (e.g. '/services/db') | '/main' | 'tab:123'
    e.timestamp;   // Date.now() at emit
    e.stop();      // stop propagation to remaining listeners
});

// Omit types → catch-all listener (fires for every type on the path)
ev.on('/chat/messages', function(e) { });

// Listen once
ev.once('/auth', ['login'], function(e) { });

// Emit — returns the number of listeners that fired
ev.emit('/chat/messages', 'new', { text: 'hello', user: 'dan' });

// Remove
ev.off(id);                 // specific listener
ev.clear('/chat/messages'); // one exact path
ev.clear('/chat/**');       // everything under a path (recursive)
```

**Event data snapshot:** plain-object payloads are shallow-copied at emit time, so async handlers that hold on to `e.data` see stable scalar fields. `Uint8Array` / `ArrayBuffer` / arrays pass by reference (no copying of binary blobs).

### Ask / Reply

Request-response over the bus. First reply wins, auto-stops propagation.

```js
ev.ask('/math/add', 'calc', { a: 2, b: 3 }, function(err, res) {
    if (err) return console.error(err.code);  // TIMEOUT | NO_HANDLER | ...
    console.log(res.data.sum);  // 5
});

ev.on('/math/add', ['calc'], function(e) {
    e.reply({ sum: e.data.a + e.data.b });
    // Error reply: e.reply(null, { code: 'MY_ERROR', message: '...' })
});
```

Ask options:

```js
ev.ask(path, type, data, {
    timeout: 5000,        // default 30000; 0 = no timeout
    owner: '/page/edit',  // ev.clear('/page/edit') cancels this ask (code CLEARED)
    local: true,          // never route to mounts — local dispatch only
    transfer: [buf]       // transferables for zero-copy cross-boundary
}, cb);
```

Error codes: `NO_HANDLER`, `TIMEOUT`, `CLEARED`, `WORKER_DEAD`, `MOUNT_FAILED`, `BOOT_TIMEOUT`, `NO_MOUNT`.

### Data (Redis-style API)

Per-path data store. The **path is the key**.

```js
// Set (replace entire value) — objects, arrays, or scalars
ev.set('/app/user', { name: 'Dan', lang: 'he', online: true });

// Read (returns a clone, not a reference)
var user = ev.get('/app/user');

// Merge (shallow — only updates specified keys)
ev.merge('/app/user', { online: false });

// Delete data (NOT listeners/timers — use ev.clear for that)
ev.del('/app/user');           // entire value
ev.del('/app/user', 'online'); // single field

// Listen for changes — every set/merge/del emits 'changed'
ev.on('/app/user', ['changed'], function(e) {
    e.data.online;        // false (current)
    e.data.prev.online;   // true (previous)
    e.data.changed;       // ['online']
});

// Snapshot — fire immediately with current value, then on changes
ev.on('/app/user', ['changed'], handler, { snapshot: true });

// Scalar values arrive as e.data._value
ev.set('/score', 42);   // changed event: { _value: 42, prev: {...}, changed: ['_value'] }

// Atomic counters (init to 0 if missing)
ev.incr('/stats/visits');
ev.incrby('/stats/visits', 10);
ev.decr('/stats/visits');
ev.decrby('/stats/visits', 3);

// TTL / Expiration
ev.set('/cache/session', { token: 'xyz' }, { ex: 3600 });  // expires in 1hr
ev.expire('/cache/session', 1800);   // change TTL
ev.ttl('/cache/session');            // seconds remaining (-1 = no TTL, -2 = missing)
ev.persist('/cache/session');        // remove TTL

// Conditional set
ev.set('/lock/job', 'worker-1', { nx: true });   // only if NOT exists
ev.set('/app/config', data, { xx: true });        // only if exists

// Bulk
ev.mget(['/a', '/b'], function(err, vals) { });
ev.mset({ '/a': 1, '/b': 2 });

// Pattern query — returns paths that currently hold data
ev.keys('/cache/user/*');    // ['/cache/user/1', '/cache/user/2']
ev.keys('/cache/**');
```

### Wildcards

```js
ev.on('/chat/*/messages', ['new'], handler);   // * = exactly one segment
ev.on('/chat/**', ['message'], handler);        // ** = zero or more segments
```

Wildcards work in listener paths, `ev.keys` patterns, and cross-boundary subscriptions.

### Timers

Path-owned. Die with `ev.clear`. Long delays are chunked and drift-corrected (reliable across system sleep).

```js
// The path IS the timer identity — returns the path
ev.timeout('/jobs/cleanup', callback, 5000);
ev.interval('/jobs/poll', callback, 10000);

// Re-scheduling on the same path REPLACES the previous timer (atomic)
ev.timeout('/jobs/cleanup', otherCb, 9000);   // old one is gone

// Path-less form — a synthetic path is generated and returned
var t = ev.timeout(callback, 5000);
ev.clearTimeout(t);

ev.clearTimeout('/jobs/cleanup');
ev.clearInterval('/jobs/poll');
ev.remainingTime('/jobs/cleanup');   // ms until fire
```

### Listener Options

```js
ev.on(path, types, handler, {
    consume: true,            // remove after first trigger (ev.once shorthand)
    timeout: 10000,           // auto-remove after duration
    onTimeout: fn,            // called when the timeout removes the listener
    filter: fn,               // fn(e) → boolean; trigger only if true
    match: { field: value },  // fire only if e.data matches
                              //   value can be a RegExp
                              //   or a Uint8Array (binary prefix match — great
                              //   for wire-protocol type bytes)
    debounce: 300,            // per-listener trailing debounce
    throttle: 100,            // per-listener leading throttle
    priority: 1,              // number (1=first, 10=last) or 'high'|'normal'|'low'
    replace: true,            // replace existing listener on same path+types
    unique: true,             // skip registration if one already exists
    snapshot: true,           // fire immediately with current state (if any)
});
```

### Cleanup & Disposal

```js
// Clear a path or subtree: listeners, timers, state, pending asks, DOM bindings
ev.clear('/page/dashboard');      // exact node
ev.clear('/page/dashboard/**');   // node + entire subtree

// Register cleanup callbacks that run BEFORE the path is cleared
ev.dispose('/page/dashboard', function() {
    // resources still alive here — close connections, flush buffers
});
ev.onClear === ev.dispose;   // alias
```

### Error Handling

```js
// Ask errors go to the callback
ev.ask('/services/db', 'query', data, function(err, res) {
    if (err) err.code;  // TIMEOUT | WORKER_DEAD | NO_HANDLER | CLEARED | ...
});

// Safety net for unhandled errors (listener exceptions, late replies)
ev.onError(function(err, path, type) {
    console.error('Error at', path, ':', err.message);
});

// Dev mode — verbose warnings for API misuse, frozen state clones
ev.dev(true);
```



## 🎯 Scoping: ev.create() and ev.path()

### ev.create() — the mount-module API

`de.create()` returns a scoped `ev` whose relative paths are auto-prefixed with the **current mount context**. The same file works unchanged as `require`, `import`, `vm`, `thread`, or `process`:

```js
import de from 'deep-events';
const ev = de.create();     // scope = mount path (or '/main' outside a mount)

ev.on('routes', ['match'], handler);       // → <mountPath>/routes
ev.on('/global/thing', ['x'], handler);    // absolute — unchanged
ev.ask('../http/cache', 'get', data, cb);  // '..' = sibling under parent

ev.set('config', { a: 1 });                // → <mountPath>/config
ev.clear();                                 // clears <mountPath>/**

ev.ready();                // signal "boot complete" to the mount system
ev.shutdownComplete();     // signal "cleanup done" during graceful shutdown
```

Relative paths support `..` segments (`'../cache'`, `'../../other'`). All data/timer/mount methods are available on the scoped object, plus `ev.self` (the scope path).

### ev.path() — lightweight prefix scoping

A Redis-style convenience wrapper. Note the different `.on` signature (no path argument — the prefix IS the path):

```js
var room = ev.path('/chat/room-42');
room.on(['message'], handler);              // listens on /chat/room-42
room.emit('typing', { user: 'dan' });      // emits on /chat/room-42
room.set('users', { count: 5 });           // sets /chat/room-42/users
room.incr('msg-count');
room.ask('/api/data', 'fetch', {}, cb);    // asks are auto-owned by the scope
room.timeout(fn, 5000);                    // timer owned by the scope

ev.clear('/chat/room-42/**');              // everything dies
```

Inside workers, `ev.path('..')` resolves against `ev.self` (the worker's mount path), enabling clean sibling/parent access.



## ⚙️ Workers & Mounts

Mount code to paths. Multiple isolation levels. Transparent routing.

### Node.js mount types

```js
// Worker thread (default) — full isolation, blue-green reload
ev.mount('/services/db', { file: './db.worker.js' });

// Child process — separate PID, CBOR framing over stdio (requires cbor-x)
ev.mount('/services/heavy', { file: './heavy.js', type: 'process' });

// MessagePort — external worker, you manage the lifecycle
ev.mount('/services/ext', { port: existingPort });

// In-process require — zero overhead, sync CJS
ev.mount('/services/utils', { file: './utils.js', type: 'require' });

// In-process import — async ESM
ev.mount('/services/esm', { file: './esm.js', type: 'import' });

// Sandboxed VM — restricted globals for semi-trusted code
ev.mount('/plugins/user', {
    file: './plugin.js', type: 'vm',
    sandbox: { config: {...}, helper: fn }   // extra globals (ev, console, timers built in)
});
```

### Browser mount types

```js
// Service Worker — the browser's "main process". Convention: mount at /main
ev.mount('/main', { file: '/sw.js', type: 'sw' });
ev.ask('/main/counter', 'get', {}, cb);     // routed to the SW transparently

// Web Worker
ev.mount('/services/crunch', { file: '/worker.js', type: 'worker' });

// require / import — script loading into the current context
ev.mount('/services/brotli', { file: '/lib/brotli.js', type: 'require' });
```

### Mount Options

```js
ev.mount('/services/db', {
    file: './db.worker.js',
    type: 'thread',           // thread | process | port | require | import | vm | sw | worker
    start: 'onDemand',        // boot on first ask/emit (default: 'immediate')
    idle: 60000,              // graceful shutdown after 60s of inactivity
    restart: true,            // auto-restart on crash (default: false)
    maxRestarts: 3,           // give up after N consecutive crashes → status 'failed'
    restartDelay: 1000,       // wait between crash and restart
    ping: 6000,               // health-check interval; unresponsive worker = crash
    maxMissedPings: 10,
    bootTimeout: 10000,       // max wait for ev.ready() (default 30000)
    shutdownTimeout: 5000,    // max wait for shutdownComplete before force-kill
});
```

Environment-specific drivers may declare `available()` — mounting a type that isn't supported in the current environment is a silent no-op instead of a 30-second boot timeout.

### Worker Lifecycle

```js
// Inside the worker file:
const ev = de.create();
ev.on('users', ['get'], handler);
ev.ready();                                   // → mount status: 'running'

// Graceful shutdown handshake:
ev.on('/system/mount', ['shutdown'], function() {
    closeConnections();
    ev.shutdownComplete();                    // → main proceeds to terminate
});
// (No shutdown handler registered? Completion is automatic.)

// On main — observe lifecycle:
ev.on('/system/mount', ['changed'], function(e) {
    e.data.path;    // '/services/db'
    e.data.status;  // starting | running | reloading | draining | crashed | stopped | failed
    e.data.prev;
    e.data.reason;  // 'exit' | 'bootTimeout' | 'unresponsive' | 'manual' | ...
});

ev.status();   // { mounts: { '/services/db': { type, status, uptime, pid/threadId, ... } } }
```

### Cross-Boundary Subscriptions

Inside a worker, `ev.on` with an **absolute path outside your own subtree** automatically subscribes through the parent. Matching emits from anywhere are forwarded to you — wildcards included:

```js
// Inside /explore (a worker thread):
ev.on('../transfer/block_message/receiving/*/*', ['received'], handler);
// Any matching emit on main (or forwarded to main) reaches this worker.
```

This also works for **your own sub-mounts**: a worker that mounts a child thread/process can listen on the child's paths and receive its emits (delivered directly over the peer channel for thread↔thread, or via main otherwise). One rule: call `ev.mount()` **before** registering listeners on the child's paths.

```js
// Inside /parent (a thread):
ev.mount('child', { type: 'thread', file: './child.js' });   // mount first
ev.on('/parent/child/events/*', ['fire'], handler);          // then listen
```

Subscriptions are refcounted, cleaned automatically on `ev.off` / `ev.once` consumption / `ev.clear`, and re-established fresh after reload.

The same mechanism runs **in the browser**: a tab's `ev.on('/main/...')` (or any non-local absolute path) subscribes through the Service Worker, which then forwards only matching emits to that tab — instead of broadcasting everything to every tab. The SW's table survives its frequent restarts because tabs automatically re-register on every new-SW signal; tabs without subscriptions (older bundles) keep receiving everything, so the upgrade is fully backward compatible.

### Nested Mounts & Direct Peer Channels

Workers can mount their own sub-workers. `thread → thread` pairs automatically get a **private MessageChannel** and exchange asks/emits/subscriptions directly — main is skipped entirely. Crash/unmount/reload tears the channel down and falls back to main routing.

### Blue-Green Reload

Zero-downtime hot reload. New version boots → traffic switches → old version drains in-flight requests → graceful shutdown.

```js
ev.reload('/services/db', function(err) {
    console.log('v2 is live');
});

// Swap the file on the fly
ev.reload('/services/db', { file: './db-v2.js' }, cb);

// State transfer — v1's per-path data is dumped and loaded into v2 (silently)
ev.reload('/services/db', { state: true }, cb);
ev.reload('/services/db', { state: filterFn }, cb);   // filterFn(stateMap) → stateMap

// With verification — test v2 before promoting
ev.reload('/services/db', { keep: true }, function(err, next) {
    next.ask('health', 'check', {}, function(err, res) {
        res.data.ok ? next.promote() : next.kill();
    });
});
```

Sub-mounts are reconciled across reloads: same path + same file → kept alive (no restart); same path + new file → nested blue-green reload; dropped by v2 → graceful unmount.

### Custom Mount Types

```js
ev.mountType('my-transport', {
    crossBoundary: true,
    ownedHandle: true,
    reloadable: true,
    available: function() { return typeof MyAPI !== 'undefined'; },
    boot: function(m, callbacks) { /* → handle */ },
    send: function(handle, msg, transferList) { },
    terminate: function(handle) { },
    onShutdownComplete: function(handle, cb) { /* → cleanupFn */ }
});
```

### Declarative Mounts

```js
// mounts.json
{
    "http":        { "file": "./http_service.js", "type": "require" },
    "db":          { "file": "./db_worker.js",    "type": "thread"  },
    "plugins/ext": { "file": "./plugin.js",       "type": "vm"      }
}

// server.js
mountLoader(ev, path.resolve(DIR, 'mounts.json'), DIR);
```



## 🖱️ DOM Binding (Browser)

One API for user interactions AND observers. Auto-detects the mechanism.

```js
// User interactions — event delegation via a single document listener per type
ev.dom('/ui/login', '#btn', 'click', handler);
ev.dom('/ui/search', '#input', 'input', { debounce: 300 }, handler);
ev.dom('/ui/keys', document, 'keydown', { keys: { ctrl: true, key: 's' } }, handler);
ev.dom('/ui/card', element, 'mouseenter', handler);   // enter/leave synthesized correctly

// Observers — resize, scroll, visibility
ev.dom('/ui/panel', element, 'resize');      // ResizeObserver (or native for window)
ev.dom('/ui/chat', element, 'scroll');       // scroll + content-growth detection
ev.dom('/ui/post', element, 'visibility');   // IntersectionObserver (1% steps)

// Lifecycle
ev.dom('/ui/comp', '.component', 'exist');   // fires when element appears in DOM
ev.dom('/ui/comp', element, 'removed');      // fires when element leaves the DOM
ev.dom('/ui/hero', '#img', 'load');          // load / error / abort for img, iframe...

// Batch observation — CSS selector watches ALL matching elements,
// including ones added later (auto-observe via MutationObserver)
ev.dom('/feed/images', '.lazy-img', 'visibility');

// One binding per path — calling ev.dom again on the same path REPLACES it.
// Want two handlers on one element? Use two paths.

// Cleanup — same as everything else
ev.clear('/ui/**');
```

Handlers can also be omitted — events are emitted on the path, and you attach listeners separately with `ev.on(path, [event], handler)`.

### Observer Data

```js
// Scroll (vertical + horizontal, with prev)
e.data.fromTop;  e.data.fromBottom;  e.data.totalHeight;
e.data.fromLeft; e.data.fromRight;   e.data.totalWidth;

// Visibility (cross-checked against CSS visibility/display)
e.data.ratio;    // 0.00–1.00
e.data.visible;  // boolean

// Resize
e.data.width;  e.data.height;  e.data.prev.width;
// Window resize also reports e.data.pixelRatio (zoom / screen-density changes)
```



## 🌍 Browser Runtime: Tabs & Service Worker

When the SW bundle runs, the Service Worker becomes a **tab registry and router**:

```
/tabs                  — { active: tabId, activeState, count }   (live state)
/tabs/[tabId]          — { tabId, state, url, lastActivity, ... } (per tab)
/tabs/[tabId]/...      — routed to that specific tab
/tabs/active/...       — routed to the currently active tab
/tabs/...              — broadcast to all tabs
```

```js
// From the SW — ask a specific tab:
ev.ask('/tabs/' + tabId + '/ui/dialog', 'confirm', data, cb);

// From the SW — notify the active tab:
ev.emit('/tabs/active/notifications', 'show', { text: 'Done!' });

// From a tab — everything on the SW lives under /main:
ev.ask('/main/cache/users', 'get', {}, cb);

// From a tab — cross-tab broadcast (BroadcastChannel):
ev.emit('/data', 'sync', payload, { to: 'all-tabs' });
```

### Browser Subscriptions (tab ↔ SW)

The same subscription mechanism from Node runs in the browser. A tab's `ev.on` on any non-local absolute path auto-subscribes through the Service Worker; the SW then forwards **only matching emits to that tab**, instead of broadcasting everything to everyone:

```js
// In a tab — receives ONLY /main/cache emits of type 'changed':
ev.on('/main/cache/users/*', ['changed'], handler);
```

Excluded as local-only (never subscribed): `/system/**`, `/_dom/**`, `/page**`, and the tab's own `/tabs/<id>` subtree (the SW prefix-routes that directly — subscribing would double-deliver).

**The SW is ephemeral by design** — browsers kill it after ~30s idle and re-run its script from scratch, wiping any in-memory table. deep-events handles this by making the **tab the source of truth**: on every signal that a (new) SW is listening — liveness recovery, `bootTime` change, T_TAB_REQUEST, `controllerchange` — the tab re-sends its full subscription set (idempotent on the SW side).

**Legacy rule (also the restart grace):** a known tab with zero recorded subscriptions receives *every* emit — the old broadcast behavior. This makes the upgrade fully backward compatible with older bundles, and means a freshly-rebooted SW (empty table) gracefully degrades to broadcast until re-registrations stream back in. No timers, no lost events.

The runtime also handles, automatically:

- **Tab state reporting** — visibility, URL, and activity are pushed to the SW (debounced; immediate for hidden/frozen/terminated so nothing is lost when a tab dies).
- **Active-tab election** — the SW tracks which tab is frontmost and broadcasts `/tabs` `'updated'` on change.
- **SW liveness** — tabs ping the SW; on silence the tab marks `/system/sw` `{ alive: false }`, queues state reports, and attempts wake-up. `bootTime` comparison detects SW restarts and triggers re-registration.
- **SW keep-alive** — a dedicated Web Worker pings the SW so the browser doesn't kill it while your app is open (survives background-tab timer throttling).
- **SW lifetime during work** — `ev.keepAlive(event)` / `ev.workDone()` hold the SW open across multi-step async flows (3s grace period).
- **Discovery loop** — the SW reconciles its registry against `clients.matchAll()` every 5s: dead tabs are cleaned up, unknown tabs are asked to identify.
- **SW self-update** — `ev.ask('/main/system/sw', 'checkUpdate', {}, cb)` forces a script re-fetch; `skipWaiting` + `clients.claim` make the new version take over immediately.

Tab identity: each tab gets `ev.tabId` (random uint32) and `ev.self = '/tabs/<id>'`. Transport capabilities are queryable at `/system/transport`.



## 🍳 Cookbook — Common Scenarios

Short, real recipes. Each one is complete.

### 1. SPA page lifecycle — everything dies on navigation

```js
function openDashboard() {
    ev.dom('/page/dash/search', '#q', 'input', { debounce: 250 }, onSearch);
    ev.dom('/page/dash/chart',  '#chart', 'visibility', onVisible);
    ev.interval('/page/dash/refresh', reload, 30000);
    ev.ask('/services/db/stats', 'load', {}, { owner: '/page/dash' }, render);
    ev.on('/app/user', ['changed'], onUser, { snapshot: true });
}
function closeDashboard() {
    ev.clear('/page/dash/**');   // DOM bindings, timer, pending ask — all gone
}
```

### 2. Rate limiter — the classic Redis pattern

```js
function allow(userId) {
    var key = '/ratelimit/' + userId;
    var n = ev.incr(key);
    if (n === 1) ev.expire(key, 60);   // first hit opens a 60s window
    return n <= 100;                    // 100 requests/minute
}
```

### 3. Config that consumers can join late

```js
// producer (anywhere):
ev.set('/app/config', { theme: 'dark', lang: 'he' });
// consumer (any time — before or after):
ev.on('/app/config', ['changed'], applyConfig, { snapshot: true });
```

### 4. CPU work off the main thread — with zero-copy

```js
ev.mount('/services/hash', { file: './hash.worker.js', start: 'onDemand', idle: 60000 });

var buf = new Uint8Array(fileBytes);
ev.ask('/services/hash/sha256', 'run', { data: buf }, { transfer: [buf] }, function(err, res) {
    res.data.hex;   // buf was TRANSFERRED (zero-copy) — don't touch it here anymore
});
// onDemand: worker boots on first ask. idle: dies after 60s unused. Free elasticity.
```

### 5. File watching via subscriptions (no polling, no ask-loops)

```js
// fs service (a thread) just emits:
ev.emit('/services/fs/changed', 'file', { path: p });
// any other worker, anywhere in the tree:
ev.on('/services/fs/changed', ['file'], invalidateCache);   // absolute path → auto-subscribes
```

### 6. Zero-downtime deploy with verification

```js
ev.reload('/services/api', { file: './api-v2.js', keep: true }, function(err, next) {
    next.ask('health', 'check', {}, function(err, res) {
        if (!err && res.data.ok) next.promote();   // drain v1, v2 takes over — zero drops
        else next.kill();                          // v2 discarded, v1 never stopped
    });
});
```

### 7. Cross-tab presence

```js
// each tab:
ev.on('/tabs', ['updated'], function(e) {
    renderPresence(e.data.count, e.data.active === ev.tabId);
});
// the SW maintains /tabs automatically — count, active tab, per-tab state.
```

### 8. Semi-trusted plugins in a sandbox

```js
ev.mount('/plugins/' + name, {
    file: pluginFile, type: 'vm',
    sandbox: { config: pluginConfig }        // gets ev + console + timers, no require/process
});
ev.ask('/plugins/' + name + '/render', 'run', input, { timeout: 1000 }, cb);
ev.unmount('/plugins/' + name);              // and it's gone
```



## 🔍 Debugging & Troubleshooting

### ev.trace()

```js
ev.trace(true);   // in the EMITTING context; per-context — enable inside workers too
```

Every cross-boundary hop logs a `[trace]` line and lands in a 300-entry ring buffer, retrievable after the fact from any context (including inside workers):

```js
ev.ask('/system/trace', 'dump', {}, { local: true }, function(err, res) {
    res.data.entries;   // [{ t: timestamp, m: 'ask /calc/add:go -> mount /svc (status running)' }, ...]
});
ev.ask('/system/trace', 'clear', {}, { local: true }, cb);
```

Reading trace lines:

| line | meaning |
|---|---|
| `emit P:T -> sub match M (pattern X) T_EMIT sent` | main forwarded the emit to worker M — if M still doesn't react, the problem is inside M (types? filter? match?) |
| `ask P:T -> mount M (status S)` | ask routed to mount M; non-`running` status explains delays |
| `ask P:T -> no local handler, trying subscribers` | normal fallback; nothing after it = no subscription matched |
| `inbound ask P:T -> 0 local listener(s)` | the worker got the ask but has no matching listener |
| *(no lines at all)* | the emit never left its context — see checklist step 1 |

OFF cost: one boolean read per hop.

### "My event isn't arriving" — checklist

1. **Absolute vs relative.** Inside a worker only paths starting with `/` cross the boundary. `ev.on('feed', ...)` is local-only. With `de.create()`, relative paths resolve to `<mountPath>/...` — still local; use `'../sibling'` or a full absolute path to reach out.
2. **Registration order.** (a) *mount before on*: a worker listening on its own sub-mount's paths must `ev.mount('child', ...)` **before** `ev.on('/me/child/...')` — earlier listeners are treated as purely local and never upgraded. (b) *boot buffering*: subscriptions during `starting` are buffered until ready; if your boot **depends** on the subscribed event, pass `{ immediate: true }` or you deadlock.
3. **Wildcard position.** `*` = one segment, `**` = zero or more, at any position — identical in-process and cross-boundary. One limitation: a wildcard *above* a sub-mount boundary (`/A/*` when the child is `/A/sub`) is not propagated; at or below the mount path works fully.
4. **Mount status.** `console.log(ev.status())`. `starting` queues asks and buffers subs; `crashed`/`failed` → check `restart`/`maxRestarts` and worker stderr; `reloading`/`draining` → mid blue-green, traffic routes to the NEW worker.
5. **Types array.** `ev.on(path, ['changed'], h)` fires only for `'changed'` — emitting `'change'` silently misses. Omit the array for catch-all.
6. **Event object lifetime.** Event objects are pooled and recycled after the handler returns. In async callbacks, read from a captured `var data = e.data;` — never hold `e` itself.
7. **Cleared by a parent?** `ev.clear('/page/**')` kills every listener/timer/pending ask under it, including other modules'. Check for broad clears upstream.

### Ask error codes

| code | meaning |
|---|---|
| NO_HANDLER | nothing listened — locally or via subscriptions |
| TIMEOUT | a listener existed but never replied within `timeout` |
| CLEARED | the ask's `owner` path was cleared mid-flight |
| WORKER_DEAD | target mount not running (crashed/stopped/failed) |
| MOUNT_FAILED | driver-level boot/config failure |
| BOOT_TIMEOUT | worker never called `ev.ready()` within `bootTimeout` |
| NO_MOUNT | `ev.reload`/`ev.unmount` on a path that isn't mounted |



## ⚡ Performance & Batching

| Operation | Browser | Server |
|---|---|---|
| emit (single listener) | >1M/sec | >5M/sec |
| emit (wildcard) | >500K/sec | >1M/sec |
| subscription match (hot loop) | — | ~20 ns/check |
| clear (100 listeners) | <1ms | <0.5ms |
| cross-boundary ask/reply (sequential) | — | ~8K/sec |
| cross-boundary ask/reply (concurrent) | — | ~87K/sec |
| post-burst sequential asks | — | ~10.6K/sec |

Hot-path design: event object pool, pre-allocated dispatch buffers, path/segment caches, insertion-sort priorities, subscription patterns pre-split at registration (single path split per emit, exact/literal fast paths).

### Message Batching

Every postMessage boundary is wrapped in a rate-detecting batcher. Below the transport's threshold, every message ships immediately — zero added latency. Above it, messages coalesce into single `T_BATCH` envelopes (order preserved, nothing dropped; the *fixed per-call cost* of postMessage — structured clone, event allocation, wakeups — is what gets amortized).

**Per-transport thresholds** — derived from measured send costs (`threshold = 10% CPU budget / avgSendCost`), not guessed:

| boundary | threshold | measured send cost |
|---|---|---|
| worker_threads (Node) | 8,000/sec | ~2.5µs small / ~11µs 4KB binary |
| MessagePort peer channel | 8,000/sec | ~1.7µs |
| child process (CBOR/stdio) | 6,000/sec | ~4.5µs |
| tab ↔ Web Worker | 2,000/sec | ~3µs desktop, ~59µs 4KB on 6x-throttled CPU |
| tab ↔ Service Worker | 2,000/sec | send is worker-class; the expensive part is round-trip (~0.5ms SW scheduling — batching can't help that; prefer emits/subscriptions or concurrent asks for SW-heavy flows) |

Thresholds anchor to *weak-device* (CPU-throttled) measurements: entering burst too early on a fast machine is nearly free, entering too late on a weak one is the freeze scenario. Custom drivers set `batchThreshold` in `ev.mountType`; calibrate on your hardware with `node test/bench-transports.js` (Node) and `test/bench-browser.html` (browser — run it in a background tab too).

**Flush scheduling** never uses a throttleable timer: `setImmediate` in Node, a MessageChannel self-task in browsers/workers/SW (a macrotask that background tabs cannot clamp — unlike `setTimeout(1)`, which is clamped to 1000ms+ in background tabs). Macrotask (not microtask) on purpose: it lands *behind* messages already queued, so they join the batch first — better latency *and* better coalescing.

Control messages (T_READY, T_PING/PONG, T_SHUTDOWN*, T_PEER_*) are never batched. Inspect any batcher at runtime: `send.stats()` → `{ scheduler, burstMode, totalBatched, burstEntries, pending, ... }`.



## 🌎 Environment Detection

`ev.env` reports where this instance is running — set once at load:

```js
ev.env = {
    web, webWorker, serviceWorker,          // browser contexts
    node, nodeMain, nodeWorker,             // Node contexts
    bun, deno,                              // alt runtimes
    electron, electronMain, electronRenderer
}
```

Browser tab identity: `ev.tabId` (random uint32) and `ev.self = '/tabs/<id>'`. Inside Node workers `ev.self` is the mount path. Transport capabilities: `ev.get('/system/transport')`; SW liveness: `ev.get('/system/sw')`.



## 🧪 Testing

The `test/` folder ships a full suite — run from the package root:

```bash
node test/run-all.js
```

10 test files, covering:

| file | covers |
|---|---|
| test-fixes-unit.js | wildcard matcher matrix, batcher burst exit + binary sizing, ev.once argument forms, incr+TTL, inbound-ask leak cleanup |
| int1-wildcard-sub.js | thread subscription with mid-path `*/*` wildcards |
| int2-bootqueue.js | onDemand require boot-queue flush reaching subscribed threads |
| int3-parent-hears-child.js | parent thread hearing its own sub-thread via direct peer channel (exactly-once) |
| int4-zombie.js | no zombie restart after worker self-exit during unmount |
| int5-sanity.js | ask/reply, genuine crash → restart, ev.clear cancelling subscriptions |
| int6-reload-under-load.js | blue-green: 400 asks streaming through a reload, zero dropped |
| int7-reload-subs.js | blue-green: subscription handover v1→v2, no duplicates, no stragglers |
| int8-reload-keep.js | blue-green: keep → verify via handle.ask → both kill and promote paths |
| int9-reload-nested.js | blue-green: nested mounts kept alive, peer channel re-wired to v2 |

Benchmarks: `test/bench-transports.js` (Node send costs + derived thresholds), `test/bench-browser.html` + `test/bench-sw.js` (browser calibration), `test/perf.js` (post-burst latency scenario).



## 🛣 Roadmap

✅ = Complete  🔄 = In Progress  ⏳ = Planned

### ✅ Complete

| Status | Item |
|:---:|---|
| ✅ | Core event bus — on, emit, off, clear, wildcards, priorities, filters, binary match |
| ✅ | Ask / reply with correlation IDs, owner-based cancellation |
| ✅ | Data engine — get/set/merge/del, counters, TTL, nx/xx, mget/mset, keys, change detection |
| ✅ | Timer system — path-owned, atomic replace, drift-corrected |
| ✅ | Scoping — ev.create (mount-aware), ev.path, '..' resolution |
| ✅ | Node mounts — thread, process, port, require, import, vm |
| ✅ | Browser mounts — sw, worker, port, require, import |
| ✅ | Blue-green reload with drain, state transfer, keep/promote/kill |
| ✅ | Graceful shutdown with sub-mount lifecycle |
| ✅ | Ping/pong health checks, idle shutdown, crash restart policies |
| ✅ | Cross-boundary subscriptions (wildcards, refcounted) |
| ✅ | Direct thread↔thread peer channels (MessageChannel) |
| ✅ | Adaptive message batching at every postMessage boundary |
| ✅ | CBOR framing for child-process transport |
| ✅ | Browser transport — tab registry, SW routing, liveness, keep-alive, discovery |
| ✅ | ev.dom — delegation, observers (visibility/resize/scroll), lifecycle (exist/removed/load) |
| ✅ | Browser subscriptions — tab↔SW targeted delivery, SW-restart resilient, legacy-compatible |
| ✅ | ev.trace — cross-boundary flow tracing + ring buffer (/system/trace) |
| ✅ | Measured per-transport batch thresholds + throttle-proof flush scheduler |
| ✅ | Test suite — 10 files: cross-boundary matrix, blue-green, lifecycle edge cases |
| ✅ | Declarative mount management (mount_loader + JSON) |

### 🔄 In Progress

| Status | Item | Notes |
|:---:|---|---|
| 🔄 | Platform APIs | /system/network, battery, GPS, display |
| 🔄 | SPA router service | Route matching, guards, page lifecycle |
| 🔄 | Client services | page_manager, render, content_provider |

### ⏳ Planned

| Status | Item | Notes |
|:---:|---|---|
| ⏳ | SharedWorker mount type | — |
| ⏳ | Electron mount type | BrowserWindow via ipcMain/ipcRenderer |
| ⏳ | Dev tools | Chrome extension, visual flow inspector (ev.trace core is done) |
| ⏳ | Event tracing | Cross-boundary `e.id` + `e.trace[]` |
| ⏳ | Middleware / interceptor API | — |
| ⏳ | TypeScript typings | IDE support |
| ⏳ | Framework integration | React hooks, Vue composables |
| ⏳ | Benchmarks & profiling | CPU, memory, latency |

_Community contributions welcome!_
_Please ⭐ star the repo to follow progress._



## 📜 License

**MIT**

```
Copyright © 2025 colocohen

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
