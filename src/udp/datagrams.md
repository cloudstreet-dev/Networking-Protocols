# UDP Header and Datagrams

The UDP header is remarkably simple—just 8 bytes compared to TCP's minimum of 20. This minimalism is by design, providing just enough functionality to multiplex applications and detect corruption.

## UDP Datagram Structure

```
┌─────────────────────────────────────────────────────────────┐
│                      UDP Datagram                           │
├─────────────────────────────────────────────────────────────┤
│     UDP Header      │           UDP Payload                 │
│      (8 bytes)      │      (0 to 65,507 bytes)              │
└─────────────────────────────────────────────────────────────┘
```

Maximum payload: 65,535 (IP max) - 20 (IP header) - 8 (UDP header) = 65,507 bytes

But practical limit is usually much smaller due to MTU.

## The UDP Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┼───────────────────────────────┤
│            Length             │           Checksum            │
├───────────────────────────────┴───────────────────────────────┤
│                                                               │
│                           Payload                             │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

That's it. Four 16-bit fields. Compare to TCP's 10+ fields!

## Header Fields

### Source Port (16 bits)

```
The sender's port number.

Optional: Can be 0 if no reply is expected
  (Though this is rarely done in practice)

Used by receiver to send responses back.
```

### Destination Port (16 bits)

```
The receiver's port number.

Identifies the application/service.
Well-known ports same as TCP:
  53  = DNS
  67  = DHCP Server
  68  = DHCP Client
  69  = TFTP
  123 = NTP
  161 = SNMP
  500 = IKE (IPsec)
```

### Length (16 bits)

```
Total length of UDP datagram (header + payload).

Minimum: 8 (header only, no payload)
Maximum: 65535 (theoretical, rarely practical)

Length = 8 + payload_size
```

### Checksum (16 bits)

```
Error detection for header and data.

IPv4: Optional (0 = disabled)
IPv6: Mandatory

Calculated over:
  - UDP pseudo-header (from IP)
  - UDP header
  - UDP payload (padded if odd length)
```

#### Pseudo-Header

```
Like TCP, UDP checksum covers IP addresses:

IPv4 Pseudo-Header:
┌───────────────────────────────────────────────────────────────┐
│                       Source IP Address                       │
├───────────────────────────────────────────────────────────────┤
│                    Destination IP Address                     │
├───────────────┬───────────────┬───────────────────────────────┤
│    Zero (8)   │ Protocol (17) │         UDP Length            │
└───────────────┴───────────────┴───────────────────────────────┘

This ensures the datagram reaches the intended destination.
If IP addresses were modified, checksum fails.
```

## Comparing Headers

```
TCP Header (minimum):               UDP Header:
┌───────────────────────────┐      ┌───────────────────────────┐
│ Source Port (16)          │      │ Source Port (16)          │
│ Destination Port (16)     │      │ Destination Port (16)     │
│ Sequence Number (32)      │      │ Length (16)               │
│ Acknowledgment (32)       │      │ Checksum (16)             │
│ Data Offset/Flags (16)    │      └───────────────────────────┘
│ Window (16)               │           8 bytes total
│ Checksum (16)             │
│ Urgent Pointer (16)       │
│ [Options...]              │
└───────────────────────────┘
     20+ bytes

TCP overhead: 20+ bytes
UDP overhead: 8 bytes

For small messages, the difference matters!
```

## UDP Socket Programming

### Sending a Datagram

```python
import socket

# Create UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# No connect() needed - just send!
message = b"Hello, UDP!"
sock.sendto(message, ("192.168.1.100", 12345))

# Can send to different destinations with same socket
sock.sendto(b"Hello A", ("192.168.1.101", 12345))
sock.sendto(b"Hello B", ("192.168.1.102", 12345))

sock.close()
```

### Receiving Datagrams

```python
import socket

# Create and bind UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", 12345))

print("Listening on port 12345...")

while True:
    # recvfrom returns data AND sender address
    data, addr = sock.recvfrom(65535)  # Buffer size
    print(f"Received from {addr}: {data.decode()}")

    # Can reply directly
    sock.sendto(b"Got it!", addr)
```

### Connected UDP (Optional)

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Can "connect" UDP socket - sets default destination
sock.connect(("192.168.1.100", 12345))

# Now can use send() instead of sendto()
sock.send(b"Hello!")

# recv() instead of recvfrom()
response = sock.recv(1024)

