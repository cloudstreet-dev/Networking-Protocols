# Retransmission Mechanisms

TCP guarantees reliable delivery by detecting lost packets and retransmitting them. This chapter explores how TCP knows when to retransmit and the mechanisms it uses to recover efficiently.

## The Challenge

IP provides no delivery guarantee. Packets can be:
- Lost (router overflow, corruption, route failure)
- Duplicated (rare, but possible)
- Reordered (different paths)
- Delayed (congestion, buffering)

TCP must distinguish between these cases and respond appropriately.

## How TCP Detects Loss

TCP uses two primary loss detection mechanisms:

### 1. Timeout (RTO)

If no ACK arrives within the **Retransmission Timeout (RTO)**, assume the packet is lost:

```
Sender                               Receiver
   │                                    │
   │─── Seq=1000 (data) ───────X        │  ← Packet lost!
   │                                    │
   │    [Timer starts]                  │
   │    [waiting...]                    │
   │    [RTO expires!]                  │
   │                                    │
   │─── Seq=1000 (retransmit) ─────────>│
   │                                    │
   │<── ACK=1500 ───────────────────────│
```

### 2. Fast Retransmit (Triple Duplicate ACK)

Three duplicate ACKs indicate a packet was lost but later packets arrived:

```
Sender                               Receiver
   │                                    │
   │─── Seq=1000 ──────────────────────>│
   │─── Seq=1500 ─────────X             │  ← Lost!
   │─── Seq=2000 ──────────────────────>│
   │─── Seq=2500 ──────────────────────>│
   │─── Seq=3000 ──────────────────────>│
   │                                    │
   │<── ACK=1500 ──────────────────────│  (got 1000, want 1500)
   │<── ACK=1500 (dup 1) ──────────────│  (got 2000, still want 1500)
   │<── ACK=1500 (dup 2) ──────────────│  (got 2500, still want 1500)
   │<── ACK=1500 (dup 3) ──────────────│  (got 3000, still want 1500)
   │                                    │
   │   [3 dup ACKs = loss!]             │
   │                                    │
   │─── Seq=1500 (retransmit) ─────────>│
   │                                    │
   │<── ACK=3500 ──────────────────────│  (got everything!)
```

Fast retransmit is faster than waiting for timeout—often by hundreds of milliseconds.

## Retransmission Timeout (RTO) Calculation

RTO must adapt to network conditions:

```
Too short: Unnecessary retransmissions (network already delivered)
Too long:  Slow recovery from actual loss

RTO is calculated from measured RTT:

SRTT = (1 - α) × SRTT + α × RTT_sample
       (Smoothed RTT, exponential moving average)
       α = 1/8

RTTVAR = (1 - β) × RTTVAR + β × |SRTT - RTT_sample|
         (RTT variance)
         β = 1/4

RTO = SRTT + max(G, 4 × RTTVAR)
      G = clock granularity (typically 1ms)

Example:
  SRTT = 100ms, RTTVAR = 25ms
  RTO = 100 + 4×25 = 200ms
```

### RTO Bounds

```
Minimum RTO: Typically 200ms (RFC 6298 recommends 1 second!)
Maximum RTO: Typically 120 seconds

Initial RTO: 1 second (before any measurements)
```

### RTO Backoff

On repeated timeouts, RTO doubles (exponential backoff):

```
1st timeout: RTO = 200ms
2nd timeout: RTO = 400ms
3rd timeout: RTO = 800ms
4th timeout: RTO = 1600ms
...
Gives up after max retries (typically ~15)

This prevents overwhelming an already congested network.
```

## Selective Acknowledgment (SACK)

SACK dramatically improves retransmission efficiency when multiple packets are lost:

### Without SACK

```
Lost: packets 3 and 5 out of 1,2,3,4,5,6,7

Sender                               Receiver
   │                                    │
   │ Receives ACK=3                     │
   │ (receiver has 1,2)                 │
   │                                    │
   │ Retransmits 3                      │
   │                                    │
   │ Receives ACK=5                     │
   │ (receiver has 1,2,3,4)             │
   │                                    │
   │ Retransmits 5                      │
   │                                    │
   │ Finally ACK=8                      │

Each loss requires a separate round trip!
```

### With SACK

```
Lost: packets 3 and 5

Sender                               Receiver
   │                                    │
   │ Receives:                          │
   │   ACK=3, SACK=4-5,6-8              │
   │   "Got 1-2 (ack), 4-5 (sack),      │
   │    6-8 (sack). Missing: 3, 5"      │
   │                                    │
   │ Retransmits 3 and 5 together       │
   │                                    │
   │ Receives ACK=8                     │

Both lost packets identified and retransmitted in one round trip!
```

### SACK Format

```
TCP Option:
┌─────────┬────────┬─────────────┬─────────────┬─────┐
│ Kind=5  │ Length │ Left Edge 1 │ Right Edge 1│ ... │
│ (1 byte)│(1 byte)│  (4 bytes)  │  (4 bytes)  │     │
└─────────┴────────┴─────────────┴─────────────┴─────┘

Example: SACK 5001-6000, 7001-9000
  "I have bytes 5001-6000 and 7001-9000"
  "I'm missing 1-5000 and 6001-7000"

Maximum 4 SACK blocks (40 bytes option max, minus timestamps)
```

## Duplicate Detection

TCP must handle duplicate packets (from retransmission or network duplication):

### Sequence Number Check

