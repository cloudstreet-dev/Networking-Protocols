# Routing Fundamentals

**Routing** is the process of selecting paths for network traffic. When you send a packet to a destination across the internet, it passes through many intermediate devices (routers) that make forwarding decisions. Understanding routing helps you debug connectivity issues and design better network architectures.

## The Core Concept

Routing works through a simple, repeated process:

```
At each router:
┌─────────────────────────────────────────────────────────────┐
│  1. Receive packet                                          │
│  2. Examine destination IP address                          │
│  3. Consult routing table                                   │
│  4. Forward packet to next hop (or deliver locally)         │
│  5. Decrement TTL                                           │
│  6. Forget about the packet                                 │
└─────────────────────────────────────────────────────────────┘

No router knows the complete path. Each makes a local decision.
```

This **hop-by-hop** routing is fundamental to the internet's scalability and resilience.

## Direct vs. Indirect Delivery

When a host wants to send a packet, it first determines if the destination is **local** (same network) or **remote** (different network):

```
Source: 192.168.1.100/24

Case 1: Destination 192.168.1.200 (same network)
┌───────────────────────────────────────────────────────────────┐
│  Apply subnet mask:                                           │
│    192.168.1.100 AND 255.255.255.0 = 192.168.1.0              │
│    192.168.1.200 AND 255.255.255.0 = 192.168.1.0              │
│  Same network! → Direct delivery via ARP                      │
└───────────────────────────────────────────────────────────────┘

Case 2: Destination 10.0.0.50 (different network)
┌───────────────────────────────────────────────────────────────┐
│  Apply subnet mask:                                           │
│    192.168.1.100 AND 255.255.255.0 = 192.168.1.0              │
│    10.0.0.50 AND 255.255.255.0 = 10.0.0.0                     │
│  Different networks! → Send to default gateway                │
└───────────────────────────────────────────────────────────────┘
```

## The Routing Table

A **routing table** maps destination networks to next hops. Every device with IP networking has one:

```bash
$ ip route show  # Linux
$ netstat -rn    # Linux/Mac
$ route print    # Windows

Example output:
┌──────────────────┬──────────────────┬─────────────┬───────────┐
│   Destination    │     Gateway      │   Iface     │  Metric   │
├──────────────────┼──────────────────┼─────────────┼───────────┤
│   0.0.0.0/0      │   192.168.1.1    │    eth0     │    100    │
│   192.168.1.0/24 │   0.0.0.0        │    eth0     │    0      │
│   10.10.0.0/16   │   192.168.1.254  │    eth0     │    100    │
│   127.0.0.0/8    │   0.0.0.0        │    lo       │    0      │
└──────────────────┴──────────────────┴─────────────┴───────────┘

Entry meanings:
- 0.0.0.0/0: Default route ("everything else") → send to 192.168.1.1
- 192.168.1.0/24: Local network → deliver directly (0.0.0.0 gateway)
- 10.10.0.0/16: Route to remote network → via 192.168.1.254
- 127.0.0.0/8: Loopback → handled locally
```

### Routing Table Lookup

When forwarding a packet, the router finds the **most specific matching route** (longest prefix match):

```
Destination: 10.10.5.100

Routing table entries:
  0.0.0.0/0      → Gateway A (default)
  10.0.0.0/8     → Gateway B
  10.10.0.0/16   → Gateway C
  10.10.5.0/24   → Gateway D

Matching process:
  0.0.0.0/0    - Matches (but only 0 bits specific)
  10.0.0.0/8   - Matches (8 bits specific)
  10.10.0.0/16 - Matches (16 bits specific)
  10.10.5.0/24 - Matches (24 bits specific) ← WINNER

Result: Forward to Gateway D (most specific match)
```

## Static vs. Dynamic Routing

### Static Routing

Routes manually configured by an administrator:

```bash
# Add a static route (Linux)
$ sudo ip route add 10.20.0.0/16 via 192.168.1.254

# Persistent (varies by distro, often in /etc/network/interfaces or netplan)
```

**Pros:**
- Simple, predictable
- No protocol overhead
- Good for small, stable networks

**Cons:**
- Doesn't adapt to failures
- Tedious for large networks
- Error-prone at scale

### Dynamic Routing

Routes learned automatically via routing protocols:

```
┌─────────────────────────────────────────────────────────────┐
│                    Routing Protocols                        │
├─────────────────────────────────────────────────────────────┤
│  Interior Gateway Protocols (within organization):          │
│    RIP   - Simple, distance-vector, limited scale          │
│    OSPF  - Link-state, widely used, complex                │
│    EIGRP - Cisco proprietary, efficient                    │
│    IS-IS - Link-state, used by large ISPs                  │
│                                                             │
│  Exterior Gateway Protocol (between organizations):         │
│    BGP   - Border Gateway Protocol, runs the internet      │
└─────────────────────────────────────────────────────────────┘
```

**Pros:**
- Automatically adapts to failures
- Scales to large networks
- Finds optimal paths

**Cons:**
- Protocol overhead
- More complex to configure
- Convergence time during changes

## How Routing Protocols Work

### Distance-Vector (RIP)

Routers share their entire routing table with neighbors periodically:

```
Initial state:
                  ┌───────────────┐
  Router A ─────── Router B ─────── Router C
  Knows:          Knows:          Knows:
  Net 1           Net 2           Net 3

After exchange:
  Router A        Router B        Router C
  Knows:          Knows:          Knows:
  Net 1 (direct)  Net 1 (via A)   Net 1 (via B)
  Net 2 (via B)   Net 2 (direct)  Net 2 (via B)
  Net 3 (via B)   Net 3 (via C)   Net 3 (direct)
```

### Link-State (OSPF)

