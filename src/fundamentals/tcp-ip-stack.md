# The TCP/IP Stack

While the OSI model is a useful teaching framework, the **TCP/IP model** is what the internet actually runs on. Developed in the 1970s by Vint Cerf and Bob Kahn, it's simpler, more pragmatic, and battle-tested by decades of real-world use.

## Four Layers vs. Seven

The TCP/IP model condenses networking into four layers:

```
┌─────────────────────────────────────────────────────────────┐
│          TCP/IP Model          │       OSI Model           │
├─────────────────────────────────────────────────────────────┤
│                                │   Application  (Layer 7)  │
│   Application Layer            │   Presentation (Layer 6)  │
│                                │   Session      (Layer 5)  │
├─────────────────────────────────────────────────────────────┤
│   Transport Layer              │   Transport    (Layer 4)  │
├─────────────────────────────────────────────────────────────┤
│   Internet Layer               │   Network      (Layer 3)  │
├─────────────────────────────────────────────────────────────┤
│   Network Access Layer         │   Data Link    (Layer 2)  │
│   (Link Layer)                 │   Physical     (Layer 1)  │
└─────────────────────────────────────────────────────────────┘
```

This simplification isn't accidental—it reflects reality. The top three OSI layers often blend together in practice, and the bottom two are typically handled by the same hardware/drivers.

## Layer 1: Network Access Layer

Also called the **Link Layer**, this combines OSI's physical and data link layers. It handles everything needed to send packets across a physical network segment.

**Responsibilities:**
- Physical transmission
- MAC addressing
- Frame formatting
- Local delivery

