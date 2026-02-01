# Encapsulation

**Encapsulation** is the process by which each layer wraps data with its own header (and sometimes trailer) information. It's how layers communicate without knowing about each other's internals.

## The Concept

Think of encapsulation like mailing a letter:

```
1. You write a letter                    [Your message]
2. Put it in an envelope                 [+ Your address, recipient address]
3. The post office puts it in a bin      [+ Sorting codes, routing info]
4. The bin goes in a truck               [+ Truck manifest, destination hub]
```

Each layer adds information needed for its job, without looking inside what it received.

## Layer-by-Layer Encapsulation

Let's trace a web request through the TCP/IP stack:

```
┌─────────────────────────────────────────────────────────────────┐
│  Application Layer                                               │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              HTTP Request (Data)                          │  │
│  │  "GET /index.html HTTP/1.1\r\nHost: example.com\r\n\r\n"  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Transport Layer (TCP)                                           │
│                                                                  │
│  ┌──────────────┬────────────────────────────────────────────┐  │
│  │ TCP Header   │              Data                          │  │
│  │ (20+ bytes)  │  (HTTP Request from above)                 │  │
│  └──────────────┴────────────────────────────────────────────┘  │
│                                                                  │
│  TCP Segment                                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Internet Layer (IP)                                             │
│                                                                  │
│  ┌──────────────┬────────────────────────────────────────────┐  │
│  │  IP Header   │              Data                          │  │
│  │  (20+ bytes) │  (TCP Segment from above)                  │  │
│  └──────────────┴────────────────────────────────────────────┘  │
│                                                                  │
│  IP Packet (or Datagram)                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Network Access Layer (Ethernet)                                 │
│                                                                  │
│  ┌──────────────┬────────────────────────────────────┬───────┐  │
│  │Ethernet Hdr  │              Data                  │  FCS  │  │
│  │  (14 bytes)  │  (IP Packet from above)            │(4 B)  │  │
│  └──────────────┴────────────────────────────────────┴───────┘  │
│                                                                  │
│  Ethernet Frame                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Terminology

Different layers use different names for their data units:

```
┌─────────────────────────────────────────┐
│  Layer        │  Data Unit Name        │
├─────────────────────────────────────────┤
│  Application  │  Message / Data        │
│  Transport    │  Segment (TCP)         │
│               │  Datagram (UDP)        │
│  Internet     │  Packet                │
│  Network      │  Frame                 │
└─────────────────────────────────────────┘
```

These terms matter when debugging—if someone mentions "packet loss," they're typically talking about the IP layer.

## Detailed Header View

Here's what each header actually contains:

### Ethernet Header (14 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────────────────────────────────────┤
│                    Destination MAC Address                    │
│                         (6 bytes)                             │
├───────────────────────────────────────────────────────────────┤
│                      Source MAC Address                       │
│                         (6 bytes)                             │
├───────────────────────────────────────────────────────────────┤
│         EtherType (2 bytes)       │
│     (0x0800 = IPv4, 0x86DD = IPv6) │
└───────────────────────────────────┘
```

### IPv4 Header (20-60 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────┬───────┬───────────────┬───────────────────────────────┤
│Version│  IHL  │    DSCP/ECN   │         Total Length          │
├───────┴───────┴───────────────┼───────┬───────────────────────┤
│         Identification        │ Flags │    Fragment Offset    │
├───────────────┬───────────────┼───────┴───────────────────────┤
│      TTL      │   Protocol    │        Header Checksum        │
├───────────────┴───────────────┴───────────────────────────────┤
│                       Source IP Address                       │
├───────────────────────────────────────────────────────────────┤
│                    Destination IP Address                     │
├───────────────────────────────────────────────────────────────┤
│                    Options (if IHL > 5)                       │
└───────────────────────────────────────────────────────────────┘
```

### TCP Header (20-60 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────┬───────────────────────────────┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┴───────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────┬───────┬───────────────┬───────────────────────────────┤
│  Data │       │C│E│U│A│P│R│S│F│                               │
│ Offset│ Rsrvd │W│C│R│C│S│S│Y│I│          Window Size          │
│       │       │R│E│G│K│H│T│N│N│                               │
├───────┴───────┴───────────────┼───────────────────────────────┤
│           Checksum            │        Urgent Pointer         │
├───────────────────────────────┴───────────────────────────────┤
│                    Options (if Data Offset > 5)               │
└───────────────────────────────────────────────────────────────┘
```

## Overhead Analysis

Each layer adds overhead. For a small HTTP request:

