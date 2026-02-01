# The IP Layer

The **Internet Protocol (IP)** is the foundation of the internet. It provides logical addressing and routing—the ability to send packets from any device to any other device, regardless of the physical networks in between.

## IP's Simple Contract

IP makes a simple promise: "I'll try to get this packet to its destination."

Notice what IP *doesn't* promise:
- Packets will arrive (they might be dropped)
- Packets will arrive in order (they might take different routes)
- Packets will arrive only once (duplicates can happen)
- Packets will arrive intact (corruption is possible, though detected)

This "best-effort" service might seem inadequate, but it's deliberately minimal. By keeping IP simple, it can be:
- **Fast**: Minimal processing per packet
- **Scalable**: Routers don't maintain connection state
- **Universal**: Works over any link layer

Higher layers (like TCP) can add reliability when needed.

## The Two IP Versions

Today's internet runs on two versions of IP:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   IPv4 (1981)              │   IPv6 (1998)                  │
│   ─────────────────────────│─────────────────────────────   │
│   32-bit addresses         │   128-bit addresses            │
│   ~4.3 billion addresses   │   ~340 undecillion addresses   │
│   Widely deployed          │   Growing adoption             │
│   NAT commonly used        │   NAT generally unnecessary    │
│   Simpler header           │   Fixed header, extensions     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Both are in active use. Your device likely uses both daily.

## What You'll Learn

In this chapter, we'll cover:

1. **IPv4 Addressing**: The original 32-bit addressing scheme
2. **IPv6**: The next generation with its vastly larger address space
3. **Subnetting**: Dividing networks into smaller segments
4. **Routing**: How packets find their way across networks
5. **Fragmentation**: What happens when packets are too big

## Key Concepts Preview

### Addresses Identify Interfaces, Not Hosts

A common misconception is that an IP address identifies a computer. Actually, it identifies a **network interface**. A computer with two network cards has two IP addresses:

```
┌────────────────────────────────────────────────┐
│                    Server                      │
│                                                │
│   ┌────────────┐          ┌────────────┐      │
│   │    eth0    │          │    eth1    │      │
│   │192.168.1.10│          │ 10.0.0.10  │      │
│   └──────┬─────┘          └──────┬─────┘      │
└──────────┼───────────────────────┼────────────┘
           │                       │
      ┌────┴─────┐            ┌────┴─────┐
      │ Network A │            │ Network B │
      └──────────┘            └──────────┘
```

### Routing Is Hop-by-Hop

No device knows the complete path to a destination. Each router makes a local decision about the next hop:

```
Source ──> Router1 ──> Router2 ──> Router3 ──> Destination

Each router:
1. Looks at destination IP
2. Consults routing table
3. Forwards to next hop
4. Forgets about the packet

No router knows the full path. Each just knows "for this
destination, send to that next router."
```

### TTL Prevents Infinite Loops

The **Time to Live (TTL)** field starts at some value (typically 64 or 128) and decrements at each hop. If it reaches 0, the packet is discarded. This prevents packets from circulating forever if there's a routing loop.

```
TTL at source:     64
After router 1:    63
After router 2:    62
...
If routing loop:   Eventually hits 0 → packet dropped
```

Let's dive into the details, starting with IPv4.
