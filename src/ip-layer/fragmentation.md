# IP Fragmentation

Different network links have different maximum packet sizes. When a packet is too large for a link, it must be **fragmented**—split into smaller pieces. Understanding fragmentation helps you diagnose performance problems and configure networks properly.

## MTU: Maximum Transmission Unit

The **MTU** is the largest packet size a link can carry:

```
┌─────────────────────────────────────────────────────────────┐
│                    Common MTU Values                        │
├─────────────────────────────────────────────────────────────┤
│  Ethernet:              1500 bytes (standard)               │
│  Jumbo Frames:          9000 bytes (data centers)           │
│  PPPoE (DSL):           1492 bytes                          │
│  VPN Tunnels:           ~1400-1450 bytes (overhead)         │
│  IPv6 minimum:          1280 bytes                          │
│  Dial-up (PPP):         576 bytes (historical)              │
└─────────────────────────────────────────────────────────────┘
```

When a packet exceeds the outgoing link's MTU, something must happen.

## How Fragmentation Works

IPv4 routers can fragment packets when needed:

```
Original Packet (3000 bytes payload + 20 byte header = 3020 bytes)
┌──────────────────────────────────────────────────────────────┐
│IP Hdr│                     Payload (3000 bytes)              │
│ 20B  │                                                       │
└──────────────────────────────────────────────────────────────┘

Link MTU: 1500 bytes
Max payload per fragment: 1500 - 20 = 1480 bytes

After Fragmentation:
┌────────────────────────────────┐
│IP Hdr│  Fragment 1 (1480 B)   │  Offset: 0, MF=1
│ 20B  │  ID: 12345             │
└────────────────────────────────┘
┌────────────────────────────────┐
│IP Hdr│  Fragment 2 (1480 B)   │  Offset: 1480, MF=1
│ 20B  │  ID: 12345             │
└────────────────────────────────┘
┌─────────────────────┐
│IP Hdr│ Fragment 3   │  Offset: 2960, MF=0 (last fragment)
│ 20B  │   (40 B)     │  ID: 12345
└─────────────────────┘
```

### Fragmentation Header Fields

Three IP header fields manage fragmentation:

```
┌───────────────────────────────────────────────────────────────┐
│  Identification (16 bits)                                     │
│    Unique ID for the original packet                          │
│    All fragments share the same ID                            │
├───────────────────────────────────────────────────────────────┤
│  Flags (3 bits)                                               │
│    Bit 0: Reserved (must be 0)                                │
│    Bit 1: DF (Don't Fragment)                                 │
│            0 = May fragment                                   │
│            1 = Don't fragment (drop if too big)               │
│    Bit 2: MF (More Fragments)                                 │
│            0 = Last fragment (or unfragmented)                │
│            1 = More fragments follow                          │
├───────────────────────────────────────────────────────────────┤
│  Fragment Offset (13 bits)                                    │
│    Position of this fragment in original packet               │
│    Measured in 8-byte units (not bytes!)                      │
│    Max offset: 8191 × 8 = 65,528 bytes                        │
└───────────────────────────────────────────────────────────────┘
```

### Fragment Offset Calculation

Fragment offsets must be multiples of 8 bytes:

```
Original payload: 3000 bytes
MTU: 1500 bytes
Max payload per fragment: 1480 bytes (must be multiple of 8)

Fragment 1:
  Offset: 0 (bytes) / 8 = 0
  Size: 1480 bytes
  MF: 1 (more fragments)

Fragment 2:
  Offset: 1480 / 8 = 185
  Size: 1480 bytes
  MF: 1 (more fragments)

Fragment 3:
  Offset: 2960 / 8 = 370
  Size: 40 bytes (remaining)
  MF: 0 (last fragment)
```

## Reassembly

Fragments are reassembled only at the **final destination**, not at intermediate routers:

```
Sender → Router1 → Router2 → Router3 → Receiver
                                        │
                                        ▼
                             ┌──────────────────┐
                             │   Reassembly     │
                             │                  │
                             │  Wait for all    │
                             │  fragments with  │
                             │  same ID         │
                             │                  │
                             │  Arrange by      │
                             │  offset          │
                             │                  │
                             │  Check MF=0 for  │
                             │  last piece      │
                             │                  │
                             │  Reconstruct     │
                             │  original packet │
                             └──────────────────┘
```

### Reassembly Timeout

If fragments don't all arrive within a timeout (typically 30-120 seconds), the partial packet is discarded:

```
Fragment 1: ✓ Received
Fragment 2: ✓ Received
Fragment 3: ✗ Lost

After timeout:
  All fragments discarded
  ICMP "Fragment Reassembly Time Exceeded" may be sent
  Upper layer (TCP) must retransmit entire original packet
```

## Problems with Fragmentation

Fragmentation has significant drawbacks:

### 1. Performance Overhead

```
Single 3000-byte packet vs. 3 fragments:

Original (1 packet):
  Processing: 1 header lookup
  Transmission: 1 packet

Fragmented (3 packets):
  Processing: 3 header lookups (3x)
  Headers: 60 bytes (vs. 20)
  Reassembly: Buffer management, timeout tracking
```

### 2. Fragment Loss Amplification

If any fragment is lost, the entire packet is lost:

```
3 fragments, 1% loss rate each:

Probability all arrive = 0.99³ = 97%
Probability of packet loss = 3%

vs. unfragmented: 1% loss

More fragments = higher effective loss rate
```

### 3. Security Issues

- **Tiny fragment attacks**: Malicious fragments too small to contain port numbers
- **Overlapping fragment attacks**: Crafted to bypass firewalls
- **Fragment flood DoS**: Exhaust reassembly buffers

Many firewalls drop fragments by default.

### 4. Stateful Firewall Problems