**The TCP/IP model is agnostic about this layer.** Whether you're using:
- Ethernet
- WiFi
- Cellular (4G/5G)
- Satellite
- Carrier pigeon (yes, there's an RFC for that: RFC 1149)

...the upper layers don't care. This abstraction is what allows the internet to work across wildly different physical media.

## Layer 2: Internet Layer

The internet layer handles logical addressing and routing. Its job is getting packets from source to destination across multiple networks.

**Key Protocol: IP (Internet Protocol)**

```
IP's Job: Get this packet from A to B, somehow.

   Network A          Network B           Network C
┌───────────┐      ┌───────────┐       ┌───────────┐
│   Host A  │      │  Router   │       │   Host B  │
│192.168.1.5├──────┤  1  ║  2  ├───────┤10.0.0.100 │
└───────────┘      └─────╨─────┘       └───────────┘

IP handles: addressing, routing, fragmentation
IP doesn't handle: reliability, ordering, delivery confirmation
```

**Other Internet Layer Protocols:**

- **ICMP** (Internet Control Message Protocol): Error reporting, ping
- **ARP** (Address Resolution Protocol): Finds MAC address for an IP
- **IGMP** (Internet Group Management Protocol): Multicast group membership

**Key Characteristics of IP:**
- **Connectionless**: Each packet is independent
- **Best-effort**: No guarantee of delivery
- **Unreliable**: Packets can be lost, duplicated, or reordered

This might seem like a weakness, but it's actually a feature. By keeping IP simple, it can be fast and widely implemented. Reliability can be added at higher layers when needed.

## Layer 3: Transport Layer

The transport layer provides end-to-end communication between applications. It's where we choose between reliability and speed.

### TCP (Transmission Control Protocol)

TCP provides reliable, ordered, error-checked delivery.

```
TCP Provides:
✓ Connection-oriented (explicit setup and teardown)
✓ Reliable delivery (acknowledgments, retransmission)
✓ Ordered delivery (sequence numbers)
✓ Flow control (don't overwhelm the receiver)
✓ Congestion control (don't overwhelm the network)

TCP Costs:
✗ Connection overhead (handshake latency)
✗ Head-of-line blocking (one lost packet stalls everything)
✗ Higher latency than UDP
```

### UDP (User Datagram Protocol)

UDP provides minimal transport services—just multiplexing and checksums.

```
UDP Provides:
✓ Connectionless (no setup overhead)
✓ Fast (minimal processing)
✓ No head-of-line blocking
✓ Optional checksum

UDP Lacks:
✗ No reliability (packets can be lost)
✗ No ordering (packets can arrive out of order)
✗ No flow control
✗ No congestion control
```

**When to use which?**

| Use Case | Protocol | Why |
|----------|----------|-----|
| Web browsing | TCP | Need complete, ordered pages |
| File transfer | TCP | Can't have missing bytes |
| Email | TCP | Reliability required |
| Video streaming | UDP* | Some loss acceptable, low latency important |
| Online gaming | UDP | Real-time updates, old data worthless |
| DNS queries | UDP | Small, single request/response |
| VoIP | UDP | Real-time, loss preferable to delay |

*Modern streaming often uses TCP or QUIC for adaptive bitrate streaming.

## Layer 4: Application Layer

The application layer is where user-facing protocols live. It combines the application, presentation, and session layers from OSI.

**Common Application Layer Protocols:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
├──────────────┬──────────────┬──────────────┬───────────────┤
│     HTTP     │     DNS      │     SMTP     │     SSH       │
│   (Web)      │   (Names)    │   (Email)    │   (Secure     │
│              │              │              │    Shell)     │
├──────────────┼──────────────┼──────────────┼───────────────┤
│     FTP      │    DHCP      │    SNMP      │     NTP       │
│   (Files)    │   (Config)   │ (Management) │    (Time)     │
└──────────────┴──────────────┴──────────────┴───────────────┘
```

This layer handles:
- Data formatting and encoding
- Session management
- Application-specific protocols
- User authentication (in many protocols)

## Putting It All Together

Let's trace what happens when you request a webpage:

```
1. APPLICATION LAYER
   Your browser creates an HTTP request:
   "GET /index.html HTTP/1.1"

2. TRANSPORT LAYER
   TCP segments the data, adds:
   - Source port (e.g., 52431)
   - Destination port (80)
   - Sequence number
   - Checksum

3. INTERNET LAYER
   IP adds:
   - Source IP (192.168.1.100)
   - Destination IP (93.184.216.34)
   - TTL (Time to Live)

4. NETWORK ACCESS LAYER
   Ethernet adds:
   - Source MAC
   - Destination MAC (router's MAC)
   - Frame check sequence

5. PHYSICAL
   Converted to electrical signals on the wire
```

On the receiving end, this process reverses—each layer strips its header and passes data up.

## The Protocol Graph

Rather than a strict stack, TCP/IP is better visualized as a graph:

```
                 ┌─────────────────────────────────────┐
                 │           Applications              │
                 │  HTTP   SMTP   DNS   SSH   Custom   │
                 └──────────────┬──────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
         ┌────┴────┐                        ┌────┴────┐
         │   TCP   │                        │   UDP   │
         └────┬────┘                        └────┬────┘
              │                                   │
              └─────────────────┬─────────────────┘
                                │
                          ┌─────┴─────┐
                          │    IP     │
                          └─────┬─────┘
                                │
         ┌────────────────┬─────┴─────┬────────────────┐
         │                │           │                │
    ┌────┴────┐     ┌────┴────┐ ┌────┴────┐     ┌────┴────┐
    │Ethernet │     │  WiFi   │ │Cellular │     │  Other  │
    └─────────┘     └─────────┘ └─────────┘     └─────────┘
```

Any application can use TCP or UDP. Both use IP. IP can run over any network technology. This flexibility is why the internet works.

## Why TCP/IP Won

The OSI model was designed by committee to be complete and correct. TCP/IP was designed by engineers to work. Key differences:

| Aspect | OSI | TCP/IP |
|--------|-----|--------|
| Design approach | Top-down, theoretical | Bottom-up, practical |
| Implementation | Came after spec | Spec described working code |
| Layer count | 7 (sometimes awkward) | 4 (pragmatic) |
| Real-world use | Reference model | Running on billions of devices |

TCP/IP's success came from:
1. **Working code first**: The spec described implementations that already worked
2. **Simplicity**: Fewer layers, clearer responsibilities
3. **Flexibility**: "Be liberal in what you accept, conservative in what you send"
4. **Open standards**: Anyone could implement it

## Summary

The TCP/IP model is the practical foundation of the internet:

| Layer | Function | Key Protocols |
|-------|----------|---------------|
| Application | User services | HTTP, DNS, SMTP, SSH |
| Transport | End-to-end delivery | TCP, UDP |
| Internet | Routing between networks | IP, ICMP |
| Network Access | Local delivery | Ethernet, WiFi |

Understanding this model—especially the separation between IP (best-effort routing) and TCP (reliable delivery)—is essential for understanding how the internet works.

Next, we'll look at how data is wrapped and unwrapped as it moves through these layers: encapsulation.
