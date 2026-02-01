# Multiplexing Without Head-of-Line Blocking

QUIC's stream multiplexing is its most impactful feature for application performance. Unlike TCP, QUIC streams are truly independent.

## Streams in QUIC

A QUIC connection supports multiple concurrent streams:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        QUIC Connection                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Stream 0 (bidirectional): [HTTP request/response]             │ │
│  │  Stream 4 (bidirectional): [Another request/response]          │ │
│  │  Stream 8 (bidirectional): [Third request/response]            │ │
│  │  Stream 2 (unidirectional): [Control messages →]               │ │
│  │  Stream 6 (unidirectional): [Server push →]                    │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘

Stream types:
  Bidirectional:  Data flows both ways
  Unidirectional: Data flows one way only

Stream IDs:
  0, 4, 8, 12...  Client-initiated bidirectional
  1, 5, 9, 13...  Server-initiated bidirectional
  2, 6, 10, 14... Client-initiated unidirectional
  3, 7, 11, 15... Server-initiated unidirectional
```

## Independence Guarantee

Each stream maintains its own state:

```
Stream states:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Stream 0: offset 0-1000 received, expecting 1001                   │
│  Stream 4: offset 0-500 received, expecting 501                     │
│  Stream 8: offset 0-2000 received, complete                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

If Stream 4 data at offset 501-1000 is lost:
  - Stream 0: Continues receiving, delivering data
  - Stream 4: Waits for retransmit of 501-1000
  - Stream 8: Already complete, unaffected

NO cross-stream blocking!
```

## Comparison with HTTP/2 over TCP

```
HTTP/2 over TCP:

Request 1: [----data----]
Request 2: [--data--]
Request 3: [------data------]

TCP sees: [1][2][3][1][2][3][1][3][1][3]

TCP packet 3 lost (contains Request 2 data):
  All subsequent packets buffered by TCP
  Requests 1 and 3 blocked waiting for Request 2!

─────────────────────────────────────────────────────────────────────

HTTP/3 over QUIC:

Request 1: Stream 0
Request 2: Stream 4
Request 3: Stream 8

QUIC packet with Stream 4 data lost:
  Stream 4 data retransmitted
  Streams 0 and 8 continue independently!

Each HTTP request truly independent.
```

## Stream Flow Control

QUIC has two levels of flow control:

```
1. Stream-level flow control:
   Each stream has its own receive window
   Prevents one stream from consuming all buffer

2. Connection-level flow control:
   Total bytes across all streams
   Prevents connection from overwhelming receiver

┌─────────────────────────────────────────────────────────────────────┐
│  Connection MAX_DATA: 1,000,000 bytes                               │
│  ├── Stream 0 MAX_STREAM_DATA: 100,000 bytes                        │
│  ├── Stream 4 MAX_STREAM_DATA: 100,000 bytes                        │
│  └── Stream 8 MAX_STREAM_DATA: 100,000 bytes                        │
│                                                                     │
│  Stream 0 can use up to 100KB                                       │
│  All streams combined can use up to 1MB                             │
└─────────────────────────────────────────────────────────────────────┘

Flow control frames:
  MAX_DATA: Update connection limit
  MAX_STREAM_DATA: Update stream limit
  DATA_BLOCKED: Signal sender is blocked
  STREAM_DATA_BLOCKED: Signal stream is blocked
```

## Stream Prioritization

Applications can indicate stream importance:

```
Priority information per stream:
  - Urgency: 0-7 (0 highest)
  - Incremental: true/false (can process partially)

Example (HTTP/3):
  HTML:   Stream 0, urgency=0, incremental=false
  CSS:    Stream 4, urgency=1, incremental=false
  JS:     Stream 8, urgency=1, incremental=false
  Images: Stream 12+, urgency=5, incremental=true

Sender should:
  1. Send all urgency=0 data first
  2. Round-robin among same urgency
  3. Incremental streams can be interleaved

Note: Priority is a hint, not enforced by QUIC itself.
```

## Stream Lifecycle

```
                    ┌──────────────────┐
                    │       Idle       │
                    └────────┬─────────┘
                             │ Open (send/receive)
                    ┌────────▼─────────┐
                    │       Open       │
                    │                  │
                    │ Send/Receive data│
                    └──┬──────────┬────┘
                       │          │
         Send FIN ─────┘          └───── Receive FIN
                       │          │
              ┌────────▼──┐    ┌──▼─────────┐
              │Half-Closed│    │Half-Closed │
              │  (local)  │    │  (remote)  │
              └─────┬─────┘    └──────┬─────┘
                    │                 │
    Receive FIN ────┘                 └──── Send FIN
                    │                 │
                    └────────┬────────┘
                    ┌────────▼─────────┐
                    │      Closed      │
                    └──────────────────┘

Stream can also be reset (RST_STREAM) at any point.
```

## Practical Impact

```
Scenario: Page with 50 resources over lossy network (2% loss)

HTTP/2 over TCP:
  Any lost packet blocks ALL pending responses
  On 2% loss: Significant stalls and delays
  Measured: 3-4x slower on lossy mobile

HTTP/3 over QUIC:
  Lost packet only affects its stream
  Other 49 resources continue loading
  Measured: Near-optimal even with loss

Real-world impact is most visible on:
  - Mobile networks (variable quality)
  - Satellite connections (high latency + loss)
  - Congested WiFi
```

## Stream Limits

```
Connections limit maximum streams:

MAX_STREAMS frames:
  - MAX_STREAMS (bidi): Max bidirectional streams
  - MAX_STREAMS (uni): Max unidirectional streams

Typical defaults:
  100 bidirectional streams
  100 unidirectional streams

If limit reached:
  Sender must wait for streams to close
  Or wait for MAX_STREAMS increase
  STREAMS_BLOCKED frame signals waiting
```

## Summary

QUIC stream multiplexing provides:

| Feature | Benefit |
|---------|---------|
| Independent streams | No head-of-line blocking |
| Per-stream flow control | Fair resource allocation |
| Stream priorities | Important content first |
| Unidirectional streams | Efficient one-way data |
| Low overhead | Stream creation is cheap |

This is QUIC's key advantage over TCP for multiplexed protocols. Loss on one stream doesn't impact others, making it ideal for modern web applications with many parallel requests.
