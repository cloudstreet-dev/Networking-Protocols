# TCP States and Lifecycle

A TCP connection progresses through a series of **states** from creation to termination. Understanding these states helps you debug connection issues, interpret netstat output, and understand why connections sometimes linger.

## The State Diagram

```
                            ┌───────────────────────────────────────┐
                            │                CLOSED                  │
                            └───────────────────┬───────────────────┘
                                                │
              ┌─────────────────────────────────┼─────────────────────────────────┐
              │                                 │                                 │
              │ Passive Open                    │ Active Open                     │
              │ (Server: listen())              │ (Client: connect())             │
              ▼                                 ▼                                 │
      ┌───────────────┐                ┌───────────────┐                          │
      │    LISTEN     │                │   SYN_SENT    │                          │
      │               │                │               │                          │
      │ Waiting for   │                │ SYN sent,     │                          │
      │ connection    │                │ waiting for   │                          │
      │ request       │                │ SYN-ACK       │                          │
      └───────┬───────┘                └───────┬───────┘                          │
              │                                 │                                 │
              │ Receive SYN                     │ Receive SYN-ACK                 │
              │ Send SYN-ACK                    │ Send ACK                        │
              ▼                                 ▼                                 │
      ┌───────────────┐                ┌───────────────┐                          │
      │   SYN_RCVD    │                │  ESTABLISHED  │◄─────────────────────────┘
      │               │                │               │
      │ SYN received, │                │  Connection   │
      │ SYN-ACK sent  │──────────────>│  is open      │
      │ waiting ACK   │ Receive ACK   │               │
      └───────────────┘                └───────────────┘
                                                │
                                    ┌───────────┴───────────┐
                                    │                       │
                                    │ Close requested       │
                                    │                       │
                                    ▼                       ▼
                            (Active Close)          (Passive Close)
                            ┌───────────────┐      ┌───────────────┐
                            │   FIN_WAIT_1  │      │  CLOSE_WAIT   │
                            │               │      │               │
                            │ FIN sent,     │      │ FIN received, │
                            │ waiting ACK   │      │ ACK sent,     │
                            └───────┬───────┘      │ waiting for   │
                                    │              │ app to close  │
                      Receive ACK   │              └───────┬───────┘
                            ┌───────┴───────┐              │
                            │               │              │ App calls close()
                            ▼               │              │ Send FIN
                    ┌───────────────┐       │              ▼
                    │   FIN_WAIT_2  │       │      ┌───────────────┐
                    │               │       │      │   LAST_ACK    │
                    │ Waiting for   │       │      │               │
                    │ peer's FIN    │       │      │ FIN sent,     │
                    └───────┬───────┘       │      │ waiting ACK   │
                            │               │      └───────┬───────┘
                Receive FIN │               │              │
                Send ACK    │               │ Receive FIN  │ Receive ACK
                            │               │ Send ACK     │
                            ▼               ▼              ▼
                    ┌───────────────────────────┐  ┌───────────────┐
                    │        TIME_WAIT          │  │    CLOSED     │
                    │                           │  │               │
                    │  Wait 2×MSL before        │  │  Connection   │
                    │  fully closing            │  │  terminated   │
                    │  (typically 60-120 sec)   │  │               │
                    └─────────────┬─────────────┘  └───────────────┘
                                  │
                                  │ Timeout (2×MSL)
                                  ▼
                          ┌───────────────┐
                          │    CLOSED     │
                          └───────────────┘
```

## State Descriptions

### CLOSED

```
The starting and ending state. No connection exists.
Not actually tracked—absence of connection state.
```

### LISTEN

```
Server is waiting for incoming connections.
Created by: listen() system call

$ netstat -an | grep LISTEN
tcp   0   0  0.0.0.0:80      0.0.0.0:*    LISTEN
tcp   0   0  0.0.0.0:22      0.0.0.0:*    LISTEN
```

### SYN_SENT

```
Client has sent SYN, waiting for SYN-ACK.
Created by: connect() system call

Typical duration: 1 RTT (plus retries if lost)

If you see many SYN_SENT:
  - Remote server might be down
  - Firewall blocking SYN-ACKs
  - Network connectivity issues
```

