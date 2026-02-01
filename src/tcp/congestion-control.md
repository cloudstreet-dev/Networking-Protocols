# Congestion Control

While flow control prevents overwhelming the *receiver*, **congestion control** prevents overwhelming the *network*. Without it, the internet would suffer from **congestion collapse**—everyone sending as fast as possible, causing massive packet loss and near-zero throughput.

## The Congestion Problem

```
Multiple senders sharing a bottleneck:

Sender A (100 Mbps) ─┐
                     │
Sender B (100 Mbps) ─┼────[Router]────> 50 Mbps link ────> Internet
                     │     (bottleneck)
Sender C (100 Mbps) ─┘

If everyone sends at full speed:
  Input: 300 Mbps
  Capacity: 50 Mbps
  Result: Router drops 250 Mbps worth of packets!

Dropped packets → Retransmissions → Even more traffic → More drops
This is "congestion collapse"
```

## TCP's Solution: Congestion Window

TCP maintains a **congestion window (cwnd)** that limits how much unacknowledged data can be in flight:

```
Effective window = min(cwnd, rwnd)

rwnd: What the receiver can accept (flow control)
cwnd: What the network can handle (congestion control)

Even if receiver says "send 1 MB", if cwnd=64KB,
sender only sends 64KB before waiting for ACKs.
```

## The Four Phases

TCP congestion control has four main phases:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  cwnd                                                       │
│    │                                                        │
│    │                    Congestion        │                 │
│    │                    Avoidance         │                 │
│    │               ____/                  │ ssthresh        │
│    │              /                       │←───────         │
│    │             /                  ______│                 │
│    │            /                  /      │                 │
│    │           /──────────────────/       │                 │
│    │          /                           │                 │
│    │         /                            │                 │
│    │        / Slow Start                  │                 │
│    │       /                              │                 │
│    │      /                               │                 │
│    │     /                                │                 │
│    │────/                                 │                 │
│    └─────────────────────────────────────────────> Time     │
│           Loss detected: cwnd cut,                          │
│           ssthresh lowered                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1. Slow Start

Despite the name, slow start grows cwnd **exponentially**:

```
Initial cwnd = 1 MSS (or IW, Initial Window, typically 10 MSS now)

Round 1: Send 1 segment, get 1 ACK → cwnd = 2
Round 2: Send 2 segments, get 2 ACKs → cwnd = 4
Round 3: Send 4 segments, get 4 ACKs → cwnd = 8
Round 4: Send 8 segments, get 8 ACKs → cwnd = 16

cwnd doubles every RTT (exponential growth)

Continues until:
  - cwnd reaches ssthresh (slow start threshold)
  - Packet loss is detected
```

Why "slow" start?
```
Before TCP had congestion control, senders would
immediately blast data at full speed. "Slow" start
is slow compared to that—it probes the network
capacity before going full throttle.
```

### 2. Congestion Avoidance

Once cwnd ≥ ssthresh, growth becomes **linear**:

```
For each RTT (when all cwnd bytes are acknowledged):
  cwnd = cwnd + MSS

Or equivalently, for each ACK:
  cwnd = cwnd + MSS/cwnd

Example (MSS=1000, cwnd=10000):
  ACK received → cwnd = 10000 + 1000/10000 = 10000.1
  After 10 ACKs → cwnd ≈ 10001
  After 100 ACKs → cwnd = 10010
  After 10000 ACKs (1 RTT) → cwnd = 11000

Linear growth: +1 MSS per RTT
This is much slower than slow start's doubling
```

### 3. Loss Detection and Response

When packet loss is detected, TCP assumes congestion:

**Triple Duplicate ACK (Fast Retransmit):**
```
Sender receives 3 duplicate ACKs for same sequence

Interpretation: "One packet lost, but others arriving"
  (Mild congestion, some packets getting through)

Response (TCP Reno/NewReno):
  ssthresh = cwnd / 2
  cwnd = ssthresh + 3 MSS  (Fast Recovery)
  Retransmit lost segment
  Stay in congestion avoidance
```

**Timeout (RTO expiration):**
```
No ACK received within timeout period

Interpretation: "Severe congestion, possibly multiple losses"
  (Major congestion, most packets lost)

Response:
  ssthresh = cwnd / 2
  cwnd = 1 MSS (or IW)
  Go back to slow start
```

### 4. Fast Recovery

After fast retransmit, enters fast recovery:

```
During Fast Recovery:
  For each duplicate ACK received:
    cwnd = cwnd + MSS
    (Indicates packets leaving network)

  When new ACK arrives (lost packet recovered):
    cwnd = ssthresh
    Exit fast recovery
    Enter congestion avoidance
```

## Congestion Control Algorithms

Different algorithms for different scenarios:

### TCP Reno (Classic)

```
The original widely-deployed algorithm

Slow Start:     Exponential growth
Cong. Avoid:    Linear growth (AIMD - Additive Increase)
Loss Response:  Multiplicative Decrease (cwnd/2)

AIMD = Additive Increase, Multiplicative Decrease
  - Increase by 1 MSS per RTT
  - Decrease by half on loss
  - Proven to converge to fair share
```

### TCP NewReno

```
Improvement over Reno for multiple losses:

Problem with Reno:
  Multiple packets lost in one window
  Fast retransmit fixes one, then exits fast recovery
  Has to wait for timeout for others

NewReno:
  Stays in fast recovery until all lost packets recovered
  "Partial ACK" triggers retransmit of next lost segment
  Much better for high loss environments
```

