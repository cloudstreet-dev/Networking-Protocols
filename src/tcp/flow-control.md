# Flow Control

**Flow control** prevents a fast sender from overwhelming a slow receiver. Without it, a server could blast data faster than your application can process it, leading to lost data and wasted retransmissions.

## The Problem

Consider a file download:

```
Fast Server                           Slow Client
(100 Mbps)                            (processes 1 MB/s)

   │──── 1 MB ────────────────────────>│ Buffer: [1MB]
   │──── 1 MB ────────────────────────>│ Buffer: [2MB]
   │──── 1 MB ────────────────────────>│ Buffer: [3MB]
   │──── 1 MB ────────────────────────>│ Buffer: [4MB] ← FULL!
   │──── 1 MB ────────────────────────>│ Buffer: OVERFLOW!
   │                                   │
   │  Data lost! Must retransmit.      │
   │  Waste of bandwidth.              │

Without flow control, fast senders can:
- Overflow receiver buffers
- Cause packet loss
- Trigger unnecessary retransmissions
```

## The Sliding Window

TCP uses a **sliding window** mechanism for flow control. The receiver advertises how much buffer space is available, and the sender limits itself accordingly.

### Receive Window (rwnd)

```
Receiver's perspective:

Receive Buffer (size: 65535 bytes)
┌─────────────────────────────────────────────────────────────┐
│ Data waiting │ Application    │      Available Space        │
│ to be read   │ reading...     │       (Window)              │
│    (ACKed)   │                │                             │
├──────────────┼────────────────┼─────────────────────────────┤
│    10000     │    (consuming) │         55535               │
└──────────────┴────────────────┴─────────────────────────────┘

Window advertised to sender: 55535 bytes
"I can accept 55535 more bytes"
```

### Sender's View

```
The sender tracks three pointers:

Sent & ACKed │ Sent, waiting for ACK │   Can send   │ Cannot send
             │                       │   (Window)   │ (beyond window)
─────────────┴───────────────────────┴──────────────┴────────────────
    1000         1000-5000              5000-10000      10000+

The "window" slides forward as ACKs arrive:

Before ACK:
[=====Sent=====][=====In Flight=====][===Can Send===][  Cannot  ]
                                     └──rwnd=5000───┘

After ACK (receiver consumed data):
        [===Sent===][===In Flight===][=====Can Send=====][Cannot]
                                     └─────rwnd=8000─────┘

Window "slides" right as data is acknowledged
```

## Window Flow

Let's trace a file transfer with flow control:

```
Sender                                              Receiver
   │        rwnd = 4000                                │
   │                                                   │
   │──── Seq=1000, 1000 bytes ────────────────────────>│
   │──── Seq=2000, 1000 bytes ────────────────────────>│
   │──── Seq=3000, 1000 bytes ────────────────────────>│
   │──── Seq=4000, 1000 bytes ────────────────────────>│
   │                                                   │
   │    (Sender has sent rwnd bytes, must wait)        │
   │                                                   │
   │<──── ACK=5000, Win=2000 ──────────────────────────│
   │      (App read 2000 bytes, 2000 space freed)      │
   │                                                   │
   │──── Seq=5000, 1000 bytes ────────────────────────>│
   │──── Seq=6000, 1000 bytes ────────────────────────>│
   │                                                   │
   │    (Window full again, wait)                      │
   │                                                   │
   │<──── ACK=7000, Win=4000 ──────────────────────────│
   │      (App caught up, more space)                  │
```

## Window Size and Throughput

The window limits throughput based on latency:

```
Maximum throughput = Window Size / Round Trip Time

Example 1: Window=65535, RTT=10ms
  Throughput ≤ 65535 / 0.010 = 6.5 MB/s

Example 2: Window=65535, RTT=100ms
  Throughput ≤ 65535 / 0.100 = 655 KB/s

This is why window scaling matters for high-latency links!
```

### Bandwidth-Delay Product (BDP)

For optimal throughput, window should be ≥ BDP:

```
BDP = Bandwidth × RTT

Example: 100 Mbps link, 50ms RTT
  BDP = 100,000,000 bits/s × 0.050 s
      = 5,000,000 bits = 625,000 bytes

Need window ≥ 625 KB to fully utilize the link!
Standard 65535-byte window is way too small.
Window scaling essential: 65535 × 2^4 = ~1MB
```

## Window Scaling

Window scaling multiplies the 16-bit window field:

```
Without scaling:
  Max window = 65535 bytes
  On 100Mbps, 50ms link: 65535/0.050 = 1.3 MB/s (10% utilization)

With scale factor 7:
  Max window = 65535 × 128 = 8.3 MB bytes
  On 100Mbps, 50ms link: 8.3M/0.050 = 166 MB/s (full utilization)

Negotiated during handshake:
  SYN: WScale=7
  SYN-ACK: WScale=8

Scale applies to window field in all subsequent segments
```

## Zero Window

When the receiver's buffer is full, it advertises window = 0:

