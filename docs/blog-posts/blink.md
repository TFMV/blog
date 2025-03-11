# Blink: A Filesystem Watcher for the Speed-Obsessed

**Built for speed. Designed for simplicity.**

---

Performance has always been the undercurrent. The thing I can't ignore. How data moves. How systems breathe. How to strip away everything unnecessary until only speed remains.

I've pushed Go and Arrow to their limits — building ArrowArc to test the edge of what's possible (Go Fast or Go Home), rethinking data exchange (Redefining Data Engineering), exploring how Arrow is shaping modern systems (Go, Arrow, and the State of High-Performance Data Engineering).

Somewhere along the way, I got fixated on filesystems. Not just reading them, not just writing to them, but watching them, predicting them, making them move at the speed of thought.

Maybe it's an obsession. Maybe it's a sickness. Either way, I've spent an absurd amount of time figuring out how to walk them, watch them, and move data through them at speeds that make other tools feel like they're wading through molasses.

That's how Blink happened.

It started as a simple challenge: How fast can I walk a filesystem in Go? But once you start pulling that thread, you realize just how deep the rabbit hole goes.

- What if I could watch a filesystem instead of scanning it?
- What if I could stream those events without bogging down the system?
- What if I could make it work for anything — CLI, WebSockets, Server-Sent Events, webhooks — without reinventing the wheel every time?

That's Blink.

It's got:

- ✅ Recursive watching
- ✅ Event filtering
- ✅ Batching & debouncing
- ✅ Multiple output options (SSE, WebSockets, webhooks, CLI monitoring)
- ✅ Optimized polling for network filesystems

## The Hidden Complexity of Watching Files

Most developers take filesystem watching for granted — until they need to build something that relies on it. Then, the problems pile up.

Your simple fsnotify script hums along fine in local testing, then explodes in production — bogged down by thousands of redundant events. The same file change triggers multiple events. CPU usage spikes. Polling eats resources alive.

Because here's the truth:

- ❌ Filesystems are noisy. A single file write can trigger multiple events.
- ❌ Naive watchers burn CPU. Constant polling eats cycles for breakfast.
- ❌ Network filesystems are worse. Watching an NFS mount? Good luck.

Blink was built to kill that pain. It batches events, debounces redundant changes, filters the noise — and does it all without slowing you down.

And it does it without forcing you into a specific integration. Need SSE? It's there. WebSockets? No problem. Webhooks? Easy. Just want live output in your terminal? You got it.

## Dead Simple, Blazing Fast

Blink is built to be effortless. No complex setup, no endless configuration — just instant filesystem watching with sane defaults. Whether you're monitoring a directory, streaming events in real-time, or setting up webhook notifications, Blink makes it trivial.

Just run:

```bash
blink
```

…and you're done.

Of course, if you want more control, Blink gives you that too.

```bash
# Watch the current directory
blink

# Watch a specific directory
blink --path /path/to/watch

# Use a different port
blink --event-addr :8080

# Enable verbose logging
blink --verbose
```

Blink runs out of the box with zero configuration, but if you need more, it's built to scale effortlessly.

## Streaming Events in Real-Time

Want to stream file changes instead of polling for them? Blink makes it frictionless.

```bash
# Use WebSockets for event streaming
blink --stream-method websocket

# Use Server-Sent Events (SSE) for event streaming (default)
blink --stream-method sse

# Use both WebSockets and SSE simultaneously
blink --stream-method both
```

When using `--stream-method both`, Blink serves:

- SSE events at `--event-path` (default: `/events`)
- WebSocket events at `/events/ws`

No extra setup. Just pick your method and go.

## Powerful Filtering in One Command

Need to track only certain files or ignore noisy directories? Blink's filtering system makes it dead simple.

```bash
# Only watch for changes to JavaScript files
blink --include "*.js"

# Ignore node_modules directory
blink --exclude "node_modules"

# Only trigger on write events
blink --events "write"

# Ignore chmod events
blink --ignore "chmod"

# Complex filtering
blink --include "*.js,*.css,*.html" --exclude "node_modules,*.tmp" --events "write,create" --ignore "chmod"
```

