# The Three-Way Handshake

Before TCP can transfer data, both sides must establish a **connection**. This happens through a three-message exchange called the **three-way handshake**.

## Why a Handshake?

The handshake serves several purposes:

1. **Verify both endpoints are reachable and willing**
2. **Exchange initial sequence numbers** (ISNs)
3. **Negotiate connection parameters** (MSS, window scaling, etc.)
4. **Synchronize state** between client and server

## The Three Steps

```
    Client                                    Server
       │                                         │
       │         State: LISTEN                   │
       │         (waiting for connections)       │
       │                                         │
  ┌────┴────┐                                    │
  │  SYN    │                                    │
  │ Seq=100 │────────────────────────────────────>
  │         │        "I want to connect,         │
  └─────────┘         my ISN is 100"             │
       │                                         │
       │                                    ┌────┴────┐
       │                                    │ SYN-ACK │
       <─────────────────────────────────────│Seq=300 │
       │    "OK, I acknowledge your SYN      │ACK=101 │
       │     (expecting byte 101 next),      └────────┘
       │     and here's my ISN: 300"              │
       │                                         │
  ┌────┴────┐                                    │
  │  ACK    │                                    │
  │ACK=301  │────────────────────────────────────>
  │         │    "I acknowledge your SYN,        │
  └─────────┘     expecting byte 301"            │
       │                                         │
       │       CONNECTION ESTABLISHED            │
       │                                         │
```

### Step 1: SYN (Synchronize)

Client initiates the connection:

```
TCP Header:
┌─────────────────────────────────────────┐
│  Source Port: 52431                     │
│  Dest Port: 80                          │
│  Sequence Number: 100 (client's ISN)    │
│  Acknowledgment: 0 (not yet used)       │
│  Flags: SYN=1                           │
│  Window: 65535                          │
│  Options: MSS=1460, Window Scale, etc.  │
└─────────────────────────────────────────┘
```

The **Initial Sequence Number (ISN)** is randomized for security reasons (prevents sequence prediction attacks).

### Step 2: SYN-ACK

Server acknowledges and synchronizes:

```
TCP Header:
┌─────────────────────────────────────────┐
│  Source Port: 80                        │
│  Dest Port: 52431                       │
│  Sequence Number: 300 (server's ISN)    │
│  Acknowledgment: 101 (client's ISN + 1) │
│  Flags: SYN=1, ACK=1                    │
│  Window: 65535                          │
│  Options: MSS=1460, Window Scale, etc.  │
└─────────────────────────────────────────┘
```

The ACK value (101) means "I've received everything up to byte 100, send me byte 101 next."

### Step 3: ACK

Client confirms:

```
TCP Header:
┌─────────────────────────────────────────┐
│  Source Port: 52431                     │
│  Dest Port: 80                          │
│  Sequence Number: 101                   │
│  Acknowledgment: 301 (server's ISN + 1) │
│  Flags: ACK=1                           │
│  Window: 65535                          │
└─────────────────────────────────────────┘
```

At this point, both sides have verified connectivity and exchanged initial sequence numbers.

## Why Three Messages?

Could we do it in two? No—here's why:

```
Two-way handshake problem:

Client ──SYN──> Server
Client <──ACK── Server

What if the server's ACK is lost?
- Server thinks connection is established
- Client thinks connection failed
- Server waits forever for data that won't come

Three-way solves this:
- Both sides must acknowledge the other's SYN
- Both sides know the other received their ISN
```

## State Changes During Handshake

```
Client States                    Server States
────────────────────────────────────────────────────

CLOSED                           CLOSED
   │                                │
   │                              listen()
   │                                │
   │                                ▼
   │                             LISTEN
   │                                │
connect()                           │
   │                                │
   ▼                                │
SYN_SENT ──────── SYN ─────────────>│
   │                                │
   │                                ▼
   │                            SYN_RCVD
   │                                │
   │<─────────── SYN-ACK ───────────│
   │                                │
   ▼                                │
ESTABLISHED ────── ACK ────────────>│
   │                                │
   │                                ▼
   │                           ESTABLISHED
```

## Options Negotiated in Handshake

Several important options are exchanged during the SYN and SYN-ACK:

### Maximum Segment Size (MSS)

```
"The largest TCP segment I can receive"

Typical values:
  Ethernet: MSS = 1460 (1500 MTU - 20 IP - 20 TCP)
  Jumbo:    MSS = 8960 (9000 MTU - headers)

Both sides advertise their MSS
Connection uses the minimum of the two
```

### Window Scaling (RFC 7323)

```
Original window field: 16 bits = max 65535 bytes
Too small for high-bandwidth, high-latency networks

Window scaling multiplies by 2^scale:
  Scale=7: Window can be 65535 × 128 = 8MB

SYN:     Window Scale = 7
SYN-ACK: Window Scale = 8

Enables large windows for high-performance networks
```

### Selective Acknowledgment (SACK)