```
Receiver tracks:
  RCV.NXT = next expected sequence number

Incoming sequence < RCV.NXT?
  → Duplicate! Already received. Discard (but still ACK).

Example:
  RCV.NXT = 5000
  Packet arrives: Seq=3000
  Already have this, discard.
```

### PAWS (Protection Against Wrapped Sequences)

For high-speed connections, sequence numbers can wrap:

```
32-bit sequence: 0 to 4,294,967,295

At 1 Gbps: wraps every ~34 seconds
At 10 Gbps: wraps every ~3.4 seconds

Problem:
  Old duplicate from previous wrap could be accepted
  as valid data!

Solution: Timestamps
  Each segment has timestamp
  Old segment has old timestamp
  Even if sequence matches, timestamp reveals age
  Reject segments with timestamps too old
```

## Spurious Retransmissions

Sometimes TCP retransmits unnecessarily:

```
Causes:
  - RTT suddenly increased (but packet not lost)
  - Delay spike on reverse path (ACK delayed)
  - RTO calculated too aggressively

Problems:
  - Wastes bandwidth
  - cwnd reduced unnecessarily
  - Triggers congestion response

Mitigations:
  - F-RTO: Detect spurious timeout retransmissions
  - Eifel algorithm: Use timestamps to detect
  - DSACK: Receiver reports duplicate segments received
```

### D-SACK (Duplicate SACK)

Receiver tells sender about duplicate segments:

```
Sender retransmits Seq=1000 (timeout)
Original arrives late at receiver
Retransmit also arrives

Receiver sends:
  ACK=2000, SACK=1000-1500 (D-SACK)
  "You already sent this, I got it twice"

Sender learns: My RTO was too aggressive!
Can adjust RTO calculation.
```

## Retransmission in Action

Real-world packet capture showing loss and recovery:

```
Time     Direction  Seq      ACK      Flags  Notes
──────────────────────────────────────────────────────────────
0.000    →          1000              PSH    Send data
0.001    →          1500              PSH    Send more
0.002    →          2000              PSH    Lost!
0.003    →          2500              PSH    Send more
0.004    →          3000              PSH    Send more

0.050    ←                   1500           ACK (got 1000)
0.051    ←                   2000           ACK (got 1500)
0.052    ←                   2000     DUP   DupACK 1 (gap!)
0.053    ←                   2000     DUP   DupACK 2
0.054    ←                   2000     DUP   DupACK 3

0.055    →          2000              PSH    Fast retransmit!

0.105    ←                   3500           ACK (recovered!)
```

## Optimizations

### Tail Loss Probe (TLP)

Probes for loss when the sender goes idle:

```
Problem:
  Send last segment of request
  Segment lost
  No more data to send → No duplicate ACKs
  Must wait for full RTO

TLP solution:
  If no ACK within 2×SRTT after sending:
    Retransmit last segment (or send new probe)
    Triggers immediate feedback

Reduces tail latency significantly.
```

### Early Retransmit

Allows fast retransmit with fewer than 3 dup ACKs:

```
Traditional: Need 3 dup ACKs
Problem: What if only 2 packets in flight?

Early retransmit:
  Small window (< 4 segments)
  Allow fast retransmit with just 1-2 dup ACKs
  Better for small transfers
```

### RACK (Recent ACKnowledgment)

Time-based loss detection:

```
Traditional: Count duplicate ACKs
Problem: Reordering looks like loss

RACK approach:
  Track time of most recent ACK
  If segment sent before recent ACK hasn't been ACKed:
    Probably lost (not reordered)

Better handles reordering vs. loss distinction
```

## Configuration

### Linux Tuning

```bash
# View retransmission stats
$ netstat -s | grep -i retrans
    1234 segments retransmitted
    567 fast retransmits

# RTO settings
$ sysctl net.ipv4.tcp_retries1  # Soft threshold
net.ipv4.tcp_retries1 = 3

$ sysctl net.ipv4.tcp_retries2  # Hard maximum
net.ipv4.tcp_retries2 = 15

# Enable SACK (usually default)
$ sysctl net.ipv4.tcp_sack
net.ipv4.tcp_sack = 1

# Enable TLP
$ sysctl net.ipv4.tcp_early_retrans
net.ipv4.tcp_early_retrans = 3  # TLP enabled
```

### Monitoring Retransmissions

```bash
# Count retransmits on a connection
$ ss -ti dst example.com
    ... retrans:5/10 ...
         │      └── Total retransmits
         └── Unrecovered retransmits

# Watch for retransmissions
$ tcpdump -i eth0 'tcp[tcpflags] & tcp-syn == 0' | grep -i retrans
```

## Summary

TCP uses multiple mechanisms to recover from loss:

| Mechanism | Detection | Speed | Use Case |
|-----------|-----------|-------|----------|
| Timeout (RTO) | Timer expires | Slow | Last resort |
| Fast Retransmit | 3 dup ACKs | Fast | Most losses |
| SACK | Explicit gaps | Fast | Multiple losses |
| TLP | Probe timeout | Fast | Tail losses |

RTO calculation:
```
RTO = SRTT + 4 × RTTVAR
```

Key principles:
- Fast retransmit beats waiting for timeout
- SACK enables efficient multi-loss recovery
- Timestamps help detect spurious retransmissions
- Modern algorithms (RACK) improve reordering tolerance

Understanding retransmission helps you diagnose network issues. High retransmission rates indicate packet loss—which could be congestion, bad hardware, or misconfiguration.

Next, we'll cover TCP states—the lifecycle of a TCP connection from creation to termination.
