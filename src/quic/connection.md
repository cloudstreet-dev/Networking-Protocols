# Connection Establishment and 0-RTT

QUIC's handshake integrates transport and cryptographic establishment, dramatically reducing connection latency.

## 1-RTT Handshake

A new QUIC connection to a server:

```
Client                                            Server
   │                                                 │
   │─── Initial[TLS ClientHello, CRYPTO] ───────────>│
   │                                                 │
   │    (1 RTT passes)                               │
   │                                                 │
   │<── Initial[TLS ServerHello, CRYPTO] ────────────│
   │<── Handshake[TLS EncryptedExtensions] ──────────│
   │<── Handshake[TLS Certificate] ──────────────────│
   │<── Handshake[TLS CertVerify, Finished] ─────────│
   │                                                 │
   │─── Handshake[TLS Finished] ────────────────────>│
   │                                                 │
   │    === Connection Established ===               │
   │                                                 │
   │─── Application Data ───────────────────────────>│
   │<── Application Data ────────────────────────────│
```

### Packet Types During Handshake

```
Initial Packets:
  - First packets sent
  - Protected with Initial Keys (derived from DCID)
  - Contains CRYPTO frames with TLS messages
  - Minimum 1200 bytes (amplification protection)

Handshake Packets:
  - Sent after Initial exchange
  - Protected with Handshake Keys
  - Complete the TLS 1.3 handshake

1-RTT Packets:
  - After handshake completes
  - Protected with Application Keys
  - Used for all application data
```

## 0-RTT Resumption

If client has previously connected, it can send data immediately:

```
First connection:
  - Client receives "session ticket" from server
  - Contains resumption secret and server config
  - Cached for future use

Subsequent connection:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Client                                            Server           │
│     │                                                 │             │
│     │─── Initial[TLS ClientHello] ───────────────────>│             │
│     │─── 0-RTT[Application Data] ────────────────────>│ ← Data      │
│     │                                                 │   sent      │
│     │<── Initial[TLS ServerHello] ────────────────────│   before    │
│     │<── Handshake[TLS Finished] ─────────────────────│   handshake │
│     │<── 1-RTT[Application Data Response] ────────────│   completes!│
│     │                                                 │             │
└─────────────────────────────────────────────────────────────────────┘

Client sends request BEFORE receiving server's response!
```

### 0-RTT Security Considerations

```
0-RTT data can be replayed:

Attacker captures:
  [Initial + 0-RTT packets]

Attacker replays:
  [Initial + 0-RTT packets] → Server processes request again!

Safe for 0-RTT:
  ✓ GET requests (idempotent)
  ✓ Read-only operations
  ✓ Operations with other replay protection

NOT safe for 0-RTT:
  ✗ POST/PUT (non-idempotent)
  ✗ Financial transactions
  ✗ Anything with side effects

Server controls:
  - Can reject 0-RTT entirely
  - Can accept but limit to safe operations
  - Can implement replay detection (within limits)
```

## Connection IDs

QUIC connections are identified by Connection IDs, not IP/port tuples:

```
Connection ID structure:
  - Variable length (0-20 bytes)
  - Chosen by each endpoint
  - Destination CID: What I put in packets TO you
  - Source CID: What you put in packets TO ME

Initial exchange:
  Client → Server: Dest CID = random, Source CID = client's CID
  Server → Client: Dest CID = client's CID, Source CID = server's CID

After handshake:
  Both sides agree on CIDs to use
  Server typically provides multiple CIDs for migration
```

### Connection ID Benefits

```
1. NAT Rebinding Tolerance
   NAT timeout changes source port
   CID unchanged → Connection continues

2. Load Balancer Routing
   CID can encode server selection
   Any frontend can route to correct backend

3. Privacy (with rotation)
   CID can be changed periodically
   Harder to track connections across time
```

## Amplification Attack Protection

QUIC prevents DDoS amplification:

```
Attack scenario without protection:
  Attacker: Sends 50-byte Initial with spoofed source IP
  Server: Responds with 10,000 bytes to victim
  Amplification factor: 200x

QUIC protection:
  Before address validation:
    Server can send ≤ 3× what client sent

  Client Initial minimum: 1200 bytes
  Server can send: ≤ 3600 bytes

  For more, server requires address validation:
    - Send Retry packet (stateless)
    - Or use address validation token
```

### Retry Flow

```
Client                                            Server
   │                                                 │
   │─── Initial (1200 bytes) ──────────────────────>│
   │                                                 │
   │<── Retry[token, new SCID] ─────────────────────│
   │                                                 │
   │─── Initial[token] ────────────────────────────>│
   │                                                 │
   │    (server validates token, proceeds normally)  │
```

## Connection Termination

### Graceful Close

```
Endpoint sends CONNECTION_CLOSE frame:
  - Error code: NO_ERROR (0x0) for clean close
  - Reason phrase (optional)

Both sides:
  - Stop sending new data
  - Send acknowledgments for received data
  - Enter closing period (3× PTO)
  - Then fully close
```

### Stateless Reset

For when connection state is lost:

```
Server crashes and restarts:
  - Lost all connection state
  - Client sends packets server doesn't recognize

Server sends Stateless Reset:
  - Looks like a regular packet
  - Contains reset token (derived from CID)
  - Client recognizes token, closes connection

Prevents hanging connections after server restart.
```

## Summary

QUIC connection establishment:

| Scenario | Round Trips | Data Delay |
|----------|-------------|------------|
| TCP + TLS 1.2 | 3 RTT | 3 RTT |
| TCP + TLS 1.3 | 2 RTT | 2 RTT |
| QUIC new | 1 RTT | 1 RTT |
| QUIC 0-RTT | 0 RTT | 0 RTT |

Key mechanisms:
- **Integrated handshake**: Transport + crypto combined
- **Connection IDs**: Enable migration, load balancing
- **0-RTT**: Instant resumption (with replay caveats)
- **Amplification protection**: Prevents DDoS abuse