### SYN_RCVD (SYN_RECEIVED)

```
Server received SYN, sent SYN-ACK, waiting for ACK.
Part of the half-open connection.

Typical duration: 1 RTT

If you see many SYN_RCVD:
  - Could be SYN flood attack
  - Check SYN backlog settings
  - Consider SYN cookies
```

### ESTABLISHED

```
Three-way handshake complete. Data can flow.
This is the normal "connection open" state.

$ netstat -an | grep ESTABLISHED
tcp  0   0  192.168.1.100:52431  93.184.216.34:80  ESTABLISHED
```

### FIN_WAIT_1

```
Application called close(), FIN sent.
Waiting for ACK of FIN (or FIN from peer).

Brief transitional state.
```

### FIN_WAIT_2

```
FIN acknowledged, waiting for peer's FIN.
Peer's application hasn't closed yet.

Can persist if peer doesn't close:
  - Application bug (not calling close())
  - Half-close intentional

Linux: tcp_fin_timeout controls how long to wait
```

### CLOSE_WAIT

```
Received FIN from peer, sent ACK.
Waiting for local application to close.

If you see many CLOSE_WAIT:
  - Application not calling close()!
  - Resource leak / application bug
  - Common source of "too many open files"
```

### LAST_ACK

```
Sent FIN after receiving peer's FIN.
Waiting for final ACK.

Brief transitional state.
```

### TIME_WAIT

```
Connection fully closed, waiting before reuse.
The "lingering" state that often confuses people.

Duration: 2 × MSL (Maximum Segment Lifetime)
  MSL typically 30-60 seconds
  TIME_WAIT typically 60-120 seconds

Why it exists: (see below)
```

### CLOSING

```
Rare state: Both sides sent FIN simultaneously.
Each waiting for ACK of their FIN.

Simultaneous close scenario.
```

## Why TIME_WAIT Exists

TIME_WAIT serves two important purposes:

### 1. Reliable Connection Termination

```
Scenario: Final ACK is lost

Client                               Server
   │                                    │
   │──── FIN ──────────────────────────>│
   │<─── ACK, FIN ──────────────────────│
   │──── ACK ───────────X               │  ← Lost!
   │                                    │
   │    (Client in TIME_WAIT)           │  (Server in LAST_ACK)
   │                                    │
   │<─── FIN (retransmit) ──────────────│
   │──── ACK ──────────────────────────>│  (Re-ACK)
   │                                    │
   │    TIME_WAIT ensures we can        │
   │    re-acknowledge if needed        │
```

### 2. Prevent Stale Segments

```
Old connection: 192.168.1.100:52431 → 93.184.216.34:80
  Some segments still in network (delayed)

New connection with same 4-tuple:
  If TIME_WAIT didn't exist, could reuse immediately
  Old segments might be accepted as valid!

TIME_WAIT (2×MSL) ensures old segments expire:
  MSL = Maximum Segment Lifetime in network
  2×MSL = round-trip time for any lingering data
```

## TIME_WAIT Problems and Solutions

### The Problem

High-traffic servers can accumulate thousands of TIME_WAIT connections:

```
$ netstat -an | grep TIME_WAIT | wc -l
15234

Each TIME_WAIT:
  - Consumes memory (small, but adds up)
  - Holds ephemeral port (can exhaust ports!)
  - 4-tuple unavailable for new connections
```

### Solutions

**1. SO_REUSEADDR**
```python
# Allow bind() to reuse address in TIME_WAIT
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# Server can restart immediately after crash
# Doesn't allow simultaneous bind to same port
```

**2. tcp_tw_reuse (Linux)**
```bash
# Allow reusing TIME_WAIT sockets for outgoing connections
$ sysctl -w net.ipv4.tcp_tw_reuse=1

# Safe because timestamps prevent confusion
# Only for outgoing connections (client side)
```

**3. Reduce TIME_WAIT duration (careful!)**
```bash
# Not recommended - violates TCP specification
# Some systems allow it anyway

# Linux doesn't have direct control
# tcp_fin_timeout only affects FIN_WAIT_2
```

**4. Connection pooling**
```
Reuse established connections
  - HTTP Keep-Alive
  - Database connection pools
  - gRPC persistent connections

Fewer connections = fewer TIME_WAITs
```

