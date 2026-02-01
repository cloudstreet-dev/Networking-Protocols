# WebSocket Use Cases

WebSockets excel when you need persistent, low-latency, bidirectional communication. Understanding when to use them—and when not to—helps you choose the right tool.

## Ideal Use Cases

### Real-Time Chat

```
Chat requirements:
  ✓ Instant message delivery
  ✓ Multiple participants
  ✓ Typing indicators
  ✓ Presence status

WebSocket flow:
  User types ──[typing indicator]──> Server ──broadcast──> All users
  User sends ──[message]──────────> Server ──broadcast──> All users
  User joins ──[presence]─────────> Server ──broadcast──> All users

Single connection handles all message types.
Sub-100ms latency achievable.
```

### Live Notifications

```
Without WebSocket (polling):
  Client: "Any notifications?" (every 5 seconds)
  Server: "No"
  Client: "Any notifications?"
  Server: "No"
  ... 50 requests later ...
  Server: "Yes, you have a message!"

With WebSocket:
  Server: "New message!" (instant when it happens)

Benefits:
  - Immediate delivery
  - No wasted requests
  - Lower server load
```

### Collaborative Editing

```
Google Docs / Figma style:

User A types ──> Server ──> User B (sees cursor, text)
User B draws ──> Server ──> User A (sees drawing)

Requirements:
  - Low latency (feels responsive)
  - High frequency (keystroke level)
  - Bidirectional (everyone sees everyone)
  - Reliable (no lost changes)

WebSocket + Operational Transform/CRDT
```

### Online Gaming

```
Game server sending:
  - Player positions (60 fps)
  - Game events
  - Chat messages

Players sending:
  - Input commands
  - Actions
  - Chat

WebSocket provides:
  - Single connection (efficient)
  - Binary messages (compact)
  - Low latency (responsive)

Note: Competitive games may prefer UDP/QUIC for
      lower latency at cost of reliability.
```

### Financial Data

```
Stock ticker:
  [AAPL: 150.25] ──> [AAPL: 150.30] ──> [AAPL: 150.28]

Dozens of updates per second.
HTTP request-response unsuitable.
Server-Sent Events work but one-way only.
WebSocket ideal for bidirectional (quotes + orders).
```

### IoT Dashboard

```
Sensors ──> Server ──WebSocket──> Dashboard

Real-time display of:
  - Temperature
  - Humidity
  - Motion
  - System status

Dashboard can also send commands back:
  Dashboard ──> Server ──> Device (turn on/off)
```

## When NOT to Use WebSockets

### Simple CRUD APIs

```
Creating a user:
  POST /users
  {"name": "Alice"}

Response:
  201 Created
  {"id": 123, "name": "Alice"}

One request, one response, done.
WebSocket is overkill—use REST/HTTP.
```

### Infrequent Updates

```
Weather data (updates hourly):
  - Polling once per hour is fine
  - SSE (Server-Sent Events) works well
  - WebSocket connection overhead not justified

Rule of thumb:
  Updates > 1/minute: Consider WebSocket
  Updates < 1/minute: HTTP polling or SSE
```

### Public APIs

```
Public REST API considerations:
  - Stateless (easy to scale)
  - Cacheable
  - Standard tools (curl, Postman)
  - Rate limiting straightforward

WebSocket:
  - Stateful (harder to scale)
  - Not cacheable
  - Fewer debugging tools
  - Rate limiting complex
```

## Alternatives Comparison

```
┌────────────────────────────────────────────────────────────────────┐
│ Technique          │ Direction    │ Latency │ Best For             │
├────────────────────┼──────────────┼─────────┼──────────────────────┤
│ HTTP Polling       │ Client→Svr   │ High    │ Simple, infrequent   │
│ Long Polling       │ Client→Svr   │ Medium  │ Moderate updates     │
│ Server-Sent Events │ Server→Clt   │ Low     │ One-way streaming    │
│ WebSocket          │ Bidirectional│ Low     │ Real-time, two-way   │
│ WebRTC             │ P2P          │ Lowest  │ Audio/video, gaming  │
└────────────────────────────────────────────────────────────────────┘
```

### Server-Sent Events (SSE)

```
Good for:
  - Live feeds (news, sports)
  - Notifications (server→client only)
  - Simpler than WebSocket

Limitations:
  - One-way (server to client)
  - Text only (no binary)
  - Fewer connections per browser

// Server
res.setHeader('Content-Type', 'text/event-stream');
res.write('data: Hello\n\n');

// Client
const source = new EventSource('/stream');
source.onmessage = (e) => console.log(e.data);
```

## Scaling WebSockets

### Connection Limits

```
Challenge:
  10,000 concurrent users = 10,000 open connections
  Each connection uses memory and file descriptors

Solutions:
  - Horizontal scaling (multiple servers)
  - Connection limits per server
  - Load balancing by connection ID
```

### State Distribution

```
User A connected to Server 1
User B connected to Server 2
User A sends message to User B

Server 1 must route to Server 2!

Solutions:
  - Redis pub/sub
  - Dedicated message queue
  - Sticky sessions (same user → same server)
```

### Architecture Pattern

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
  ┌──────▼─────┐      ┌──────▼─────┐      ┌──────▼─────┐
  │ WS Server 1│      │ WS Server 2│      │ WS Server 3│
  └──────┬─────┘      └──────┬─────┘      └──────┬─────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Redis Pub/Sub  │
                    │  (or similar)   │
                    └─────────────────┘

Messages published to Redis, all servers receive.
```

## Summary

Use WebSockets for:
- Real-time bidirectional communication
- High-frequency updates
- Push from server
- Interactive applications

Consider alternatives for:
- Simple request-response (HTTP)
- One-way server→client (SSE)
- Infrequent updates (polling)
- Peer-to-peer (WebRTC)

WebSocket is powerful but adds complexity. Choose based on actual requirements.
