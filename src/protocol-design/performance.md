# Performance Considerations

Protocol design choices significantly impact performance. Understanding the trade-offs helps you make informed decisions.

## Key Performance Factors

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Performance Dimensions                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Latency:     Time for message round-trip                           │
│               Affected by: RTTs, encoding time, processing          │
│                                                                     │
│  Throughput:  Data volume per unit time                             │
│               Affected by: Message size, connection limits          │
│                                                                     │
│  Overhead:    Wasted bandwidth (headers, framing)                   │
│               Affected by: Protocol verbosity, encoding             │
│                                                                     │
│  Efficiency:  CPU/memory per message                                │
│               Affected by: Parsing, serialization                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Message Format Trade-offs

### Text vs Binary

```
Text (JSON, XML):
  + Human readable
  + Easy debugging
  + Universal parsers
  - Larger messages
  - Slower parsing
  - Ambiguous types

Binary (Protocol Buffers, MessagePack):
  + Compact messages
  + Fast parsing
  + Precise types
  - Requires schema/decoder
  - Harder debugging
  - Versioning complexity

Rule of thumb:
  Internal services: Binary (efficiency)
  Public APIs: JSON (interoperability)
  High-volume: Binary (worth complexity)
```

### Size Comparison

```
Same data in different formats:

JSON (70 bytes):
  {"id":123,"name":"Alice","email":"alice@example.com"}

Protocol Buffers (35 bytes):
  [binary encoded, ~50% smaller]

MessagePack (45 bytes):
  [binary JSON, ~35% smaller]

For millions of messages, these differences matter!
```

## Connection Strategies

### Persistent vs Per-Request

```
Per-request connections:
  Each request: TCP handshake + TLS handshake + request
  Latency: High (multiple RTTs)
  Resources: Connection churn

Persistent connections:
  One connection: Multiple requests
  Latency: Low (no repeated handshakes)
  Resources: Connection management

Always prefer persistent for repeated interactions.
```

### Multiplexing

```
HTTP/1.1 (no multiplexing):
  Connection 1: Request A ─────> Response A
  Connection 2: Request B ─────> Response B
  (Need multiple connections for parallelism)

HTTP/2 (multiplexing):
  Connection 1: [A][B][C]───>[A][B][C]
  (All requests on one connection)

Multiplexing reduces:
  - Connection overhead
  - Memory usage
  - Head-of-line blocking (with QUIC)
```

## Batching and Pipelining

```
Individual requests:
  Request 1 → Response 1 → Request 2 → Response 2
  Time: 4 RTT for 2 requests

Pipelining:
  Request 1 → Request 2 → Response 1 → Response 2
  Time: 2 RTT for 2 requests

Batching:
  [Request 1, Request 2] → [Response 1, Response 2]
  Time: 1 RTT for 2 requests

Trade-off: Batching adds latency for first item.
```

## Compression

```
When to compress:
  ✓ Large text payloads (JSON, HTML)
  ✓ Repeated patterns in data
  ✓ Slow/metered networks

When not to compress:
  ✗ Already compressed (images, video)
  ✗ Tiny messages (overhead > savings)
  ✗ CPU-constrained environments

Common algorithms:
  gzip:   Universal, good compression
  br:     Better ratio, slower
  zstd:   Fast, good ratio (emerging)
```

## Caching

```
Cacheable responses reduce load:

Without caching:
  Every request → Server processing → Response

With caching:
  First request → Server → Response (cached)
  Repeat requests → Cache hit → Immediate response

Design for cacheability:
  - Stable URLs for same content
  - Proper cache headers
  - ETags for validation
  - Separate static/dynamic content
```

## Measurement

```
Measure before optimizing:

Latency metrics:
  - P50, P95, P99 response times
  - Time to first byte (TTFB)
  - Round-trip time

Throughput metrics:
  - Requests per second
  - Bytes per second
  - Messages per connection

Tools:
  - wrk, ab (HTTP benchmarking)
  - tcpdump, wireshark (packet analysis)
  - perf, flamegraphs (CPU profiling)
```

## Summary

Performance optimization priorities:

1. **Reduce round trips** (biggest impact)
2. **Use persistent connections**
3. **Choose appropriate message format**
4. **Enable compression for large text**
5. **Implement caching where possible**
6. **Batch when latency allows**

Measure, then optimize. Premature optimization is the root of all evil, but informed optimization is essential.