No regex nightmares. No headaches. Just fast, readable filtering.

## Webhook Integrations

Blink plays nice with webhooks, letting you push events anywhere.

```bash
# Send webhooks to a URL
blink --webhook-url "https://example.com/webhook"

# Use a specific HTTP method
blink --webhook-method "PUT"

# Add custom headers
blink --webhook-headers "Authorization:Bearer token,Content-Type:application/json"

# Set timeout and retry options
blink --webhook-timeout 10s --webhook-max-retries 5

# Debounce webhooks
blink --webhook-debounce-duration 500ms

# Combine with filters
blink --include "*.js" --events "write" --webhook-url "https://example.com/webhook"
```

## Server-Sent Events: The Unsung Hero of Real-Time Systems

Everyone hypes WebSockets, but for something like Blink, Server-Sent Events (SSE) is an absolute killer feature.

SSE keeps things simple:

- ✅ Low overhead (no need for full-duplex communication)
- ✅ Auto-reconnect (without extra work on the client side)
- ✅ Built into every major browser (without extra dependencies)

If you just need a 'give me the damn file events' stream, SSE is the way to go. Blink leans into that, giving you an SSE endpoint that just works — no need for an SDK, no need for extra WebSocket logic, just a clean, efficient, "give me the damn file events" stream.

But I also get it — sometimes, you do need WebSockets. That's why Blink supports both. Choose what works for your system.

## Built for Performance

Blink isn't some toy project. It's engineered for serious workloads.

- 🟢 Parallel directory scanning with worker pools → No slow, single-threaded nonsense.
- 🟢 Non-blocking event processing → No bottlenecks. Events keep flowing.
- 🟢 Efficient memory use → Blink cleans up after itself.
- 🟢 Optimized filtering → Ignore what you don't need before it even reaches your pipeline.

And because I don't expect you to take my word for it, here's real benchmark data from Blink running on an M2 Pro:

### 🚦 Watcher Performance

- ⚙️ Operations/sec: 484 ops/sec
- ⏱️ Time/op: 2.07 ms/op
- 💾 Memory/op: 85,124 B/op
- 📦 Allocations/op: 530 allocs/op

### 🎯 Filter Performance

- ⚙️ 337.1M ops/sec
- ⏱️ 3.51 ns/op
- 💾 0 B/op
- 📦 0 allocs/op

## Where Blink Shines

Blink is for anyone who needs to watch filesystems in real-time — without headaches.

1️⃣ **Live Build & Reload for Dev Environments**

- 🔹 You're running a hot-reload server for a frontend project.
- 🔹 Blink watches your source files, triggers a build only when needed, and streams changes to the browser over SSE.

**CLI Example:**

```bash
blink --path ./src --events write,create --stream-method sse
```

2️⃣ **Log Monitoring & Security Audits**

- 🔹 You need real-time alerts for changes to sensitive system logs.
- 🔹 Blink watches /var/log, filters out the noise, and pushes security-critical events to your SIEM via webhook.

**Webhook Example:**

```bash
blink --path /var/log --webhook-url "https://security.example.com/webhook"
```

3️⃣ **Distributed File Sync & Backup Pipelines**

- 🔹 Your system needs to detect file changes and push updates to cloud storage.
- 🔹 Blink listens for new/modified files and triggers an upload the moment they appear.

**WebSocket Example:**

```bash
blink --path /mnt/data --stream-method websocket
```

## Blink is Open Source

If you've read this far, you're probably the kind of developer who gets excited about doing things the right way.

Blink is free, open source, and built to scratch a very specific itch:

- ✔ Give me real-time file change events
- ✔ Make it fast
- ✔ Make it scalable
- ✔ Make it easy to integrate

If that sounds like something you need, don't just read about it — run it. See it in action. It's fast. It's effortless. And, honestly? It's fun.

🔗 [GitHub: github.com/TFMV/blink](https://github.com/TFMV/blink)
🔗 [Docs: tfmv.github.io/blink](https://tfmv.github.io/blink)