```
"I support SACK - I can tell you exactly which
 bytes I've received, not just the contiguous ones"

Without SACK: ACK=1000 means "got 1-999"
              If 1000 is lost but 1001-2000 arrived,
              can only ACK up to 999

With SACK:   ACK=1000, SACK: 1001-2000
             "Got 1-999 and 1001-2000, missing 1000"
             Sender retransmits only byte 1000
```

### Timestamps (RFC 7323)

```
Used for:
1. RTT measurement (Round Trip Time)
2. PAWS (Protection Against Wrapped Sequences)

TSval:  Sender's timestamp
TSecr:  Echoed timestamp from peer

Helps with:
- Accurate timeout calculation
- Distinguishing old duplicate packets
```

## Handshake in Action

Here's a real handshake captured with tcpdump:

```
$ tcpdump -i eth0 port 80

14:23:15.123456 IP 192.168.1.100.52431 > 93.184.216.34.80:
    Flags [S], seq 1823761425, win 65535,
    options [mss 1460,sackOK,TS val 1234567 ecr 0,
             nop,wscale 7], length 0

14:23:15.156789 IP 93.184.216.34.80 > 192.168.1.100.52431:
    Flags [S.], seq 2948572615, ack 1823761426, win 65535,
    options [mss 1460,sackOK,TS val 9876543 ecr 1234567,
             nop,wscale 8], length 0

14:23:15.156801 IP 192.168.1.100.52431 > 93.184.216.34.80:
    Flags [.], ack 2948572616, win 512,
    options [nop,nop,TS val 1234568 ecr 9876543], length 0
```

Reading the flags:
- `[S]` = SYN
- `[S.]` = SYN-ACK (the dot means ACK is set)
- `[.]` = ACK only

## Connection Latency

The handshake adds latency before data transfer can begin:

```
Timeline:
────────────────────────────────────────────────────────
0ms      Client sends SYN
                            │
50ms                        Server receives SYN
                            Server sends SYN-ACK
                            │
100ms    Client receives SYN-ACK
         Client sends ACK
         Client can NOW send data!
                            │
150ms                       Server receives ACK
                            Server can NOW send data!

Minimum: 1 RTT before client can send
         1.5 RTT before server can send

For a 100ms RTT connection:
  100ms before HTTP request can be sent
  150ms before HTTP response can begin
```

This is why connection reuse (HTTP keep-alive, connection pooling) matters for performance.

## TCP Fast Open (TFO)

**TCP Fast Open** allows data in the SYN packet:

```
First connection (normal):
Client ──SYN──────────────> Server
Client <──SYN-ACK + Cookie── Server
Client ──ACK──────────────> Server
Client ──Data─────────────> Server

Subsequent connections (with TFO cookie):
Client ──SYN + Cookie + Data──> Server
                                Server can respond immediately!
Client <───────Response───────── Server

Saves 1 RTT on repeat connections!
```

TFO requires:
- Both client and server support
- Idempotent initial request (retry-safe)
- Not universally deployed due to middlebox issues

## Handshake Failures

### Connection Refused

```
Client ──SYN──> Server (no service on port)
Client <──RST── Server

"RST" (Reset) means "I'm not accepting connections on this port"

$ telnet example.com 12345
Connection refused
```

### Connection Timeout

```
Client ──SYN──> (packet lost or server unreachable)
         ... wait ...
Client ──SYN──> (retry 1)
         ... wait longer ...
Client ──SYN──> (retry 2)
         ... give up after multiple attempts

Typical: 3 retries over ~75 seconds
```

### SYN Flood Attack

```
Attacker sends many SYNs without completing handshake:

Attacker ──SYN (fake source)──> Server
Attacker ──SYN (fake source)──> Server
Attacker ──SYN (fake source)──> Server
           ... thousands more ...

Server:
- Allocates resources for each half-open connection
- SYN queue fills up
- Can't accept legitimate connections

Mitigations:
- SYN cookies (stateless SYN handling)
- Rate limiting
- Larger SYN queues
```

## Simultaneous Open (Rare)

Both sides can simultaneously send SYN:

```
Client ──SYN──> <──SYN── Server
       ↓              ↓
Client ──SYN-ACK──> <──SYN-ACK── Server

Both sides:
1. Receive SYN while in SYN_SENT
2. Send SYN-ACK
3. Move to ESTABLISHED when ACK received

Same result, different path. Rare in practice.
```

## Summary

The three-way handshake establishes TCP connections:

| Step | Direction | Flags | Purpose |
|------|-----------|-------|---------|
| 1 | Client → Server | SYN | "I want to connect" |
| 2 | Server → Client | SYN-ACK | "OK, I acknowledge" |
| 3 | Client → Server | ACK | "Confirmed" |

Key points:
- Exchanges initial sequence numbers
- Negotiates options (MSS, window scale, SACK)
- Takes 1-1.5 RTT before data can flow
- Connection reuse avoids repeated handshakes
- SYN cookies protect against SYN floods

Next, we'll examine the TCP header in detail—understanding each field and how they work together.
