# HTTP/2: Multiplexing Revolution

**HTTP/2** (2015) reimagined how HTTP works at the wire level while maintaining full compatibility with HTTP/1.1 semantics. The result: dramatically faster page loads with no application changes required.

## The Core Innovation: Multiplexing

HTTP/2's killer feature is **multiplexing**—sending multiple requests and responses over a single TCP connection simultaneously:

```
HTTP/1.1 (head-of-line blocking):
┌─────────────────────────────────────────────────────────────────────────┐
│  Connection 1: [──────Req 1──────][──────Req 2──────][──Req 3──]       │
│  Connection 2: [──────Req 4──────][──Req 5──]                          │
│  Connection 3: [──Req 6──][──────Req 7──────]                          │
│                                                                         │
│  Sequential on each connection. Multiple connections needed.            │
└─────────────────────────────────────────────────────────────────────────┘

HTTP/2 (multiplexed):
┌─────────────────────────────────────────────────────────────────────────┐
│  Single connection:                                                     │
│    [R1][R2][R3][R1][R4][R2][R5][R3][R1][R6][R7]...                      │
│                                                                         │
│  All requests interleaved on one connection!                            │
│  No head-of-line blocking at HTTP level.                                │
└─────────────────────────────────────────────────────────────────────────┘
```

## Binary Framing Layer

HTTP/2 replaces text with binary frames:

```
HTTP/1.1 (text):
┌────────────────────────────────────────┐
│ GET /page HTTP/1.1\r\n                 │
│ Host: example.com\r\n                  │
│ Accept: text/html\r\n                  │
│ \r\n                                   │
└────────────────────────────────────────┘

HTTP/2 (binary frames):
┌────────────────────────────────────────┐
│ ┌─────────────┐ ┌─────────────┐        │
│ │HEADERS Frame│ │ DATA Frame  │        │
│ │ Stream ID: 1│ │ Stream ID: 1│        │
│ │ (compressed)│ │ (payload)   │        │
│ └─────────────┘ └─────────────┘        │
└────────────────────────────────────────┘

Binary format:
  + Efficient parsing (no text scanning)
  + Compact representation
  + Clear frame boundaries
  - Not human-readable (need tools)
```

## Frame Structure

Every HTTP/2 message is a series of frames:

```
Frame Format:
┌────────────────────────────────────────────────────────────────────┐
│ Length (24 bits) │ Type (8) │ Flags (8) │ R │ Stream ID (31 bits) │
├────────────────────────────────────────────────────────────────────┤
│                        Frame Payload                               │
└────────────────────────────────────────────────────────────────────┘

Length:    Size of payload (max 16KB default, configurable)
Type:      DATA, HEADERS, PRIORITY, RST_STREAM, SETTINGS, etc.
Flags:     Type-specific flags (END_STREAM, END_HEADERS, etc.)
Stream ID: Which stream this frame belongs to
```

### Frame Types

```
┌──────────────────────────────────────────────────────────────────┐
│  Type        │ Purpose                                           │
├──────────────┼───────────────────────────────────────────────────┤
│  DATA        │ Request/response body data                        │
│  HEADERS     │ Request/response headers (compressed)             │
│  PRIORITY    │ Stream priority information                       │
│  RST_STREAM  │ Terminate a stream                                │
│  SETTINGS    │ Connection configuration                          │
│  PUSH_PROMISE│ Server push notification                          │
│  PING        │ Connection health check                           │
│  GOAWAY      │ Graceful connection shutdown                      │
│  WINDOW_UPDATE│ Flow control window adjustment                   │
│  CONTINUATION│ Continuation of HEADERS                           │
└──────────────┴───────────────────────────────────────────────────┘
```

## Streams

A **stream** is a bidirectional sequence of frames within a connection:

```
Single HTTP/2 connection with multiple streams:

┌─────────────────────────────────────────────────────────────────────┐
│                         TCP Connection                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Stream 1: [HEADERS]──[DATA]──[DATA]──[DATA]                 │   │
│  │ Stream 3: [HEADERS]──[DATA]                                 │   │
│  │ Stream 5: [HEADERS]                                         │   │
│  │ Stream 7: [HEADERS]──[DATA]──[DATA]                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

Stream IDs:
  - Odd numbers: Client-initiated
  - Even numbers: Server-initiated (push)
  - 0: Connection-level messages (SETTINGS, PING, GOAWAY)
```