```
Firewall examines:
  Source/Dest IP: In every fragment ✓
  Source/Dest Port: Only in FIRST fragment!

Fragment 2 arrives first:
  No port information
  Firewall can't apply port-based rules
  May drop or pass incorrectly
```

## Path MTU Discovery (PMTUD)

Modern systems avoid fragmentation using **Path MTU Discovery**:

```
1. Sender sends packet with DF (Don't Fragment) bit set
2. If packet is too large, router sends ICMP "Fragmentation Needed"
3. Sender reduces packet size and retries
4. Repeat until path MTU is found

┌────────┐                              ┌────────┐
│ Sender │                              │  Dest  │
└───┬────┘                              └───┬────┘
    │                                       │
    │──── 1500 byte packet, DF=1 ──────────>│
    │                                       │
    │        ┌────────┐                     │
    │        │ Router │                     │
    │        │MTU=1400│                     │
    │        └───┬────┘                     │
    │            │                          │
    │<── ICMP "Frag Needed, MTU=1400" ──────│
    │                                       │
    │──── 1400 byte packet, DF=1 ──────────>│
    │                                       │
    │<─────────── Response ─────────────────│
```

### PMTUD Problems

**Black Hole Routers**: Some routers don't send ICMP messages (or firewalls block them):

```
Sender → Router1 → Router2 → Dest
          │
          └── Has MTU 1400
              Drops packet (too big, DF=1)
              Doesn't send ICMP (broken/filtered)

Sender keeps trying 1500-byte packets
All silently dropped = "black hole"
```

**Workarounds:**
- MSS clamping (TCP)
- Fallback to minimum MTU
- Manual MTU configuration

## TCP and MTU

TCP uses the **MSS (Maximum Segment Size)** to avoid IP fragmentation:

```
MSS = MTU - IP header - TCP header
MSS = 1500 - 20 - 20 = 1460 bytes (typical)

TCP segments are sized to fit in one IP packet:
┌──────────────────────────────────────────────────┐
│ IP Hdr │ TCP Hdr │      TCP Data (≤ MSS)        │
│  20B   │   20B   │         1460 bytes           │
└──────────────────────────────────────────────────┘
             Total: ≤ MTU (1500 bytes)
```

MSS is negotiated during the TCP handshake:

```
Client → SYN, MSS=1460
Server → SYN-ACK, MSS=1400
         (Server's path MTU is smaller)

Connection uses minimum: MSS=1400
```

## IPv6 and Fragmentation

IPv6 handles fragmentation differently:

```
IPv4:
- Routers can fragment
- Sender can fragment
- Minimum MTU: 576 bytes

IPv6:
- Routers CANNOT fragment (must be done at source)
- PMTUD is mandatory
- Minimum MTU: 1280 bytes
- Uses Fragment extension header
```

IPv6's approach improves router performance (no fragmentation processing) but requires working PMTUD.

## Practical Considerations

### Checking MTU

```bash
# Linux - show interface MTU
$ ip link show eth0
2: eth0: <BROADCAST,MULTICAST,UP> mtu 1500

# macOS
$ ifconfig en0 | grep mtu

# Test path MTU with ping
$ ping -M do -s 1472 example.com   # Linux: -M do = DF bit
$ ping -D -s 1472 example.com      # macOS: -D = DF bit
# 1472 + 8 (ICMP) + 20 (IP) = 1500

# If it fails, reduce size until it works
```

### Setting MTU

```bash
# Linux - temporary
$ sudo ip link set eth0 mtu 1400

# Linux - permanent (varies by distro)
# Netplan (Ubuntu):
# /etc/netplan/01-network.yaml
network:
  ethernets:
    eth0:
      mtu: 1400

# Windows
> netsh interface ipv4 set subinterface "Ethernet" mtu=1400
```

### Common MTU Issues

**VPN tunnels:**
```
Original: 1500 MTU
VPN overhead: ~60-80 bytes (encryption, headers)
Effective MTU: ~1420-1440 bytes

If not configured, causes fragmentation or black holes
```

**Docker/containers:**
```
Host MTU: 1500
Container default: 1500
Overlay network: Adds headers

May need: MTU 1450 or lower inside containers
```

**PPPoE (DSL):**
```
Ethernet MTU: 1500
PPPoE overhead: 8 bytes
Effective: 1492 MTU

ISP-provided routers usually handle this
Manual configurations may need adjustment
```

## Debugging Fragmentation Issues

### Symptoms

- Large file transfers fail, small requests work
- Connections hang during data transfer
- Works on LAN, fails over VPN/WAN
- PMTUD blackhole (DF packets disappear)

### Diagnostics

```bash
# Check if fragmentation is occurring
$ netstat -s | grep -i frag   # Linux
fragments received
fragments created

# Tcpdump for fragments
$ tcpdump -i eth0 'ip[6:2] & 0x3fff != 0'

# Test specific sizes
$ ping -M do -s SIZE destination

# Traceroute with MTU discovery
$ tracepath example.com
```

## Summary

IP fragmentation handles oversized packets but comes with costs:

| Aspect | Impact |
|--------|--------|
| Performance | Multiple packets, reassembly overhead |
| Reliability | One lost fragment = lost packet |
| Security | Fragment attacks, firewall issues |
| Modern approach | Avoid via PMTUD, MSS clamping |

Best practices:
- Design for 1500-byte MTU (or smaller if tunneling)
- Use PMTUD where possible
- Configure MSS clamping on border routers
- Test with large packets during deployment

IPv6 eliminates router fragmentation entirely, making PMTUD mandatory but more predictable.

This completes our coverage of the IP layer. Next, we'll dive deep into TCP—the protocol that provides reliable, ordered delivery on top of IP's best-effort service.
