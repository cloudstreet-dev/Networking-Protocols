# When to Use UDP

Choosing UDP over TCP is a significant architectural decision. UDP shines in specific scenarios where its characteristics—low latency, no connection overhead, and application control—outweigh the lack of built-in reliability.

## Primary Use Cases

### Real-Time Communication

**Voice over IP (VoIP)**

```
Why UDP?

TCP behavior on packet loss:
  Packet lost → Retransmit → Arrives 200ms later
  Audio: "Hello, how are--[200ms pause]--you?"

UDP behavior:
  Packet lost → Move on
  Audio: "Hello, how are [brief glitch] you?"

Humans tolerate small audio gaps.
Humans hate delays in conversation.

VoIP typically tolerates 1-5% packet loss gracefully.
Delay > 150ms makes conversation awkward.
```

**Video Streaming (Live)**

```
Live video constraints:
  - Frame every 33ms (30 fps)
  - Old frames are worthless
  - Viewer can't wait for retransmit

UDP approach:
  Lost packet? Skip it, show next frame.
  Minor visual artifact better than frozen video.

Note: Buffered streaming (Netflix) often uses TCP.
      TCP's reliability works when you can buffer ahead.
```

**Online Gaming**

```
Game server sends world state 60 times/second:

Frame 1: Player at (100, 200)
Frame 2: Player at (102, 201)  ← Lost!
Frame 3: Player at (104, 202)
Frame 4: Player at (106, 203)

With TCP: Wait for Frame 2 retransmit
          Game stutters, all updates delayed

With UDP: Skip Frame 2
          Frame 3 has newer position anyway!
          Smooth gameplay

Games implement their own:
  - Sequence numbers (detect loss)
  - Interpolation (smooth missing frames)
  - Prediction (guess missing data)
```

### Simple Request-Response

**DNS (Domain Name System)**

```
DNS query:
  Client: "What's the IP for example.com?"
  Server: "93.184.216.34"

Why UDP (historically)?
  - Single small request (<512 bytes)
  - Single small response
  - No connection state needed
  - Low latency critical (affects every web request)

UDP saves: 1-2 RTT from TCP handshake

Modern note: DNS over TCP exists and is growing
  - Large responses (DNSSEC)
  - DNS over HTTPS/TLS (encrypted, uses TCP)
```

**NTP (Network Time Protocol)**

```
Time sync:
  Client: "What time is it?"
  Server: "2024-01-15 14:23:45.123456"

Latency matters for accuracy!
  Every ms of delay affects time calculation

UDP request-response: ~1 RTT
TCP setup + request: ~3 RTT
```

**DHCP (Dynamic Host Configuration Protocol)**

```
Network bootstrapping:
  Client: "I need an IP address!" (broadcast)
  Server: "You can use 192.168.1.100"

Special challenge: Client has NO IP address yet!
  TCP requires an IP to establish connection
  UDP can broadcast without source IP

Also: DHCP predates TCP optimizations
```

### Broadcast and Multicast

**Service Discovery**

```
Finding services on local network:

Option 1 (TCP): Connect to every device, ask "Are you a printer?"
                Slow, inefficient, doesn't scale

Option 2 (UDP multicast):
  Send to multicast address: "Who's a printer?"
  All printers respond: "Me! I'm at 192.168.1.50"

mDNS/Bonjour uses this (224.0.0.251, port 5353)
```

**IPTV / Live TV Distribution**

```
Sending same video to 10,000 viewers:

TCP: 10,000 separate connections
     10,000 copies of each packet
     Massive server load

UDP Multicast: 1 stream
               Network duplicates as needed
               Scales infinitely

Multicast REQUIRES UDP (TCP is point-to-point).
```

### Custom Protocols

**QUIC (HTTP/3)**

```
QUIC is a custom protocol over UDP:
  - Implements reliability (like TCP)
  - Implements congestion control (like TCP)
  - But with multiplexing, 0-RTT, migration

Why not just improve TCP?
  - TCP is in operating system kernels
  - Kernel changes take years to deploy
  - Middleboxes (firewalls, NAT) expect TCP behavior

UDP is a blank slate:
  - Implement in userspace (fast iteration)
  - Passes through middleboxes (they don't inspect UDP)
  - Customize behavior completely
```

**Custom Game Protocols**