```
Sender                                              Receiver
   │                                                   │
   │<──── ACK=5000, Win=0 ────────────────────────────│
   │      "My buffer is full, stop sending!"          │
   │                                                   │
   │    (Sender stops, starts "persist timer")        │
   │                                                   │
   │──── Window Probe (1 byte) ──────────────────────>│
   │                                                   │
   │<──── ACK=5000, Win=0 ────────────────────────────│
   │      (Still full)                                │
   │                                                   │
   │    (Wait, probe again)                           │
   │                                                   │
   │──── Window Probe (1 byte) ──────────────────────>│
   │                                                   │
   │<──── ACK=5000, Win=4000 ─────────────────────────│
   │      (Space available, resume!)                  │
```

### Persist Timer

The **persist timer** prevents deadlock when window = 0:

```
Without persist timer:
  Receiver: Window=0 (buffer full)
  Sender: Stops sending, waits for window update
  Receiver: Window update packet is lost!
  Both sides wait forever → Deadlock

With persist timer:
  Sender periodically probes with 1-byte segments
  Eventually receives window update
  No deadlock possible
```

## Silly Window Syndrome

A pathological condition where tiny windows cause inefficiency:

```
Problem scenario:
  Application reads 1 byte at a time
  Receiver advertises 1-byte window
  Sender sends 1-byte segments (huge overhead!)

1 byte payload + 20 TCP + 20 IP = 41 bytes
Efficiency: 1/41 = 2.4%

This is "Silly Window Syndrome" (SWS)
```

### Prevention

**Receiver side (Clark's algorithm):**
```
Don't advertise tiny windows.
Wait until either:
  - Window ≥ MSS, or
  - Window ≥ buffer/2

"I have space" → If space < MSS, advertise Win=0
```

**Sender side (Nagle's algorithm):**
```
Don't send tiny segments.
If there's unacknowledged data:
  - Buffer small writes
  - Wait for ACK before sending

Can be disabled with TCP_NODELAY socket option
(Important for latency-sensitive apps)
```

## Flow Control in Action

Here's a real-world example captured with tcpdump:

```
Time    Direction  Seq      ACK      Win    Len
──────────────────────────────────────────────────
0.000   →          1        1        65535  1460   # Send data
0.001   →          1461     1        65535  1460   # More data
0.050   ←          1        2921     32768  0      # ACK, window shrunk
0.051   →          2921     1        65535  1460   # Continue
0.052   →          4381     1        65535  1460
0.100   ←          1        5841     16384  0      # Window shrinking
0.101   →          5841     1        65535  1460
0.150   ←          1        7301     0      0      # ZERO WINDOW!
0.650   →          7301     1        65535  1      # Window probe
0.700   ←          1        7302     8192   0      # Window opened
0.701   →          7302     1        65535  1460   # Resume
```

## Tuning Flow Control

### Receiver Buffer Size

```bash
# Linux - check current buffer sizes
$ sysctl net.core.rmem_default
net.core.rmem_default = 212992

$ sysctl net.core.rmem_max
net.core.rmem_max = 212992

# Increase for high-bandwidth applications
$ sudo sysctl -w net.core.rmem_max=16777216
$ sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
#                                   min  default  max
```

### Application-Level Control

```python
import socket

# Create socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Set receive buffer (affects advertised window)
s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 1048576)  # 1MB

# Check actual buffer size (OS may adjust)
actual = s.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
print(f"Receive buffer: {actual}")
```

## Visualizing the Window

```
Receiver's buffer over time:

Time=0 (empty buffer, large window)
┌────────────────────────────────────────────────────────────┐
│                        Available (Win=64KB)                │
└────────────────────────────────────────────────────────────┘

Time=1 (receiving faster than app reads)
┌───────────────────────────┬────────────────────────────────┐
│    Buffered (32KB)        │      Available (Win=32KB)      │
└───────────────────────────┴────────────────────────────────┘

Time=2 (app not reading, buffer filling)
┌───────────────────────────────────────────┬────────────────┐
│           Buffered (48KB)                 │ Avail(Win=16KB)│
└───────────────────────────────────────────┴────────────────┘

Time=3 (buffer full!)
┌────────────────────────────────────────────────────────────┐
│                    Buffered (64KB) - Win=0!                │
└────────────────────────────────────────────────────────────┘

Time=4 (app reads 32KB)
┌───────────────────────────┬────────────────────────────────┐
│    Buffered (32KB)        │      Available (Win=32KB)      │
└───────────────────────────┴────────────────────────────────┘
```

## Summary

Flow control ensures receivers aren't overwhelmed:

| Mechanism | Purpose |
|-----------|---------|
| Receive Window (rwnd) | Advertises available buffer space |
| Window Scaling | Enables windows > 65535 bytes |
| Zero Window | Signals "stop sending" |
| Persist Timer | Prevents deadlock on zero window |
| Nagle's Algorithm | Prevents sending tiny segments |
| Clark's Algorithm | Prevents advertising tiny windows |

Key formulas:
```
Max throughput = Window / RTT
BDP = Bandwidth × RTT (optimal window size)
```

Flow control handles **receiver** capacity. But what about the **network** itself? That's **congestion control**—our next topic.