# Also enables receiving ICMP errors
# (Unconnected UDP sockets don't see them)
```

## Message Boundaries

Unlike TCP's byte stream, UDP preserves message boundaries:

```
Sender:
  sendto(b"Message 1")
  sendto(b"Message 2")
  sendto(b"Message 3")

Receiver:
  recvfrom() → b"Message 1"
  recvfrom() → b"Message 2"
  recvfrom() → b"Message 3"

Each datagram is delivered as a complete unit (or not at all).
Never get partial messages or merged messages.
```

## Datagram Size Considerations

### Practical Limits

```
Maximum theoretical: 65,507 bytes
Maximum without fragmentation: MTU - IP header - UDP header
  Ethernet: 1500 - 20 - 8 = 1472 bytes
  Jumbo:    9000 - 20 - 8 = 8972 bytes

Recommended maximum for reliability:
  512-1400 bytes (avoids fragmentation)

DNS uses 512 bytes historically (EDNS allows larger)
```

### Fragmentation Problem

```
UDP datagram > MTU gets fragmented at IP layer:

Original: 3000-byte UDP datagram
          │
          ▼
┌─────────────────┐  ┌─────────────────┐  ┌────────────┐
│ IP Frag 1       │  │ IP Frag 2       │  │ IP Frag 3  │
│ UDP hdr + 1472B │  │ Data (1480B)    │  │ Data (48B) │
└─────────────────┘  └─────────────────┘  └────────────┘

Problems:
  - Any fragment lost = entire datagram lost
  - No automatic retransmission
  - Higher effective loss rate

Best practice: Keep datagrams under MTU
```

## No Connection State

UDP sockets don't track connections:

```
TCP Server:
  listen()
  while True:
      client = accept()  ← New socket per connection
      handle(client)
      client.close()

UDP Server:
  bind()
  while True:
      data, addr = recvfrom()  ← All messages on same socket
      # addr tells you who sent it
      handle(data, addr)
      sendto(response, addr)

UDP has no notion of "accepted connections"
Just receives datagrams with source addresses.
```

## Common UDP Patterns

### Request-Response

```
Client                              Server
   │                                   │
   │─── Request (with client port) ───>│
   │                                   │
   │<── Response (to client port) ─────│
   │                                   │

Simple: One datagram each direction.
DNS works this way.
```

### Streaming

```
Source                              Destination
   │                                     │
   │─── Packet 1 (seq=1) ───────────────>│
   │─── Packet 2 (seq=2) ───────────────>│
   │─── Packet 3 (seq=3) ──X             │  Lost!
   │─── Packet 4 (seq=4) ───────────────>│
   │─── Packet 5 (seq=5) ───────────────>│
   │                                     │
   │   Receiver notices gap, may request │
   │   retransmit or skip packet 3       │

Application implements sequencing/recovery as needed.
Video streaming, gaming use this pattern.
```

### Multicast

```
One sender, multiple receivers:

Source ─────────────────┬───────────────> Receiver A
    │                   │
    │    Multicast      ├───────────────> Receiver B
    │    Group          │
    │                   └───────────────> Receiver C

UDP is required for multicast (TCP is point-to-point only).
Used for IPTV, service discovery, LAN gaming.
```

## Viewing UDP Traffic

```bash
# Linux: Show UDP sockets
$ ss -u -a
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
UNCONN  0       0       0.0.0.0:68           0.0.0.0:*
UNCONN  0       0       127.0.0.1:323        0.0.0.0:*

# Capture UDP packets
$ tcpdump -i eth0 udp port 53
14:23:15.123 IP 192.168.1.100.52431 > 8.8.8.8.53: UDP, length 32

# Show UDP statistics
$ netstat -su
Udp:
    1234567 packets received
    12 packets to unknown port received
    0 packet receive errors
    1234560 packets sent
```

## Summary

The UDP header is minimal by design:

| Field | Size | Purpose |
|-------|------|---------|
| Source Port | 16 bits | Reply address |
| Destination Port | 16 bits | Target application |
| Length | 16 bits | Datagram size |
| Checksum | 16 bits | Error detection |

Key characteristics:
- **8-byte header** (vs TCP's 20+)
- **Message-oriented** (boundaries preserved)
- **Connectionless** (no state to manage)
- **No fragmentation at UDP level** (handled by IP)

UDP provides just enough to identify applications and detect corruption. Everything else—reliability, ordering, flow control—is the application's responsibility (if needed at all).

Next, we'll explore when UDP is the right choice and common use cases.