### Request/Response as Streams

```
HTTP/2 Request (Stream 1):
┌────────────────────────────────────────────────────────────────────┐
│  HEADERS Frame (Stream 1)                                          │
│    :method = GET                                                   │
│    :path = /index.html                                             │
│    :scheme = https                                                 │
│    :authority = example.com                                        │
│    accept = text/html                                              │
│    END_HEADERS, END_STREAM flags                                   │
└────────────────────────────────────────────────────────────────────┘

HTTP/2 Response (Stream 1):
┌────────────────────────────────────────────────────────────────────┐
│  HEADERS Frame (Stream 1)                                          │
│    :status = 200                                                   │
│    content-type = text/html                                        │
│    END_HEADERS flag                                                │
├────────────────────────────────────────────────────────────────────┤
│  DATA Frame (Stream 1)                                             │
│    [HTML content...]                                               │
│    END_STREAM flag                                                 │
└────────────────────────────────────────────────────────────────────┘
```

## Header Compression (HPACK)

HTTP/2 compresses headers using HPACK:

```
HTTP/1.1 headers (sent every request):
  Host: example.com                    ~17 bytes
  User-Agent: Mozilla/5.0...           ~70 bytes
  Accept: text/html,application/...    ~100 bytes
  Accept-Language: en-US,en;q=0.5      ~25 bytes
  Accept-Encoding: gzip, deflate       ~25 bytes
  Cookie: session=abc123;...           ~50+ bytes
  ──────────────────────────────────────────────
  Total: ~300 bytes per request!

  10 requests = 3KB just in headers!

HPACK compression:
  1. Static table: 61 common headers (predefined)
  2. Dynamic table: Recently used headers (learned)
  3. Huffman coding: Compress literal values

  First request: ~300 bytes → ~150 bytes (Huffman)
  Second request: Same headers → ~30 bytes (indexed!)

  10 requests ≈ 300 bytes total (vs 3KB)
```

### HPACK Example

```
First request headers sent:
  :method: GET           → Index 2 (static table)
  :path: /index.html     → Literal, indexed
  :authority: example.com → Literal, indexed, Huffman
  accept: text/html      → Literal, indexed

Dynamic table after request:
  [62] :path: /index.html
  [63] :authority: example.com
  [64] accept: text/html

Second request to same server:
  :method: GET           → Index 2 (static)
  :path: /style.css      → Literal (new path)
  :authority: example.com → Index 63 (dynamic!)
  accept: text/css       → Literal (different)

Much smaller because authority is now indexed!
```

## Server Push

Servers can proactively send resources:

```
Without server push:
  Client: GET /index.html
  Server: (sends HTML)
  Client: (parses, sees style.css link)
  Client: GET /style.css       ← Extra round trip!
  Server: (sends CSS)

With server push:
  Client: GET /index.html
  Server: PUSH_PROMISE /style.css    ← "I'll send this too"
  Server: (sends HTML)
  Server: (sends CSS on separate stream)
  Client: (already has CSS when parsing HTML!)

Saves round trip for critical resources.
```

### Server Push Caveats

```
Push sounds great but has issues:

1. May push already-cached resources
   Server doesn't know client cache state
   Waste bandwidth pushing what client has

2. Priority problems
   Pushed resources may compete with requested ones
   Can slow down critical content

3. Limited browser support
   Chrome deprecated push support (2022)
   Most CDNs recommend disabling

Alternative: 103 Early Hints
  Server sends hints before full response
  Client can preload without full push complexity
```

## Stream Prioritization

Clients can indicate resource importance:

```
Priority information:
  - Weight: 1-256 (relative importance)
  - Dependency: Stream this depends on

Example:
  Stream 1 (HTML):      Weight=256 (highest)
  Stream 3 (CSS):       Weight=128, depends on Stream 1
  Stream 5 (JS):        Weight=128, depends on Stream 1
  Stream 7 (image):     Weight=64, depends on Stream 3

Priority tree:
          [Stream 1 - HTML]
              /       \
    [Stream 3-CSS]  [Stream 5-JS]
          |
    [Stream 7-image]

Server should:
  1. Complete Stream 1 first
  2. Then CSS and JS equally
  3. Images last

Reality: Server implementation varies
         Many servers ignore priorities
```

## Flow Control

HTTP/2 has stream-level flow control:

```
Connection flow control:
  Each side advertises receive window
  Similar to TCP flow control

Stream flow control:
  Each stream has its own window
  Prevents one stream from consuming all bandwidth

WINDOW_UPDATE frame:
  Signals capacity for more data

┌────────────────────────────────────────────────────────────────────┐
│  Stream 1: Window=65535                                            │
│  Stream 3: Window=65535                                            │
│  Connection: Window=1048576                                        │
│                                                                    │
│  Server sends 32768 bytes on Stream 1:                             │
│    Stream 1: Window=32767                                          │
│    Connection: Window=1015808                                      │
│                                                                    │
│  Client sends WINDOW_UPDATE (Stream 1, 32768):                     │
│    Stream 1: Window=65535 (restored)                               │
└────────────────────────────────────────────────────────────────────┘
```

## HTTP/2 Connection Setup

HTTP/2 requires TLS in practice (browsers require HTTPS):

```
1. TCP handshake
2. TLS handshake (ALPN negotiates HTTP/2)
3. HTTP/2 connection preface:
   Client sends: "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"
   Both send: SETTINGS frame

4. Ready for requests!

ALPN (Application-Layer Protocol Negotiation):
  Client TLS hello includes: "I support h2, http/1.1"
  Server chooses: "Let's use h2"
  Connection established as HTTP/2
```

## The Remaining Problem: TCP HOL Blocking

HTTP/2 solved HTTP-level head-of-line blocking but TCP has its own:

```
HTTP/2 over TCP problem:

Stream 1: [Frame 1][Frame 2][Frame 3]
Stream 3: [Frame A][Frame B]
Stream 5: [Frame X][Frame Y]

TCP sees: [1][A][2][X][B][3][Y]

If TCP packet containing [2] is lost:
  TCP retransmits [2]
  ALL subsequent data waits!
  Frames [X][B][3][Y] all blocked

Even though X,B,Y are independent streams!

This is TCP-level head-of-line blocking.
HTTP/3 solves this with QUIC.
```

## HTTP/2 Performance

```
When HTTP/2 shines:
  - Many small resources (multiplexing wins)
  - High latency connections (fewer round trips)
  - Header-heavy requests (compression helps)
  - HTTPS (required anyway, TLS overhead amortized)

When HTTP/2 helps less:
  - Single large download (one stream anyway)
  - Very low latency networks (overhead matters less)
  - Lossy networks (TCP HOL blocking hurts)

Typical improvements:
  Page load time: 10-50% faster
  Time to first byte: Similar or slightly better
  Number of connections: 1 vs 6+ (simpler)
```

## Debugging HTTP/2

```bash
# curl with HTTP/2
$ curl -I --http2 https://example.com
HTTP/2 200
content-type: text/html

# nghttp client
$ nghttp -nv https://example.com

# Chrome DevTools
  Network tab → Protocol column shows "h2"

# Wireshark
  Filter: http2
  Decode TLS with SSLKEYLOGFILE
```

## Summary

HTTP/2's key innovations:

| Feature | Benefit |
|---------|---------|
| Binary framing | Efficient parsing, clear boundaries |
| Multiplexing | Multiple requests on one connection |
| Header compression | ~85% reduction in header size |
| Stream prioritization | Better resource loading order |
| Server push | Proactive resource delivery |
| Flow control | Per-stream bandwidth management |

Limitations:
- TCP head-of-line blocking remains
- Server push deprecated in browsers
- Complexity increased

HTTP/2 is a significant improvement over HTTP/1.1, but TCP's head-of-line blocking motivated HTTP/3's move to QUIC—which we'll cover next.