**5. Use server-side close**
```
If server closes first → Server gets TIME_WAIT
If client closes first → Client gets TIME_WAIT

For servers with many short-lived connections:
  Let clients close first (HTTP/1.1 does this)
```

## Viewing Connection States

### Linux/macOS

```bash
# All connections with state
$ netstat -an
$ ss -an

# Count by state
$ ss -s
TCP:   2156 (estab 234, closed 1856, orphaned 12, timewait 1844)

# Filter by state
$ ss -t state established
$ ss -t state time-wait

# Show process info
$ ss -tp
$ netstat -tp
```

### State Distribution Check

```bash
# Quick state summary
$ ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
   1844 TIME-WAIT
    234 ESTAB
     56 FIN-WAIT-2
     23 CLOSE-WAIT
      5 LISTEN
      3 SYN-SENT
```

## Connection Termination: Normal vs. Abort

### Graceful Close (FIN)

```
Normal termination - all data delivered

Client:  close() → sends FIN
         Waits for peer's FIN
         Both sides agree connection is done

4-way handshake:
  FIN →
  ← ACK
  ← FIN
  ACK →

Can be combined (FIN+ACK together)
```

### Abortive Close (RST)

```
Immediate termination - may lose data

Triggers:
  - SO_LINGER with timeout=0
  - Receiving data on closed socket
  - Connection to non-listening port
  - Firewall timeout/rejection

No TIME_WAIT needed - immediate cleanup
But: any in-flight data is lost
```

### Half-Close

```
TCP allows closing one direction:

Client: shutdown(SHUT_WR)
  - Client can't send more data
  - Client can still receive
  - Server sees EOF when reading

Use case:
  "I'm done sending, but I'll wait for your response"

Example: HTTP request sent, waiting for response
```

## Common Issues

### Too Many CLOSE_WAIT

```
Symptoms:
  - Connections stuck in CLOSE_WAIT
  - "Too many open files" errors
  - Application eventually fails

Cause:
  - Application receiving FIN but not calling close()
  - Bug in cleanup code
  - Exception handling not closing sockets

Fix:
  - Fix application to properly close sockets
  - Use finally blocks / context managers
  - Check for file descriptor leaks
```

### Too Many TIME_WAIT

```
Symptoms:
  - Thousands of TIME_WAIT connections
  - Port exhaustion for outgoing connections
  - "Cannot assign requested address" errors

Cause:
  - Many short-lived outgoing connections
  - Server closing connections (gets TIME_WAIT)

Fix:
  - Connection pooling
  - tcp_tw_reuse (client-side)
  - Let clients close first (server-side)
  - Longer-lived connections
```

### SYN_RECV Accumulation

```
Symptoms:
  - Many connections in SYN_RCVD
  - New connections rejected
  - Server appears slow or unresponsive

Cause:
  - SYN flood attack
  - Slow/lossy network (ACKs not arriving)

Fix:
  - Enable SYN cookies
  - Increase SYN backlog
  - Rate limiting
  - DDoS protection
```

## Summary

TCP states track the connection lifecycle:

| State | Side | Meaning |
|-------|------|---------|
| LISTEN | Server | Waiting for connections |
| SYN_SENT | Client | Handshake in progress |
| SYN_RCVD | Server | Handshake in progress |
| ESTABLISHED | Both | Connection open |
| FIN_WAIT_1 | Closer | Sent FIN, waiting ACK |
| FIN_WAIT_2 | Closer | FIN ACKed, waiting peer FIN |
| CLOSE_WAIT | Receiver | Received FIN, app hasn't closed |
| LAST_ACK | Receiver | Sent FIN, waiting final ACK |
| TIME_WAIT | Closer | Waiting to ensure clean close |
| CLOSED | Both | No connection |

Key debugging insights:
- **CLOSE_WAIT accumulation** = application not closing sockets
- **TIME_WAIT accumulation** = many short connections (may be normal)
- **SYN_RCVD accumulation** = possible SYN flood attack

This completes our deep dive into TCP. You now understand the protocol that powers most of the internet. Next, we'll look at UDP—the simpler, faster alternative.