```
Games often need:
  - Reliable delivery for some messages (chat, purchases)
  - Unreliable for others (position updates)
  - Priority levels
  - Custom congestion handling

TCP: One-size-fits-all, no customization
UDP: Build exactly what you need

Many game engines implement hybrid:
  - Reliable ordered channel (mimics TCP)
  - Reliable unordered channel
  - Unreliable channel
  All over single UDP socket.
```

### Lightweight IoT

**Sensor Networks**

```
Thousands of sensors reporting temperature:
  - Small messages (few bytes)
  - Frequent updates
  - Individual readings not critical
  - Network/power constrained

TCP overhead per reading:
  20-byte header (often > payload!)
  Connection state on server

UDP overhead:
  8-byte header
  No state
  Fire and forget

CoAP (Constrained Application Protocol) uses UDP.
```

## When NOT to Use UDP

### File Transfer

```
File transfer requirements:
  ✓ Complete delivery (every byte matters)
  ✓ Correct order
  ✓ Error detection

UDP would require implementing:
  - Sequence numbers
  - Acknowledgments
  - Retransmission
  - Congestion control

...basically reimplementing TCP poorly.

Use TCP for file transfer. Or QUIC.
```

### Web APIs / HTTP

```
HTTP requires:
  ✓ Reliable delivery (incomplete JSON is useless)
  ✓ Request-response matching
  ✓ Large responses

TCP is the right choice.
(HTTP/3 uses QUIC over UDP, but QUIC handles reliability)
```

### Anything Through Firewalls

```
Many corporate firewalls:
  - Allow TCP 80, 443
  - Block most UDP
  - May even block all UDP

If targeting corporate networks:
  Consider TCP for better connectivity.

WebSocket (TCP) often works where custom UDP doesn't.
```

## Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│               Should I Use UDP?                             │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
           ┌──────────────────────────────┐
           │ Is low latency critical?     │
           │ (Real-time, interactive)     │
           └──────────────┬───────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
             Yes                      No
              │                       │
              ▼                       ▼
   ┌────────────────────┐   ┌────────────────────┐
   │ Can you tolerate   │   │ Do you need        │
   │ some data loss?    │   │ broadcast/multicast?│
   └─────────┬──────────┘   └─────────┬──────────┘
             │                        │
     ┌───────┴───────┐        ┌───────┴───────┐
     │               │        │               │
    Yes              No      Yes              No
     │               │        │               │
     ▼               ▼        ▼               ▼
  ┌──────┐     ┌───────┐   ┌──────┐     ┌───────┐
  │ UDP! │     │QUIC or│   │ UDP! │     │  TCP  │
  │      │     │  TCP  │   │      │     │       │
  └──────┘     └───────┘   └──────┘     └───────┘
```

## Real-World Examples

### Discord Voice

```
Text chat: TCP (reliable)
Voice chat: UDP (low latency)

Voice handling:
  - Opus codec (tolerates loss)
  - Packet loss concealment
  - Jitter buffer
  - Falls back to TCP if UDP blocked
```

### Zoom

```
Video: UDP preferred, TCP fallback
Audio: UDP preferred, TCP fallback
Screen share: UDP preferred

Quality adapts to conditions:
  - High loss? Reduce quality
  - UDP blocked? Switch to TCP
  - Still works, but with higher latency
```

### DNS

```
Traditional: UDP port 53
  - Fast, simple
  - Limited to 512 bytes (without EDNS)

DNS over TCP: Port 53
  - Large responses (DNSSEC)
  - Zone transfers

DNS over HTTPS: TCP port 443
  - Encrypted
  - Privacy focused
  - More overhead
```

### Online Games

```
Fortnite, Valorant, etc.:
  Position updates: UDP (unreliable, frequent)
  Game events: UDP (reliable channel)
  Chat: UDP or TCP
  Downloads/patches: TCP

Hybrid approach is common.
```

## Summary

Use UDP when:
- **Latency matters more than reliability**
- **Data has short lifespan** (real-time)
- **Some loss is acceptable**
- **Broadcast/multicast needed**
- **Building custom protocol** (like QUIC)
- **Extreme resource constraints** (IoT)

Use TCP when:
- **Every byte must arrive**
- **Order matters**
- **Simplicity preferred** (let TCP handle complexity)
- **Firewall traversal important**
- **Building on existing TCP-based protocols**

The key question: **Is old data still valuable?**
- Yes → TCP (file, web page, API)
- No → Consider UDP (voice, video, game state)

Next, we'll do a detailed comparison of UDP vs TCP trade-offs.
