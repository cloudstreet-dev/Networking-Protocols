# WebSockets

**WebSockets** provide full-duplex, bidirectional communication over a single TCP connection. Unlike HTTP's request-response model, WebSockets allow both client and server to send messages at any time.

## Why WebSockets?

HTTP is request-response: client asks, server answers. But many applications need real-time, bidirectional communication:

```
HTTP Limitations:
  - Client must initiate every exchange
  - Server can't push data spontaneously
  - New request needed for each interaction
  - Header overhead for every message

Workarounds before WebSockets:
  Polling:        Client asks "any updates?" every N seconds
  Long-polling:   Server holds request until data available
  Server-Sent:    Server streams events (one-way only)

All have overhead, latency, or direction limitations.
```

## WebSocket Advantages

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WebSocket Benefits                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Full-Duplex: Both sides send simultaneously                        │
│  Low Latency: No per-message handshake                              │
│  Low Overhead: 2-10 bytes per frame (vs ~100+ for HTTP)             │
│  Persistent: Single connection for entire session                   │
│  Push: Server sends without client request                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Perfect for:
  - Chat applications
  - Live notifications
  - Real-time collaboration
  - Live sports/stock updates
  - Online gaming
  - IoT device communication
```

## The Protocol

WebSocket starts as HTTP, then "upgrades" to a different protocol:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WebSocket Connection                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. HTTP Request with Upgrade header                                │
│  2. Server responds 101 Switching Protocols                         │
│  3. Connection becomes WebSocket                                    │
│  4. Bidirectional frames flow                                       │
│  5. Either side can close                                           │
│                                                                     │
│  ┌──────────┐                              ┌──────────┐             │
│  │  Client  │                              │  Server  │             │
│  └────┬─────┘                              └────┬─────┘             │
│       │                                         │                   │
│       │─── HTTP Upgrade Request ───────────────>│                   │
│       │<── HTTP 101 Switching ──────────────────│                   │
│       │                                         │                   │
│       │═══ WebSocket Connection ════════════════│                   │
│       │                                         │                   │
│       │─── Message ────────────────────────────>│                   │
│       │<── Message ─────────────────────────────│                   │
│       │<── Message ─────────────────────────────│                   │
│       │─── Message ────────────────────────────>│                   │
│       │                                         │                   │
│       │─── Close Frame ────────────────────────>│                   │
│       │<── Close Frame ─────────────────────────│                   │
│       │                                         │                   │
│       ╳                                         ╳                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## What You'll Learn

1. **The Upgrade Handshake**: How HTTP becomes WebSocket
2. **Full-Duplex Communication**: Frame format and messaging
3. **WebSocket Use Cases**: When and why to use WebSockets
