# Connection Migration

QUIC connections can survive network changes—a game-changer for mobile users who constantly switch between WiFi and cellular networks.

## The Problem with TCP

TCP connections are bound to IP addresses:

```
TCP connection tuple:
  (192.168.1.100, 52000, 93.184.216.34, 443)
    └── Client IP ──┘

When IP changes (WiFi → cellular):
  New tuple: (10.0.0.50, 48000, 93.184.216.34, 443)

Server: "Who are you? I don't have a connection from 10.0.0.50"
Connection dies. Application must reconnect.

Impact:
  - HTTP request fails
  - Download interrupted
  - Streaming buffers
  - User experience degraded
```

## QUIC's Solution: Connection IDs

QUIC identifies connections by Connection ID, not IP:

```
QUIC connection:
  Connection ID: 0x1A2B3C4D5E6F

Packets from 192.168.1.100 with CID 0x1A2B3C4D5E6F
Packets from 10.0.0.50 with CID 0x1A2B3C4D5E6F

Server: "Same CID? Same connection! Continue."
```

### Migration Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Connection Migration                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Time 0: Client on WiFi (192.168.1.100)                             │
│    Client ──[CID: ABC]──> Server                                    │
│    Server <──[CID: XYZ]── Client                                    │
│                                                                     │
│  Time 1: Client switches to cellular (10.0.0.50)                    │
│    Client ──[CID: ABC]──> Server (from new IP!)                     │
│                                                                     │
│  Time 2: Server validates new path                                  │
│    Server ──[PATH_CHALLENGE]──> Client                              │
│    Client ──[PATH_RESPONSE]──> Server                               │
│                                                                     │
│  Time 3: Migration complete                                         │
│    Connection continues seamlessly                                  │
│    In-flight data retransmitted if needed                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Path Validation

Migration requires validating the new path:

```
Why validate?
  - Prove client owns new address (anti-spoofing)
  - Verify path works for bidirectional traffic
  - Update RTT estimates for new path

PATH_CHALLENGE:
  Server sends random 8-byte challenge to new address

PATH_RESPONSE:
  Client echoes the 8 bytes back

If response matches: Path validated, migration complete
If no response: Revert to previous path
```

## Connection ID Management

Multiple Connection IDs enable smooth migration:

```
Server provides multiple CIDs:

NEW_CONNECTION_ID frames:
  CID 1: 0xAAAAAA, Sequence 0
  CID 2: 0xBBBBBB, Sequence 1
  CID 3: 0xCCCCCC, Sequence 2

Client can use any of these CIDs.

On migration:
  Client switches to unused CID
  Server correlates new CID to connection
  Old CID retired for privacy

RETIRE_CONNECTION_ID:
  "I'm done using CID sequence 0"
```

### Privacy Benefits

```
Without CID rotation:
  Observer: "CID 0xABC on WiFi... same CID on cellular"
  Observer: "This is the same user, tracked!"

With CID rotation:
  WiFi: Uses CID 0xAAA
  Cellular: Uses CID 0xBBB (unused before)

  Observer: "Different CIDs, can't correlate"
  (Connection continues, but linkability reduced)
```

## Probing and Preferred Paths

QUIC can probe multiple paths:

```
Client has:
  - WiFi connection (reliable, maybe slow)
  - Cellular connection (less reliable, maybe faster)

Client can:
  1. Probe both paths with PATH_CHALLENGE
  2. Measure RTT and loss on each
  3. Choose preferred path
  4. Keep other path as backup

This enables:
  - Seamless handoff
  - Make-before-break migration
  - Multipath in future extensions
```

## NAT Rebinding

Even without physical network change, NAT can disrupt:

```
NAT timeout scenario:
  Connection idle for 30 minutes
  NAT forgets the mapping
  NAT assigns new external port

TCP: Connection times out or RST

QUIC:
  CID unchanged
  Server validates new path
  Connection continues
```

## Server-Side Considerations

### Load Balancer Routing

```
With TCP:
  Load balancer routes by 4-tuple
  IP change → Different backend → Connection state lost

With QUIC:
  Load balancer can route by CID
  Server encodes routing info in CID

CID format (example):
  [Server ID: 4 bytes][Random: 8 bytes]

Any frontend extracts Server ID from CID
Routes to correct backend regardless of client IP
```

### Connection State

```
Server must maintain:
  - Connection state by CID (not by IP)
  - Multiple CIDs per connection
  - Token→Connection mapping for 0-RTT

Storage indexed by CID, not IP address.
```

## Mobile Experience Impact

```
Real-world scenarios improved:

1. Elevator/Subway
   TCP: Connection dies, app reconnects
   QUIC: Brief pause, then continues

2. Walking between access points
   TCP: Each AP change = potential reset
   QUIC: Seamless, user unaware

3. VPN connect/disconnect
   TCP: All connections reset
   QUIC: Continues through VPN changes

4. NAT timeout during idle
   TCP: Silent failure on next request
   QUIC: Automatic path revalidation
```

## Summary

Connection migration enables:

| Feature | User Benefit |
|---------|--------------|
| IP address change survival | Seamless WiFi/cellular handoff |
| CID-based identification | Load balancer flexibility |
| Path validation | Security against spoofing |
| CID rotation | Privacy from observers |
| NAT rebinding tolerance | Fewer "connection reset" errors |

Migration is one of QUIC's most user-visible improvements, particularly for mobile users who previously experienced constant interruptions during network transitions.
