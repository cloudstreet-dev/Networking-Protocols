# HTTP/3 and QUIC

**HTTP/3** (2022) takes a radical approach: instead of building on TCP, it uses **QUIC**—a new transport protocol running over UDP. This eliminates TCP's head-of-line blocking and enables features impossible with TCP.

## Why Replace TCP?

HTTP/2's remaining problem was TCP itself:

```
TCP Head-of-Line Blocking:

HTTP/2 multiplexes streams:
  Stream 1: [A][B][C]
  Stream 3: [X][Y][Z]

TCP sees single byte stream:
  [A][X][B][Y][C][Z]

TCP packet lost (containing [B]):
  - TCP waits for retransmit
  - ALL data after [B] blocked
  - [Y][C][Z] wait even though they're independent

This defeats HTTP/2's multiplexing benefits on lossy networks.
```

## QUIC: UDP-Based Transport

QUIC (Quick UDP Internet Connections) provides TCP-like reliability over UDP:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Protocol Comparison                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  HTTP/1.1, HTTP/2:              HTTP/3:                             │
│  ┌─────────────┐               ┌─────────────┐                      │
│  │    HTTP     │               │   HTTP/3    │                      │
│  ├─────────────┤               ├─────────────┤                      │
│  │    TLS      │               │    QUIC     │  ← Includes TLS!    │
│  ├─────────────┤               ├─────────────┤                      │
│  │    TCP      │               │    UDP      │                      │
│  ├─────────────┤               ├─────────────┤                      │
│  │     IP      │               │     IP      │                      │
│  └─────────────┘               └─────────────┘                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

QUIC provides:
  - Reliable delivery (like TCP)
  - Congestion control (like TCP)
  - Encryption (built-in TLS 1.3)
  - Stream multiplexing (independent streams!)
  - Connection migration
  - 0-RTT connection resumption
```

## No Head-of-Line Blocking

QUIC streams are independent:

```
QUIC stream multiplexing:

Stream 1: [A]──[B]──[C]
Stream 3: [X]──[Y]──[Z]

UDP packets:
  Packet 1: [A][X]
  Packet 2: [B][Y]  ← Lost!
  Packet 3: [C][Z]

What happens:
  Packet 3 arrives, QUIC delivers:
    Stream 3: [Z] delivered immediately!
  Packet 2 retransmitted, then:
    Stream 1: [B] delivered
    Stream 3: [Y] delivered

Stream 3 doesn't wait for Stream 1's retransmit!
Each stream has independent delivery.
```

## Faster Connection Establishment

### TCP+TLS: 2-3 Round Trips

```
TCP + TLS 1.3 connection:

Client                                    Server
   │                                         │
   │────── TCP SYN ──────────────────────────>│
   │<───── TCP SYN-ACK ───────────────────────│
   │────── TCP ACK ──────────────────────────>│  ← 1 RTT (TCP)
   │                                         │
   │────── TLS ClientHello ──────────────────>│
   │<───── TLS ServerHello + Finished ────────│
   │────── TLS Finished + HTTP Request ──────>│  ← 1 RTT (TLS)
   │<───── HTTP Response ─────────────────────│
   │                                         │

Total: 2 RTT before first HTTP response
       (3 RTT with TLS 1.2)
```

### QUIC: 1 Round Trip (or 0!)

```
QUIC initial connection:

Client                                    Server
   │                                         │
   │────── QUIC Initial + TLS Hello ─────────>│
   │<───── QUIC Initial + TLS + ACK ──────────│
   │────── QUIC + HTTP Request ──────────────>│  ← 1 RTT total!
   │<───── HTTP Response ─────────────────────│
   │                                         │

QUIC combines transport + crypto handshake!
TLS 1.3 is integrated into QUIC.
```

### 0-RTT Connection Resumption

```
Returning to a previously visited server:

Client                                    Server
   │                                         │
   │────── QUIC 0-RTT + HTTP Request ────────>│  ← No handshake!
   │<───── HTTP Response ─────────────────────│
   │                                         │

How it works:
  - Client cached server's "resumption token"
  - Client sends encrypted request immediately
  - Server validates token, responds immediately

Caveat: 0-RTT data is replayable
  - Attackers can replay the request
  - Safe only for idempotent requests (GET)
  - Server can implement replay protection
```

## Connection Migration

QUIC connections can survive network changes:

```
Traditional TCP:
  Connection = (Source IP, Source Port, Dest IP, Dest Port)

  Phone switches WiFi → Cellular:
    IP address changes!
    TCP connection breaks
    Must establish new connection
    HTTP request fails/retries

QUIC:
  Connection = Connection ID (random identifier)

  Phone switches WiFi → Cellular:
    IP address changes
    Connection ID unchanged
    QUIC connection continues!
    HTTP request completes seamlessly