### TCP CUBIC (Linux Default)

```
Designed for high-bandwidth, high-latency networks

Key differences:
  - cwnd growth is cubic function of time since last loss
  - More aggressive than Reno in probing capacity
  - Better utilization of fat pipes

cwnd = C × (t - K)³ + Wmax

Where:
  C = scaling constant
  t = time since last loss
  K = time to reach Wmax
  Wmax = cwnd at last loss
```

### BBR (Bottleneck Bandwidth and RTT)

```
Google's model-based algorithm (2016)

Revolutionary approach:
  - Explicitly measures bottleneck bandwidth
  - Explicitly measures minimum RTT
  - Doesn't use loss as primary congestion signal

Phases:
  Startup:   Exponential probing (like slow start)
  Drain:     Reduce queue after startup
  Probe BW:  Cycle through bandwidth probing
  Probe RTT: Periodically measure minimum RTT

Advantages:
  - Better throughput on lossy links
  - Lower latency (doesn't fill buffers)
  - Fairer bandwidth sharing
```

## Visualizing Congestion Control

```
TCP Reno behavior over time:

cwnd │
     │            *
     │           *  *           *
     │          *    *         * *
     │         *      *       *   *
     │        *        *     *     *
     │       *          *   *       *
     │      *            * *         *
     │     *              *           *
     │    * (slow start)   Loss!      *
     │   *                  ↓          Loss!
     │  *                ssthresh set    ↓
     │ *
     │*
     └────────────────────────────────────────> Time

"Sawtooth" pattern is classic TCP Reno behavior
```

## Congestion Control in Practice

### Checking Your System's Algorithm

```bash
# Linux
$ sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = cubic

# See available algorithms
$ sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic bbr

# Change algorithm (root)
$ sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

### Per-Connection Algorithm (Linux)

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Set BBR for this connection
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_CONGESTION, b'bbr')

# Check what's set
algo = s.getsockopt(socket.IPPROTO_TCP, socket.TCP_CONGESTION, 16)
print(algo.decode())  # 'bbr'
```

## ECN: Explicit Congestion Notification

Instead of dropping packets, routers can **mark** them:

```
Traditional congestion signal:
  Router overloaded → Drops packets
  Sender sees loss → Reduces cwnd

With ECN:
  Router overloaded → Sets ECN bits in IP header
  Receiver sees ECN → Echoes to sender via TCP
  Sender reduces cwnd → No packet loss!

Benefits:
  - No lost data
  - Faster response
  - Lower latency
```

ECN requires support from:
- Routers (must mark instead of drop)
- Both TCP endpoints (must negotiate)

## Fairness

Congestion control isn't just about throughput—it's about sharing:

```
Two flows sharing a bottleneck:

Flow A: 100 Mbps network, long-running download
Flow B: 100 Mbps network, long-running download
Bottleneck: 10 Mbps

Fair outcome: Each gets ~5 Mbps

TCP's AIMD achieves this:
  - Both increase at same rate (additive)
  - Both decrease proportionally (multiplicative)
  - Over time, converges to fair share
```

### RTT Fairness Problem

```
Flow A: 10 ms RTT
Flow B: 100 ms RTT
Same bottleneck

Problem: Flow A increases cwnd 10× faster!
  A: +1 MSS every 10ms = +100 MSS/second
  B: +1 MSS every 100ms = +10 MSS/second

Flow A gets ~10× more bandwidth
This is why CUBIC and BBR were designed
```

## Bufferbloat

Excessive buffering causes latency issues:

```
Problem:
  Router has 100MB buffer
  TCP fills buffer to maximize throughput
  1000 Mbps link with 100MB buffer:
    Buffer delay = 100MB / 125MB/s = 800ms!

Packets wait in queue → High latency
TCP only reacts when buffer overflows → Too late

Solutions:
  - Active Queue Management (AQM)
  - CoDel, PIE, fq_codel
  - BBR (doesn't fill buffers)
```

## Debugging Congestion

### Symptoms

- Good bandwidth but high latency = bufferbloat
- Periodic throughput drops = congestion/loss
- Consistently low throughput = bottleneck or small cwnd

### Tools

```bash
# Linux: view cwnd and ssthresh
$ ss -ti
ESTAB  0  0  192.168.1.100:52431  93.184.216.34:80
    cubic wscale:7,7 rto:208 rtt:104/52 ato:40 mss:1448
    cwnd:10 ssthresh:7 bytes_sent:1448 bytes_acked:1449

# Trace cwnd over time
$ ss -ti | grep cwnd  # repeat or use watch

# tcptrace for analysis
$ tcptrace -l captured.pcap
```

## Summary

Congestion control prevents network overload through self-regulation:

| Phase | Growth | Trigger |
|-------|--------|---------|
| Slow Start | Exponential | cwnd < ssthresh |
| Congestion Avoidance | Linear | cwnd ≥ ssthresh |
| Fast Recovery | +1 MSS per dup ACK | 3 duplicate ACKs |
| Timeout | Reset to 1 | RTO expiration |

Key algorithms:
- **Reno**: Classic AIMD, good baseline
- **CUBIC**: Default Linux, better for fat pipes
- **BBR**: Model-based, good for lossy networks

Effective sending rate:
```
Rate = min(cwnd, rwnd) / RTT
```

Congestion control is why the internet works—millions of TCP connections sharing limited bandwidth without centralized coordination. Next, we'll look at retransmission mechanisms—how TCP actually recovers lost data.
