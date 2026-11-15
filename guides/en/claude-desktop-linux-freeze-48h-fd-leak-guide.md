---
title: "Debugging Claude Desktop Periodic Freezes: 27-Byte Unread Responses and Chromium FD Leaks"
description: "Tracing why Claude Desktop on Linux freezes after long sessions: from 1.5TB core dumps, Chromium shared memory pipes, and Cache-Control logic, to a local Caddy header rewrite fix."
pubDate: 2026-11-15
author: "Koko"
category: "Troubleshooting & Architecture"
tags: ["Claude Desktop", "Chromium", "Linux", "Electron", "Reverse Engineering", "Bug Hunting"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/claude-desktop-linux-freeze-48h-fd-leak-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Wednesday afternoon, my workstation stalled out.

The mouse moved at about two frames per second. Typing took three to four seconds for letters to appear.

I pulled up a terminal and ran uptime. It took twenty seconds for the command to return. Load average sat at 46.34. On a 16-core workstation, that meant nearly three jobs stacked on every core. What script could possibly run away like that? My first suspicion was a broken build process, but ps aux showed nothing rogue.

I checked Linux kernel I/O pressure metrics:

```bash
# Check kernel I/O pressure metrics
cat /proc/pressure/io
```

The full 10-second metric hit 85.3. The entire NVMe SSD was pinned by non-stop writes.

Switching over to Claude Desktop on the right monitor, the window had turned blank, leaving only skeleton loader placeholders:

<img src="https://img.freeaidog.com/images/blog/claude-desktop-frozen-skeleton.webp" alt="Claude Desktop frozen skeleton loader screen" />

Mouse clicks did nothing. The interface was dead.

Checking the process list revealed systemd-coredump thrashing the storage. Claude's renderer process had crashed with a SIGILL signal, and the OS was dumping its entire memory image to disk. It wrote 92.4GB in five minutes before timing out and getting terminated without leaving a usable core file. How are you supposed to debug that?

Running this client on Linux, it dies every couple of days like clockwork. Either a blank window or a complete lockup. The logs only repeat one vague message: main webview unresponsive, will kill and reload. It never successfully reloaded once.

I spent two weeks patching and debugging before finding the culprit: a file descriptor leak triggered by a 27-byte telemetry reply.

## Why Did a Crash Dump Swell to 1.5TB?

Why would a chat window renderer need a 1.5TB dump? That is the result of Chromium virtual address space reservations.

The desktop app bundles Electron 44, which runs on Chrome 152. At startup, V8 reserves a 1TB virtual address space for its sandbox, surrounded by 64GB and 288GB guard zones. Combined with PartitionAlloc and the GC heap, virtual address reservations easily exceed 1.3TB.

Under normal Linux conditions, madvise can pass DONTDUMP to tell the kernel to skip empty virtual ranges during crashes.

However, the renderer seccomp policy baseline_policy.cc strictly filters advice flags. It permits only six flags: DONTNEED, WILLNEED, RANDOM, REMOVE, NORMAL, and FREE. MADV_DONTDUMP is denied with EPERM.

When the crash hits, the kernel streams memory over a pipe to systemd-coredump. A pipe cannot seek across sparse holes, so unallocated virtual space turns into real zero-byte writes to disk.

Original author Vladimir Petrzhikovskii detailed in blog.dfb.sh/blog/claude-desktop-freeze that he built a Rust tool using lseek SEEK_DATA and SEEK_HOLE to search the 1.5TB sparse dump in 0.7 seconds.

## Unraveling 942 Shared Memory Regions and 27-Byte Pings

Inspecting the dump showed the renderer holding nearly 1,000 open file descriptors:

```bash
# Inspect file descriptor count for renderer process
ls -l /proc/$RENDERER_PID/fd | wc -l
```

Among them, 942 pointed to 2MB POSIX shared-memory regions. Scanning 938 of those regions with strings revealed identical 27-byte JSON payloads:

```json
{
  "refresh_after_seconds": 0
}
```

The process held over 900 shared memory regions for a 27-byte string. Why would a presence-clearing response need a refresh directive?

Tracing V8 heap maps revealed the calls came from the Code session tab.

The tab runs presence telemetry. When the window is hidden or idle, frontend scripts issue a clear ping:

```http
POST /v1/code/channels/session-id/client/presence HTTP/1.1
Host: api.anthropic.com
Content-Type: application/json

{"client_id": "client-token", "clear": true}
```

The request sets keepalive: true. It tells the server the user is away, and the backend returns 27 bytes.

Because it was a fire-and-forget call, frontend code never read the response body. It caught errors with an empty catch and moved on. With the app running in the background, these calls piled up at dozens of requests per hour.

After 35 hours, open descriptors approached 1,000.

## Root Cause: From no-store Headers to IPC Deadlocks

The server sent Cache-Control: no-store, and the frontend never consumed the response stream.

Chromium holds unread response streams in pipes until read or cancelled. Cache-Control: no-store aims to prevent disk caching, but here it prevented cleanup. In a minimal reproduction, manually triggering garbage collection reaped the JS response wrappers, but /proc/$PID/fd counts stayed unchanged. Should open handles not drop once garbage collected?

In fetch_manager.cc, Blink normally uses BufferingBytesConsumer to drain data from Mojo pipes into memory and close descriptors promptly.

A branch overrides this: if the response header contains Cache-Control: no-store, automatic buffering is skipped.

The intention is understandable: to prevent unread data from accumulating in memory when caching is disabled.

The cost is that data remains trapped inside the Mojo pipe until explicitly read or cancelled via body.cancel.

The server added no-store, and the client never read the stream. What happens if the frontend never reads? The descriptor stays open indefinitely.

Chromium follows the specification, and V8 GC cannot release C++ Mojo handles. Each ignored 27-byte ping holds a 2MB shared memory block and one file descriptor.

The renderer soft limit ulimit -Sn defaults to 1024:

```bash
# Check soft descriptor limits
ulimit -Sn
```

Once leaked pipes reach roughly 1,000, available descriptors run dry.

When the main thread executed an IndexedDB write requiring an IPC round-trip, the browser process attempted to transfer a new handle over a Unix domain socket.

The kernel hit descriptor exhaustion and flagged the socket with MSG_CTRUNC.

Older Mojo code in Chrome 152 fails to recover from missing handles, stalling the channel indefinitely.

Subsequent IPC calls lined up behind the blocked message. The main thread froze, and the window locked up dead.

## Upstream Feedback: Working as Intended and Automated Bot Closures

Vladimir filed bug reports with both upstreams.

First was the Chromium issue tracker, filing bug 582417463 to request that fetch throw a TypeError when descriptors run out rather than stalling the engine.

Chromium maintainers closed it the same day as a duplicate of issue 378822137. Is that working as intended?

The team response was clear: if a caller chooses not to read an available response, that is expected behavior. The browser cannot anticipate future reads, so it must keep the pipe open.

Second was Anthropic repository anthropics/claude-code.

Fixing this in client code requires one step: invoking cancel on the unconsumed response stream.

Issue 107887 was submitted.

Minutes later, an automated bot flagged it invalid and closed it with a note: Closing for now -- this does not appear to be about Claude Code.

The browser engine team considered the leak compliant, while the model team bot dismissed the report.

Users are left restarting frozen clients every two days.

## Local Mitigation: Rewriting Headers in Three Minutes

With upstream fixes stalled, local mitigation is the fastest route.

There are two approaches: monkey-patching fetch to cancel response streams, or stripping Cache-Control: no-store at the network layer.

Because Claude Desktop fetches frontend bundles dynamically, network rewrites are cleaner than modifying binaries.

If you run Caddy locally, a brief snippet resolves it:

```caddyfile
api.anthropic.com {
    reverse_proxy https://api.anthropic.com {
        header_up Host {upstream_hostport}
        @presence path /v1/code/channels/*/client/presence
        handle_response @presence {
            header -Cache-Control
        }
    }
}
```

Once the header is stripped, Chromium BufferingBytesConsumer automatically drains and closes pipes within 250ms. Descriptor counts remain flat at baseline.

Another approach uses remote debugging:

```bash
# Launch client with remote debugging port
claude-desktop --remote-debugging-port=9222
```

Testing the setup over seven continuous days, descriptor counts fluctuated between 36 and 41 without growing. RSS stayed flat, and system load remained below 0.1.

Both upstream issues remain closed: 582417463 marked duplicate, and 107887 marked invalid. Caddy rules in my dotfiles file claude-fix.caddy solve it on every new machine. How many 48-hour cycles will pass before upstream fixes land?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/claude-desktop-linux-freeze-48h-fd-leak-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