```

### Connection Migration Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Connection Migration                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client on WiFi:  192.168.1.100                                     │
│  Server:          93.184.216.34                                     │
│  Connection ID:   0xABCD1234                                        │
│                                                                     │
│  Client ────[QUIC 0xABCD1234]──────> Server                         │
│  Server <───[QUIC 0xABCD1234]─────── Client                         │
│                                                                     │
│  --- Client moves to cellular: 10.0.0.50 ---                        │
│                                                                     │
│  Client ────[QUIC 0xABCD1234]──────> Server                         │
│         ↑                             │                             │
│         │ New IP, same connection ID! │                             │
│         │                             ▼                             │
│  Server validates connection ID,                                    │
│  continues same connection!                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## QUIC Encryption

All QUIC packets are encrypted (except initial handshake):

```
TCP + TLS:
  TCP header:    Visible (port, seq, etc.)
  TLS record:    Encrypted
  HTTP data:     Encrypted

  Middleboxes can see TCP headers, manipulate connections

QUIC:
  UDP header:    Visible (minimal: ports only)
  QUIC header:   Partially encrypted
  QUIC payload:  Fully encrypted

  Middleboxes see only UDP ports
  Cannot inspect or manipulate QUIC layer
```

### Benefits of Always-On Encryption

```
1. Privacy: HTTP/3 headers/content always encrypted
2. Security: Harder to inject/modify traffic
3. Ossification prevention: Middleboxes can't break QUIC
4. Future-proofing: Protocol can evolve without breaking
```

## HTTP/3 Frames

HTTP/3 uses frames similar to HTTP/2, but over QUIC streams:

```
HTTP/3 Frame Types:
  DATA          - Request/response body
  HEADERS       - Headers (QPACK compressed)
  CANCEL_PUSH   - Cancel server push
  SETTINGS      - Connection settings
  PUSH_PROMISE  - Server push notification
  GOAWAY        - Connection shutdown
  MAX_PUSH_ID   - Limit on push streams

Key difference from HTTP/2:
  - Each request/response on separate QUIC stream
  - No stream multiplexing in HTTP/3 (QUIC handles it)
  - Flow control handled by QUIC layer
```

## QPACK Header Compression

HTTP/3 uses QPACK (QUIC-aware HPACK variant):

```
HPACK problem with QUIC:
  HPACK uses dynamic table updated per header
  Headers arrive out of order in QUIC
  Can't update table until all prior updates processed
  → Head-of-line blocking in header compression!

QPACK solution:
  - Separate unidirectional streams for table updates
  - Encoder/decoder can choose blocking behavior
  - Trades compression ratio for lower latency

Result: Slightly less compression than HPACK,
        but no header compression blocking
```

## HTTP/3 Adoption

```
As of 2024:
  - ~25% of websites support HTTP/3
  - All major browsers support HTTP/3
  - Major CDNs (Cloudflare, Akamai, Fastly) support HTTP/3
  - Google, Facebook, and others use HTTP/3 heavily

Server support:
  - nginx: Experimental
  - Cloudflare: Full support
  - LiteSpeed: Full support
  - Caddy: Full support
  - IIS: Not yet

Client support:
  - Chrome: Yes
  - Firefox: Yes
  - Safari: Yes
  - Edge: Yes
  - curl: Yes (with HTTP/3 build)
```

## Deploying HTTP/3

### Server Configuration

```nginx
# nginx (experimental)
server {
    listen 443 quic reuseport;
    listen 443 ssl;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Advertise HTTP/3 support
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

### Alt-Svc Header

Browsers discover HTTP/3 via Alt-Svc:

```http
HTTP/2 200 OK
alt-svc: h3=":443"; ma=86400

Meaning:
  h3=":443"  - HTTP/3 available on port 443
  ma=86400   - Cache this for 24 hours

Browser flow:
  1. Connect via HTTP/2 (known to work)
  2. See Alt-Svc header
  3. Try HTTP/3 for subsequent requests
  4. Fall back to HTTP/2 if QUIC blocked
```

## When HTTP/3 Helps Most

```
Significant improvement:
  - High latency connections (satellite, intercontinental)
  - Lossy networks (mobile, WiFi congestion)
  - Many small resources (API calls)
  - Users switching networks (mobile)

Moderate improvement:
  - Low latency, reliable networks
  - Large single downloads

May not help:
  - Local/datacenter communication
  - UDP blocked (corporate firewalls)
```

## Debugging HTTP/3

```bash
# curl with HTTP/3
$ curl --http3 https://example.com -v
* using HTTP/3
* h3 [:method: GET]
* h3 [:path: /]
...

# Check if site supports HTTP/3
$ curl -sI https://example.com | grep -i alt-svc

# Chrome DevTools
  Network tab → Protocol shows "h3"

# qlog for QUIC debugging
  Standardized logging format for QUIC
  Visualize with qvis (https://qvis.quictools.info/)
```

## Summary

HTTP/3 over QUIC provides:

| Feature | Benefit |
|---------|---------|
| UDP-based | Avoids TCP ossification |
| Independent streams | No transport HOL blocking |
| 0-RTT resumption | Instant subsequent connections |
| Connection migration | Survives network changes |
| Built-in encryption | Always secure, anti-ossification |
| QPACK compression | Efficient headers without blocking |

Trade-offs:
- UDP may be blocked by firewalls
- More CPU for encryption/decryption
- Newer, less mature implementations
- Debugging tools still evolving

HTTP/3 represents the cutting edge of web protocols. For deep dives into QUIC itself, see the next chapter.
