# TCP Deep Dive

**TCP (Transmission Control Protocol)** transforms IP's unreliable packet delivery into a reliable, ordered byte stream. It's the foundation for most internet applications—web browsing, email, file transfer, and API calls all typically use TCP.

## What TCP Provides

TCP adds these guarantees on top of IP:

```
┌─────────────────────────────────────────────────────────────┐
│                    TCP Guarantees                           │
├─────────────────────────────────────────────────────────────┤
│  ✓ Reliable Delivery                                        │
│    Lost packets are detected and retransmitted              │
│                                                             │
│  ✓ Ordered Delivery                                         │
│    Data arrives in the order it was sent                    │
│                                                             │
│  ✓ Error Detection                                          │
│    Corrupted data is detected via checksums                 │
│                                                             │
│  ✓ Flow Control                                             │
│    Sender doesn't overwhelm receiver                        │
│                                                             │
│  ✓ Congestion Control                                       │
│    Sender doesn't overwhelm the network                     │
│                                                             │
│  ✓ Connection-Oriented                                      │
│    Explicit setup and teardown                              │
│                                                             │
│  ✓ Full-Duplex                                              │
│    Data flows in both directions simultaneously             │
└─────────────────────────────────────────────────────────────┘
```

## The Trade-offs

These guarantees come at a cost:

```
Reliability vs. Latency
──────────────────────────────────────────
TCP must wait for acknowledgments
Lost packet? Wait for retransmission
Connection setup requires round trips

Ordering vs. Throughput
──────────────────────────────────────────
Head-of-line blocking: One lost packet
stalls delivery of everything behind it

Packet:  1  2  3  4  5  6  7
              ↑
              Lost

Received: 1  2  [waiting...] 3  4  5  6  7
              │
              Can't deliver 4-7 until 3 arrives
```

This is why some applications (real-time video, gaming) use UDP instead.

## TCP vs. IP

Think of TCP and IP as two different jobs:

```
┌─────────────────────────────────────────────────────────────┐
│                          IP                                 │
│  "I'll try to get this packet to the destination address"   │
│                                                             │
│  - No guarantee of delivery                                 │
│  - No guarantee of order                                    │
│  - Packets are independent                                  │
│  - Fast, simple, stateless                                  │
└─────────────────────────────────────────────────────────────┘
                            ↑
                            │
┌─────────────────────────────────────────────────────────────┐
│                         TCP                                 │
│  "I'll make sure all data arrives correctly and in order"   │
│                                                             │
│  - Reliable delivery (detects loss, retransmits)            │
│  - Ordered delivery (sequence numbers)                      │
│  - Connection state (both sides track progress)             │
│  - Slower, complex, stateful                                │
└─────────────────────────────────────────────────────────────┘
```

## Key Concepts Preview

### Sequence Numbers

Every byte in a TCP stream has a sequence number:

```
Application sends: "Hello, World!" (13 bytes)

TCP assigns:
  Seq 1000: 'H'
  Seq 1001: 'e'
  Seq 1002: 'l'
  ...
  Seq 1012: '!'

Segments might be:
  Segment 1: Seq=1000, "Hello, "
  Segment 2: Seq=1007, "World!"
```

### Acknowledgments

The receiver tells the sender what it's received:

```
Sender                          Receiver
   │                               │
   │──── Seq=1000, "Hello" ───────>│
   │                               │
   │<──── ACK=1005 ────────────────│
   │      "I've received up to     │
   │       byte 1004, send 1005"   │
```

### The Window

The receiver advertises how much data it can accept:

```
"My receive buffer can hold 65535 more bytes"
  Window = 65535

Sender can send that much without waiting for ACKs
(Sliding window protocol)
```

## What You'll Learn

This chapter covers TCP in depth:

1. **The Three-Way Handshake**: How connections are established
2. **TCP Header and Segments**: The packet format and key fields
3. **Flow Control**: Preventing receiver overload
4. **Congestion Control**: Preventing network overload
5. **Retransmission**: How lost data is recovered
6. **TCP States**: The connection lifecycle

Understanding TCP helps you:
- Debug connection problems
- Optimize application performance
- Make informed protocol choices
- Understand why things sometimes feel slow

Let's start with the handshake—how two systems establish a TCP connection.