Each router learns the complete network topology and calculates best paths:

```
1. Each router discovers neighbors
2. Each router floods link-state info to all routers
3. Every router has identical network map
4. Each router independently calculates best paths (Dijkstra's algorithm)

Advantage: Faster convergence, no routing loops during transition
Disadvantage: More memory and CPU intensive
```

## BGP: The Internet's Routing Protocol

**BGP (Border Gateway Protocol)** is how autonomous systems (AS) exchange routing information:

```
┌──────────────────┐          ┌──────────────────┐
│     AS 65001     │          │     AS 65002     │
│   (Your ISP)     │──BGP────│   (Another ISP)  │
│                  │          │                  │
│  Announces:      │          │  Announces:      │
│  203.0.113.0/24  │          │  198.51.100.0/24 │
└──────────────────┘          └──────────────────┘

BGP characteristics:
- Path-vector protocol (tracks AS path)
- Policy-based routing (not just shortest path)
- Slow convergence (stability over speed)
- ~900,000+ routes in global table
```

### BGP Path Selection

BGP chooses routes based on multiple criteria (simplified):

1. Highest local preference
2. Shortest AS path
3. Lowest origin type
4. Lowest MED (Multi-Exit Discriminator)
5. Prefer eBGP over iBGP
6. Lowest IGP metric to next hop
7. ... (many more tie-breakers)

## Routing in Action

Let's trace a packet from your laptop to a web server:

```
Your Laptop (192.168.1.100)
    │
    │ Destination: 93.184.216.34 (example.com)
    │ Different network → send to default gateway
    ▼
Home Router (192.168.1.1)
    │
    │ Routing table: default route → ISP
    ▼
ISP Router #1
    │
    │ BGP table: 93.184.216.0/24 → via AS 15133
    │ (Multiple paths available, chooses best)
    ▼
ISP Router #2
    │
    │ BGP: Next hop toward destination AS
    ▼
    ... (several more hops) ...
    │
    ▼
Destination Router
    │
    │ 93.184.216.0/24 is directly connected
    │ ARP for 93.184.216.34, deliver to server
    ▼
Web Server (93.184.216.34)
```

## Traceroute: Seeing the Path

**Traceroute** reveals the path packets take by exploiting TTL:

```bash
$ traceroute example.com
 1  192.168.1.1 (192.168.1.1)     1.234 ms
 2  96.120.92.1 (96.120.92.1)     12.456 ms
 3  68.86.90.137 (68.86.90.137)   15.789 ms
 4  * * *                         (no response)
 5  be-33651-cr02.nyc (66.109.6.81) 25.123 ms
 6  93.184.216.34 (93.184.216.34) 28.456 ms

How it works:
  Send packet with TTL=1 → First router replies "TTL exceeded"
  Send packet with TTL=2 → Second router replies
  Send packet with TTL=3 → Third router replies
  ... continue until destination reached
```

### Reading Traceroute Output

```
Hop 4: * * *

This means:
- Router didn't respond to traceroute probes
- Could be: firewall blocking, ICMP rate limiting, high latency
- Packets might still pass through this router fine
- Not necessarily a problem

Multiple times per hop:
 3  68.86.90.137  15.789 ms  16.123 ms  14.567 ms
    └─────────────┴─ Three separate probes, showing latency variation
```

## Common Routing Issues

### Routing Loops

Misconfiguration can cause packets to circle:

```
Router A: "To reach 10.0.0.0/8, send to B"
Router B: "To reach 10.0.0.0/8, send to C"
Router C: "To reach 10.0.0.0/8, send to A"

Packet bounces: A → B → C → A → B → C → ...
Until TTL reaches 0!

Solutions:
- TTL prevents infinite loops
- Routing protocols have loop prevention (split horizon, etc.)
- BGP uses AS path to detect loops
```

### Asymmetric Routing

Outbound and inbound paths can differ:

```
Request:  A → B → C → D → Server
Response: Server → E → F → A

This is normal and common!
Can complicate:
- Troubleshooting
- Stateful firewalls
- Performance analysis
```

### Black Holes

Traffic enters but doesn't come out:

```
Causes:
- Null route (route to nowhere)
- Firewall silently drops
- Network failure with no alternative path
- MTU issues (packets too large)

Debugging:
- Traceroute to find where packets stop
- Check routing tables
- Verify firewall rules
```

## Routing Table Management

### Viewing Routes

```bash
# Linux
$ ip route show
$ ip -6 route show  # IPv6

# macOS
$ netstat -rn

# Windows
> route print
```

### Adding/Removing Routes

```bash
# Linux - add route
$ sudo ip route add 10.20.0.0/16 via 192.168.1.254

# Linux - remove route
$ sudo ip route del 10.20.0.0/16

# Linux - change default gateway
$ sudo ip route replace default via 192.168.1.1

# Make persistent (varies by distro)
# Ubuntu/Debian: /etc/netplan/*.yaml
# RHEL/CentOS: /etc/sysconfig/network-scripts/route-eth0
```

## Summary

Routing is the backbone of internet connectivity:

- **Hop-by-hop forwarding**: Each router makes local decisions
- **Routing tables**: Map destinations to next hops
- **Longest prefix match**: Most specific route wins
- **Static routing**: Manual configuration for simple networks
- **Dynamic routing**: Protocols (OSPF, BGP) for automatic adaptation
- **BGP**: The protocol that makes the internet work

Key debugging tools:
- `ip route` / `netstat -rn`: View routing tables
- `traceroute` / `tracert`: See packet paths
- `ping`: Test basic connectivity

Understanding routing helps you diagnose why packets aren't reaching their destination and design networks that are resilient to failures.

Next, we'll cover IP fragmentation—what happens when packets are too large for a network link.