```
Layer           Header Size    Running Total
─────────────────────────────────────────────
HTTP Data       ~50 bytes      50 bytes
TCP Header      20 bytes       70 bytes
IP Header       20 bytes       90 bytes
Ethernet        18 bytes*      108 bytes
─────────────────────────────────────────────
                              *14 header + 4 FCS

Efficiency: 50/108 = 46% payload
```

For small packets, overhead can be significant. This is why protocols often batch multiple operations or use compression.

## Decapsulation (Receiving)

On the receiving side, each layer strips its header and passes the payload up:

```
   Receiving Host
   ─────────────────────────────────────────────────────────

   Frame arrives → Network Card

   ┌─────────────────────────────────────────────────────┐
   │  Link Layer                                         │
   │                                                     │
   │  1. Verify FCS (checksum)                           │
   │  2. Check destination MAC                           │
   │  3. Read EtherType → 0x0800 (IPv4)                  │
   │  4. Strip Ethernet header, pass up                  │
   └───────────────────────┬─────────────────────────────┘
                           │
                           ▼
   ┌─────────────────────────────────────────────────────┐
   │  Network Layer                                      │
   │                                                     │
   │  1. Verify header checksum                          │
   │  2. Check destination IP                            │
   │  3. Read Protocol field → 6 (TCP)                   │
   │  4. Strip IP header, pass up                        │
   └───────────────────────┬─────────────────────────────┘
                           │
                           ▼
   ┌─────────────────────────────────────────────────────┐
   │  Transport Layer                                    │
   │                                                     │
   │  1. Verify checksum                                 │
   │  2. Read destination port → 80                      │
   │  3. Find socket listening on port 80               │
   │  4. Process TCP state machine                       │
   │  5. Strip TCP header, pass up                       │
   └───────────────────────┬─────────────────────────────┘
                           │
                           ▼
   ┌─────────────────────────────────────────────────────┐
   │  Application Layer                                  │
   │                                                     │
   │  Web server receives: "GET /index.html HTTP/1.1"    │
   └─────────────────────────────────────────────────────┘
```

## How Layers Know What's Inside

Each layer includes a field indicating what's in the payload:

```
Ethernet EtherType:
  0x0800 = IPv4
  0x86DD = IPv6
  0x0806 = ARP

IP Protocol:
  1  = ICMP
  6  = TCP
  17 = UDP
  47 = GRE

TCP/UDP Port:
  80  = HTTP
  443 = HTTPS
  22  = SSH
  53  = DNS
```

This is how a packet finds its way to the right application.

## Encapsulation in Code

Here's a simplified view of building a packet in Python (conceptual):

```python
# Application layer - your data
http_request = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"

# Transport layer - add TCP header
tcp_segment = TCPHeader(
    src_port=52431,
    dst_port=80,
    seq_num=1000,
    ack_num=0,
    flags=SYN
) + http_request

# Network layer - add IP header
ip_packet = IPHeader(
    src_ip="192.168.1.100",
    dst_ip="93.184.216.34",
    protocol=TCP,
    ttl=64
) + tcp_segment

# Link layer - add Ethernet header
ethernet_frame = EthernetHeader(
    src_mac="00:11:22:33:44:55",
    dst_mac="aa:bb:cc:dd:ee:ff",
    ethertype=IPv4
) + ip_packet + calculate_fcs()

# Send it!
network_card.send(ethernet_frame)
```

In practice, the operating system's network stack handles this, but understanding the process helps when debugging.

## Practical Implications

### MTU (Maximum Transmission Unit)

The link layer limits frame size. For Ethernet, the MTU is typically 1500 bytes:

```
Ethernet Frame Limit: 1518 bytes total
  - Ethernet header: 14 bytes
  - Payload: 1500 bytes (MTU)
  - FCS: 4 bytes

Available for IP packet: 1500 bytes
  - IP header: 20 bytes
  - TCP header: 20 bytes
  - Application data: 1460 bytes (typical MSS)
```

If data exceeds this, it must be fragmented—which has performance costs.

### Jumbo Frames

Some networks support larger MTUs (up to 9000 bytes):
- Reduces overhead ratio
- Common in data centers
- Not universal—can cause problems if intermediate networks don't support them

## Summary

Encapsulation is the mechanism that makes layered networking work:

1. **Each layer adds its own header** with information needed for its function
2. **Headers contain "next layer" indicators** so receivers know how to decode
3. **Layers are independent**—changes to one don't affect others
4. **Overhead accumulates**—important for small packet performance

Understanding encapsulation helps you:
- Debug network issues at the right layer
- Understand packet capture output
- Make informed decisions about protocol overhead

Next, we'll explore ports and sockets—how multiple applications share a single network connection.
