---
summary: "End-to-end runtime architecture: the Gateway process, its internal loops, and how it communicates with the outside world"
read_when:
  - You want to understand what OpenClaw is doing at runtime
  - You are onboarding and need a mental model before reading deeper docs
  - You are debugging a hang, a missed message, or an unexpected agent run
title: "Runtime Architecture"
---

# Runtime Architecture

This document answers four questions:

1. [What process is running?](#the-gateway-process)
2. [What loops does that process run?](#internal-loops)
3. [What are the steps inside each loop?](#loop-steps-in-detail)
4. [How does it communicate with the outside world?](#external-communication)

For the protocol wire format, see [Gateway Protocol](/gateway/protocol).
For agent loop details, see [Agent Loop](/concepts/agent-loop).

---

## The Gateway process

OpenClaw runs as **one long-lived process** called the **Gateway**:

```bash
openclaw gateway --port 18789
```

There is exactly one Gateway per host (or per isolated profile). It owns:

- All messaging channel connections (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, Matrix, and plugin channels).
- The WebSocket control plane for clients and nodes.
- The HTTP API surface (OpenAI-compatible endpoints, Control UI, hooks, MCP loopback).
- All in-process agent execution.

The Gateway is supervised by the OS service manager (launchd on macOS, systemd on Linux, Scheduled Task on Windows). On the macOS desktop app, it runs embedded inside the app process. In all cases it is the same single Node.js process.

---

## Internal loops

The Gateway does not have one monolithic loop. It runs several **cooperative loops** layered on top of the Node.js event loop:

| Loop | Purpose | Cadence |
|------|---------|---------|
| **WebSocket I/O loop** | Accept frames, dispatch method handlers, push events | Continuous (I/O-driven) |
| **Agent / command queue** | Serialize and execute agent runs per session | Triggered by inbound messages or RPC calls |
| **Heartbeat loop** | Periodic agent turns in the main session | Configurable (default `30m`) |
| **Cron loop** | Scheduled background agent jobs | Configured `cron` entries |
| **Maintenance timers** | Channel health checks, model-pricing refresh, media cleanup, task-registry pruning | Periodic (minutes to hours) |
| **Config reload watcher** | Apply hot-safe config changes or trigger restart | File-watch or polling |

All of these are cooperative: they share the same Node.js event loop and never block it. Long-running work (model calls, shell commands) runs inside async tasks that yield between steps.

---

## Loop steps in detail

### Agent / command queue loop

This is the core loop that turns an inbound message into a reply:

```
Inbound message (channel or RPC)
        │
        ▼
  1. Route + queue
     • Channel plugin delivers message to auto-reply pipeline.
     • Message is placed on the session lane queue (one active
       run per session key at a time).
     • Typing indicator fires immediately (if supported).
        │
        ▼
  2. Dequeue + lock session
     • Queue drains up to maxConcurrent runs globally.
     • Session write lock is acquired.
     • Session transcript is loaded from disk.
        │
        ▼
  3. Resolve model + auth
     • Active model provider is resolved (config, plugin hooks,
       before_model_resolve hook).
     • Auth profile and API keys are resolved from secrets snapshot.
        │
        ▼
  4. Assemble context + system prompt
     • Bootstrap files are injected (MEMORY.md, AGENTS.md, etc.).
     • Skills snapshot is loaded.
     • Plugin hooks run (before_prompt_build, before_agent_start).
     • System prompt is finalized with model token limits enforced.
        │
        ▼
  5. Model inference (pi-agent-core)
     • Messages are sent to the model provider (streaming).
     • Assistant deltas stream back as "assistant" events.
     • If context is too large, compaction runs first and the
       run retries.
        │
        ▼
  6. Tool execution (zero or more rounds)
     • Model emits tool calls; Gateway validates and dispatches them.
     • Tool start/update/end events are emitted on the "tool" stream.
     • Exec commands go through sandbox policy + approval gates.
     • Tool results are written back into the messages list.
     • Loop returns to step 5 until the model stops calling tools.
        │
        ▼
  7. Reply shaping + delivery
     • Final payloads are assembled (text, tool summaries, errors).
     • Silent tokens (NO_REPLY) are filtered.
     • Plugin hook runs (before_agent_reply, agent_end,
       message_sending).
     • Message is delivered to the originating channel.
        │
        ▼
  8. Persist + release lock
     • Updated session transcript is written to disk.
     • Session write lock is released.
     • Lifecycle "end" event is emitted to all subscribers.
```

### Heartbeat loop

The heartbeat loop runs a lightweight version of the agent loop on a fixed interval:

1. Heartbeat timer fires (default `30m`).
2. Gateway checks active-hours window; skips if outside the window.
3. If a `HEARTBEAT.md` file exists in the workspace, it is read and only due `tasks:` entries are injected. If nothing is due, the tick is skipped.
4. A synthetic "heartbeat" user message is enqueued on the main session lane, identical to any other agent run.
5. The full agent loop (steps 2–8 above) runs. The model replies `HEARTBEAT_OK` if nothing needs attention; otherwise it sends an alert.
6. `HEARTBEAT_OK`-only replies are suppressed before delivery.

### Cron loop

Cron jobs are similar to heartbeats but have explicit schedules and fully isolated sessions:

1. Gateway's cron service checks pending jobs at each tick.
2. A job whose next-run time has passed is dequeued on a separate `cron` lane (parallel to `main`, not blocking inbound replies).
3. The agent loop runs in an isolated session (no conversation history by default).
4. Results are delivered to the configured target channel/recipient.

### WebSocket I/O loop

The WS I/O loop is purely event-driven:

1. Client sends a JSON frame (`req` or `connect`).
2. Gateway validates the frame schema and authenticates the sender.
3. The matching method handler is invoked (e.g. `agent`, `send`, `health`).
4. The handler returns a `res` frame immediately (for sync methods) or returns an ack and later pushes `event` frames (for streaming methods like `agent`).
5. Gateway pushes unsolicited `event` frames for presence changes, heartbeat ticks, channel health updates, and agent stream events.

---

## External communication

The Gateway has three communication surfaces:

### 1. Messaging channels (inbound + outbound)

Each configured channel runs a **persistent connection** inside the Gateway:

| Channel | Protocol |
|---------|----------|
| WhatsApp (web) | Baileys (WebSocket to Meta servers) |
| Telegram | grammY long-polling or webhook |
| Slack | Bolt.js (WebSocket or HTTP Events API) |
| Discord | Discord.js (WebSocket gateway) |
| Signal | signal-cli subprocess |
| iMessage | AppleScript / BlueBubbles bridge |
| Matrix (plugin) | Matrix JS SDK (sync loop) |
| Other plugins | Plugin-owned transport |

Inbound messages arrive from the channel, enter the auto-reply pipeline, and trigger the agent/command queue loop described above. Outbound replies are sent back through the same channel connection.

### 2. WebSocket control plane (clients and nodes)

All first-party clients (CLI, macOS app, web Control UI, iOS/Android apps) and headless nodes connect to the same WebSocket server on `localhost:18789` (default):

```
ws://127.0.0.1:18789
```

Connection lifecycle:

```
Gateway → Client: event:connect.challenge  { nonce, ts }
Client  → Gateway: req:connect             { role, auth, device, caps, ... }
Gateway → Client: res:connect              hello-ok { health, presence, ... }

--- after handshake ---
Client  → Gateway: req:agent               { message, ... }
Gateway → Client: res:agent                { runId, status:"accepted" }
Gateway → Client: event:agent (streaming)  { stream:"assistant", delta, ... }
Gateway → Client: res:agent (final)        { status:"ok", summary, ... }
```

Remote access is supported through Tailscale (preferred) or an SSH tunnel. The same auth token and handshake apply over the tunnel.

### 3. HTTP API

The Gateway exposes HTTP endpoints on the same port as WebSocket:

| Path | Purpose |
|------|---------|
| `GET /v1/models` | OpenAI-compatible model list |
| `POST /v1/chat/completions` | OpenAI-compatible chat completions |
| `POST /v1/responses` | OpenAI Responses API |
| `POST /v1/embeddings` | OpenAI-compatible embeddings |
| `POST /tools/invoke` | Tool invocation endpoint |
| `GET /__openclaw__/` | Control UI (web admin) |
| `GET /__openclaw__/canvas/` | Agent-editable canvas |
| `POST /hooks/*` | Incoming webhook handlers |
| `GET /mcp` | MCP loopback server |

HTTP endpoints use the same auth boundary as the WebSocket control plane.

---

## Summary diagram

```
                        ┌────────────────────────────────────────────┐
                        │              Gateway process               │
                        │                                            │
  Messaging channels ───┤  Channel plugins (Telegram, Discord, ...)  │
  (WhatsApp, Telegram,  │         │ inbound messages                 │
   Slack, Discord, ...)  │         ▼                                  │
                        │  Auto-reply pipeline → command queue       │
  WebSocket clients ────┤  ┌────────────────────────────────────┐    │
  (CLI, macOS app,      │  │         Agent loop (per run)       │    │
   iOS/Android, web UI) │  │  route → lock → model → tools      │    │
                        │  │  → reply → persist → unlock        │    │
  HTTP clients ─────────┤  └────────────────────────────────────┘    │
  (Open WebUI, curl,    │                                            │
   automations)         │  Heartbeat timer ──► heartbeat run        │
                        │  Cron scheduler  ──► cron run             │
                        │  Maintenance timers (health, cleanup, ...) │
                        └────────────────────────────────────────────┘
```

---

## Related

- [Agent Loop](/concepts/agent-loop) — detailed agent execution cycle
- [Gateway Architecture](/concepts/architecture) — WebSocket components and flows
- [Command Queue](/concepts/queue) — concurrency and queue modes
- [Heartbeat](/gateway/heartbeat) — periodic agent turns
- [Gateway Protocol](/gateway/protocol) — WebSocket wire protocol
- [Gateway Runbook](/gateway) — startup, operations, and supervision
